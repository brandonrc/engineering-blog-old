---
title: "Bare-Metal Rocky Linux Image Mode with Artifact Keeper"
slug: rocky-linux-image-mode-artifact-keeper
authors:
- brandon-geraci
categories:
- Engineering
meta_description: "Deploy Rocky Linux image mode to bare metal with Artifact Keeper as the single source of truth for RPMs, signed bootc images, keys and Kubernetes workloads."
focus_keyword: Rocky Linux image mode
---

OpenTeams spends most of its days on AI and ML and the infrastructure that runs it. This post is about something less glamorous: the hardware in a rack in a closet, the box somebody has to drive to when it misbehaves. If that is your world, keep reading. If you know someone whose world it is, send this their way.

Two invitations before we start. If you run deployments that have grown complicated and you suspect a system like this could simplify them, I would like to hear about them; my contact is at the bottom, and we can work out where Artifact Keeper fits. And if you are a nerd like me who wants to try this in a home lab, I want to hear how that goes just as much.

Last, a thank you to Edward Mora for putting Artifact Keeper to work at his company. This post is my open version of what I believe he built internally, and it would not exist without that example.

## The problem: four sources of truth

Every box in an edge fleet should be running the same operating system, the same Kubernetes, and the same site configuration, and you should be able to prove it. That is harder than it sounds. With a package-based install, two machines start to drift the first time someone runs `dnf update` on one of them and not the other. Image mode fixes this at the operating system layer. With [bootc](https://bootc-dev.github.io/bootc/) (a tool that boots and updates a Linux system from a container image), the whole operating system is built with a Containerfile, nodes boot it, and upgrading means pointing them at a new image. Rocky Linux image mode is this idea on Rocky Linux.

An image is only as trustworthy as where it came from, though. Think about a typical setup: the base RPMs come from a public mirror, the Kubernetes packages come from a vendor repository, your own configuration comes from a tarball on someone's laptop, and the final image sits in yet another registry. That is four sources of truth. When something goes wrong on node 37, nobody can say for sure what is running on it.

This post walks through a proof of concept that puts all of that in one place. [Artifact Keeper](https://github.com/artifact-keeper/artifact-keeper) is an open-source universal artifact registry, and in this setup it holds every byte a node boots from:

- Rocky Linux 10 BaseOS, AppStream, extras and EPEL, as **RPM proxy repositories**
- RKE2 (Rancher's Kubernetes distribution) packages, as another **RPM proxy**
- our own `edge-site-config` package, in a **hosted RPM repository**
- the Rocky bootc base image we build ourselves, and the edge image built on top of it, in a **hosted OCI registry**
- the container images the cluster runs, through **OCI proxies** for Docker Hub and quay.io
- the public keys everything is verified against, in a **hosted generic repository**

A node boots the stock Rocky installer with a kickstart file (the answer file the Enterprise Linux installer reads so nobody has to type anything), pulls the edge image from Artifact Keeper, and comes up running RKE2. Upgrades are a `bootc upgrade` against the same registry, and every RPM and image is signed and checked on the way in. The code is in [github.com/brandonrc/rocky-custom-baremetal-deployment](https://github.com/brandonrc/rocky-custom-baremetal-deployment). The [documentation site](https://brandonrc.github.io/rocky-custom-baremetal-deployment/) for that repo has the full reference, from [setting up the environment](https://brandonrc.github.io/rocky-custom-baremetal-deployment/environment/) to [every make target](https://brandonrc.github.io/rocky-custom-baremetal-deployment/walkthrough/). We wanted anyone to be able to run this, so our "bare metal" is a virtual machine, and nothing in the pipeline knows the difference.

```mermaid
graph LR
    subgraph AK[Artifact Keeper]
        R1[rpm-rocky10-* proxy]
        R2[rpm-rke2-* proxy]
        R3[rpm-edge-site hosted]
        O1[oci-bootc hosted]
        O2[oci-dockerhub-proxy]
        O3[oci-quay-proxy]
        KEYS[raw-edge-keys hosted]
    end
    U1[dl.rockylinux.org] --> R1
    U2[rpm.rancher.io] --> R2
    U3[registry-1.docker.io] --> O2
    U4[quay.io] --> O3
    B[podman build + cosign] --> R1 & R2 & R3 & O3
    B --> O1
    K[Rocky installer + kickstart] --> O1 & KEYS
    K --> N[Edge node: bootc + RKE2]
    N -->|bootc upgrade| O1
    N -->|workload images| O2
```

The [architecture page](https://brandonrc.github.io/rocky-custom-baremetal-deployment/architecture/) has the full version of this diagram and the complete repository list.

## Step 1: Stand up Artifact Keeper

Artifact Keeper ships a `docker-compose.yml` that brings up Postgres, OpenSearch, the Rust backend, the Next.js web UI, and a Caddy front door on port 30080. We ran it with rootless podman, and it was serving requests about a minute after we started it. There are three settings worth knowing about up front, because the rest of the walkthrough depends on them:

1. **Generate the secrets before the first start.** `JWT_SECRET` and `AK_WEBHOOK_SECRET_KEY` need real values. Our `up.sh` creates them with `openssl rand` and also sets `ADMIN_PASSWORD`, so the instance comes up ready to use with no interactive setup. We did this because we did not want anyone typing passwords into a console as part of an automated deployment.
2. **Make the edge repositories public.** Repositories are private by default, which is the right default for a registry. Edge nodes need to read anonymously, so every repository in this walkthrough is created with `is_public: true`.
3. **The OCI registry is path-based.** Images live at `host:30080/<repo-id>/<image>:<tag>`, and the first path segment picks the repository. This is what lets one registry hold separate image repositories, each with its own policy.

Bootstrapping is a handful of `POST /api/v1/repositories` calls. Remote RPM repositories need a concrete `baseurl` (mirrorlists are not supported), and Artifact Keeper generates the `repodata` for hosted repositories itself, so there is no `createrepo` step:

```bash
curl -X POST http://localhost:30080/api/v1/repositories \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"key":"rpm-rocky10-baseos","name":"Rocky 10 BaseOS","format":"rpm",
       "repo_type":"remote","is_public":true,
       "upstream_url":"https://dl.rockylinux.org/pub/rocky/10/BaseOS/x86_64/os/"}'

curl -X POST http://localhost:30080/api/v1/repositories \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"key":"oci-bootc","name":"bootc images","format":"docker",
       "repo_type":"local","is_public":true}'
```

This is the dnf configuration the image build sees, and the only one:

```ini
[rpm-rocky10-baseos]
name=Rocky Linux 10 BaseOS (proxy)
baseurl=http://host.containers.internal:30080/rpm/rpm-rocky10-baseos
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-Rocky-10
```

Uploading our own RPM is one `PUT`, and `dnf` sees it in the generated metadata right away:

```bash
curl -u "admin:$TOKEN" --upload-file edge-site-config-1.0-3.el10.noarch.rpm \
  http://localhost:30080/rpm/rpm-edge-site/packages/edge-site-config-1.0-3.el10.noarch.rpm
```

Here is the full set after bootstrapping. All of them are public so that edge nodes do not need credentials to read:

| Repository | Format | Type | Upstream or contents |
|---|---|---|---|
| `rpm-rocky10-baseos`, `-appstream`, `-extras` | RPM | proxy | dl.rockylinux.org, Rocky 10 |
| `rpm-epel10` | RPM | proxy | dl.fedoraproject.org, EPEL 10 |
| `rpm-rke2-common`, `rpm-rke2-1.36` | RPM | proxy | rpm.rancher.io, RKE2 EL10 packages |
| `rpm-k3s` | RPM | proxy | rpm.rancher.io, k3s SELinux policy only; kept for a k3s fallback |
| `rpm-edge-site` | RPM | hosted | our `edge-site-config` package |
| `oci-bootc` | OCI | hosted | base and edge bootc images, and their signatures |
| `oci-quay-proxy`, `oci-dockerhub-proxy` | OCI | proxy | quay.io, registry-1.docker.io |
| `raw-edge-keys` | generic | hosted | public keys (added in Step 6) |

The whole bootstrap is one shell script in the repo, and you can run it as many times as you want. It also creates a scoped API token so that routine pushes and uploads never use the admin password. The API calls and settings we used are collected in the [Artifact Keeper notes](https://brandonrc.github.io/rocky-custom-baremetal-deployment/artifact-keeper-notes/).

![Artifact Keeper repositories view listing the RPM proxy and hosted repositories and the OCI hosted and proxy repositories](images/rocky-linux-image-mode-artifact-keeper/01-repositories.png)

## Step 2: Build a Rocky Linux image mode base

We chose Rocky Linux because it is a community-run, open-source Enterprise Linux, and that is exactly what we want underneath hardware we are going to own for a decade. For image mode, Rocky gives you the recipe instead of a finished image. The Rocky Enterprise Software Foundation's [`rocky-bootc`](https://git.resf.org/sig_containers/rocky-bootc) repository (branch `r10`) wraps the Fedora `bootc-base-images` tooling so you can compose your own bootc base. There are prebuilt bootc images from the commercial Rocky vendor and from the community, but for a fleet whose packages all have to come from one place, building from the recipe is the better fit.

So we build the base ourselves, and that is what makes the rest of this work: the base image's RPMs come through Artifact Keeper too.

The recipe is a two-stage Containerfile. A `rockylinux:10` builder installs `rpm-ostree` (ostree is the content store bootc uses to keep the operating system read-only and versioned), composes a root filesystem from the recipe's manifests, and exports it as a chunked OCI archive into a bind-mounted directory. The second stage is `FROM oci-archive:./out.ociarchive`. The build needs `--cap-add=all --device /dev/fuse --security-opt=label=disable -v $(pwd):/buildcontext`. We expected that to be a problem for rootless podman. It was not; the recipe built unmodified in under five minutes, which matters if you want this running in a CI runner.

We changed three things, all in a wrapper Containerfile of our own. The vendored upstream recipe is untouched:

```dockerfile
#| highlight: 1, 4-6, 10
ARG BUILDER_IMAGE=localhost:30080/oci-quay-proxy/rockylinux/rockylinux:10
FROM ${BUILDER_IMAGE} as builder
COPY ak-rocky.repo /tmp/ak-rocky.repo
RUN rm -f /etc/yum.repos.d/*.repo \
 && install -m 0644 /tmp/ak-rocky.repo /etc/yum.repos.d/ak-rocky.repo \
 && dnf -y install rpm-ostree selinux-policy-targeted \
 && dnf clean all
COPY . /src
WORKDIR /src
ARG MANIFEST=minimal
RUN --mount=type=cache,target=/workdir /src/build.sh

FROM oci-archive:./out.ociarchive
# ... upstream's post-processing and labels, unchanged
```

1. **The builder image comes from the Artifact Keeper quay.io proxy** instead of Docker Hub.
2. **The builder's dnf repositories are replaced with the Artifact Keeper Rocky proxies** before anything is installed. There is no repository setting in the recipe's manifests; `rpm-ostree compose rootfs --source-root=/` just reuses the builder's `/etc/yum.repos.d`. Replace those files and every one of the 242 RPMs in the base comes through the registry. We checked the build log to be sure: every package resolved from `rpm-rocky10-baseos` or `rpm-rocky10-appstream`.
3. **`MANIFEST=minimal` instead of the upstream default `standard`.** The standard manifest is 450 packages, including sssd, nfs-utils and cloud agents, and its `autoupdates.yaml` links `bootc-fetch-apply-updates.timer` into `default.target.wants` under `/usr`. That means the node would fetch updates, apply them, and reboot on its own. `systemctl is-enabled` reports it as `disabled` because it is a static link rather than an enablement, so this is easy to miss. We wanted upgrades to be driven by retagging in the registry, not by a timer on the node, so minimal was the right choice for us.

The result is Rocky Linux 10.2 with kernel 6.12 and bootc 1.16.4: 68 chunked layers, 383 MB compressed, and a clean `bootc container lint`. It is essentially the same shape as the community image (242 RPMs and 68 layers against 238 and 65), which gave us some confidence we had done it right. We push it as `oci-bootc/rocky-bootc-base:10` plus an immutable date tag. There were two build-system details that tripped us up, about the working directory and `--no-cache`; those are in the [image build notes](https://brandonrc.github.io/rocky-custom-baremetal-deployment/findings-image/).

## Step 3: Add Kubernetes and site configuration from the same registry

The edge image is a normal Containerfile that starts `FROM` our base in Artifact Keeper. For Kubernetes we chose RKE2 over k3s for one practical reason: **RKE2 ships real EL10 RPMs** on `rpm.rancher.io`, both `rke2-server` and `rke2-selinux`, and k3s has no RPM at all. With RPMs, the Kubernetes distribution flows through the same RPM proxy type as the operating system, which is the whole point of this exercise. We proxy the `stable/1.36` tree, which is the version the RKE2 stable channel pointed at when we did this.

Tags follow one rule: `rocky-edge:<Rocky minor>-<site-config release>`. So `10.2-4` means Rocky 10.2 plus `edge-site-config` release 4. We build two releases of the site configuration so that there is something to ship on day 2. The ones you will see in this post are 3 and 4. Releases 1 and 2 were our unsigned dry run, and Step 6 explains why they stay in the registry.

```dockerfile
#| highlight: 12-13, 16, 20-22, 30-32, 34
ARG BASE_IMAGE=localhost:30080/oci-bootc/rocky-bootc-base:10
FROM ${BASE_IMAGE}

COPY build.repo /tmp/build-repos/build.repo
COPY edge.repo /tmp/edge.repo

# Layer 1: OS additions + RKE2. Unchanged across site-config releases.
RUN set -eux; \
    rm -f /etc/yum.repos.d/*.repo; \
    install -m 0644 /tmp/edge.repo /etc/yum.repos.d/edge.repo; \
    kver="$(rpm -q --qf '%{VERSION}-%{RELEASE}' kernel)"; \
    dnf -y --setopt=reposdir=/tmp/build-repos --setopt=install_weak_deps=False install \
        NetworkManager openssh-server bubblewrap "kernel-modules-extra-${kver}" rke2-server rke2-selinux; \
    systemctl enable NetworkManager.service sshd.service rke2-server.service; \
    rm -rf /run/k3s; \
    rm -rf /opt && ln -s var/opt /opt; \
    dnf clean all; rm -rf /var/cache/dnf /var/lib/dnf /var/log/dnf*.log

# Layer 2: the site configuration RPM (the only thing that changes on day 2).
ARG SITE_RELEASE=3
RUN set -eux; \
    dnf -y --setopt=reposdir=/tmp/build-repos install "edge-site-config-1.0-${SITE_RELEASE}.el10"; \
    systemctl enable edge-site-manifests.service; \
    dnf clean all; rm -rf /tmp/build-repos /var/cache/dnf /var/lib/dnf /var/log/dnf*.log

# Kernel arguments, tmpfiles.d entries, and (from Step 6) the signature policy and key.
COPY rootfs/ /

# Fail the build if any dnf repository URL does not point at Artifact Keeper.
RUN set -eu; \
    bad="$(grep -rhE '^\s*(baseurl|mirrorlist|metalink)\s*=' /etc/yum.repos.d/ | grep -v ':30080/' || true)"; \
    [ -z "$bad" ] || { echo "non-Artifact-Keeper repo URL(s): $bad" >&2; exit 1; }

RUN bootc container lint --fatal-warnings
```

Three things in here are on purpose:

- **Two views of the same registry.** The build runs with `build.repo`, where the host is `host.containers.internal:30080`, but the image ships `edge.repo`, where the host is `10.0.2.2:30080`, which is the address a node sees on our test network. Same repositories, different hostname. The [architecture page](https://brandonrc.github.io/rocky-custom-baremetal-deployment/architecture/#two-views-of-the-same-registry) explains why this matters once signatures come into the picture.
- **The repository check is a build gate.** The base image carries Rocky's stock `rocky*.repo` files with mirrorlist URLs. We delete them, and then we grep for any `baseurl`, `mirrorlist` or `metalink` that does not point at Artifact Keeper. If someone adds a stray upstream URL later, the build fails instead of quietly pulling from the internet.
- **Layer order is a day-2 optimization.** Our first version installed RKE2 and the site RPM in one `RUN`, so bumping the site configuration changed a 72 MB layer. We split it in two, and now a site configuration release changes three small layers that add up to about 8 MB. That is the download every edge node pays for a configuration change, so it was worth getting right.

Two smaller fixes are in the [build notes](https://brandonrc.github.io/rocky-custom-baremetal-deployment/findings-image/): pinning `kernel-modules-extra` to the base kernel, and declaring RKE2's `/var` directories in tmpfiles.d so that `bootc container lint` passes with warnings treated as errors.

There was one image-mode lesson here that only shows up on a booted node. Our first install came up with RKE2 active and every pod stuck in Pending. The cause was canal (RKE2's default network plugin) failing with `mkdir /opt/cni: read-only file system`. The RESF base keeps `/opt` on the read-only image on purpose, and the recipe points `/usr/local` at `/var/usrlocal` but leaves `/opt` alone. No RKE2 package ships anything under `/opt`; only the network plugin writes there, and only at runtime. Neither `podman run` nor `bootc container lint` can catch this, because nothing is read-only inside a container build. The fix is the classic ostree layout: the `/opt` symlink highlighted above, plus a tmpfiles.d entry for `/var/opt/cni/bin`. The image build now checks for that symlink in its smoke test, so it cannot regress without someone noticing.

The `edge-site-config` RPM itself is small, which is how we think site configuration should be. It carries RKE2's `config.yaml` and `registries.yaml`, a MOTD that shows the package release so you can see day 2 land, an insecure-registry drop-in for our plain-HTTP registry, and a demo nginx Deployment that gets seeded into RKE2's auto-deploy directory on every boot. Its [full contents](https://brandonrc.github.io/rocky-custom-baremetal-deployment/architecture/#images-layers-and-tags) are in the docs.

The `registries.yaml` is the piece that ties the workloads back to the registry. Artifact Keeper serves its Docker Hub proxy under a path prefix, and containerd mirrors are host-level, so we use RKE2's `rewrite` to add the prefix:

```yaml
mirrors:
  docker.io:
    endpoint:
      - "http://10.0.2.2:30080"
    rewrite:
      "^(.*)$": "oci-dockerhub-proxy/$1"
```

With this, `docker.io/library/nginx:alpine` becomes a request for `/v2/oci-dockerhub-proxy/library/nginx/manifests/alpine`. Because we mirror all of `docker.io`, RKE2's own system images such as `rancher/rke2-runtime` come through the registry as well. containerd keeps the public endpoint as a fallback, so a real site should also block egress.

Building the RPM is `rpmbuild` in a `rockylinux:10` container (pulled through the quay.io proxy again, with dnf pointed only at Artifact Keeper again), and it takes about five seconds per release. Uploading is one `PUT` per file. One thing to know: if you re-upload a file name that already exists, Artifact Keeper returns **HTTP 409** and keeps the stored file, even if your local bytes are different. Treat each name-version-release as immutable and bump the release. That is good practice anyway.

![The hosted rpm-edge-site repository in Artifact Keeper showing the edge-site-config releases](images/rocky-linux-image-mode-artifact-keeper/04-rpm-edge-site.png)

The final numbers for the edge image are 76 layers and 463 MB compressed, with a dozen packages added on top of the base. The edge build takes under half a minute cold and 8 seconds for a site-configuration bump. Pushes take a few seconds, because the 68 base layers are already in `oci-bootc`. We push `rocky-edge:10.2-3` and `rocky-edge:10.2-4` as immutable tags and point the floating `rocky-edge:10` at the first one with a registry-side `skopeo copy`. Promotion is just moving that tag.

![The oci-bootc repository showing the Rocky Linux image mode base and the rocky-edge tags with their digests and cosign signature tags](images/rocky-linux-image-mode-artifact-keeper/02-oci-bootc-tags.png)

## Step 4: Install it on bare metal with a kickstart

With the image in the registry, deploying a node works the way Enterprise Linux installs have worked for a long time: network-boot the stock installer and hand it a kickstart. There is no custom ISO, no disk image to copy around, and no `bootc-image-builder`. The installer (Anaconda) in Rocky 10.2 knows how to pull a bootc image and deploy it as the payload.

We boot the installer's own `vmlinuz` and `initrd.img` from the Rocky 10.2 mirror's `images/pxeboot/` directory, and we serve the kickstart and a cached copy of the installer's stage2 over HTTP. In a real rack that is a PXE server and a DHCP option. In our harness it is a small HTTP server next to the VM. The kickstart is the interesting part:

```text
#| highlight: 13-20, 22, 25
text
lang en_US.UTF-8
keyboard us
timezone UTC --utc
network --bootproto=dhcp --device=link --activate --hostname=edge-node-01

zerombr
clearpart --all --initlabel --disklabel=gpt
reqpart --add-boot
part / --grow --fstype=xfs

# The registry is plain HTTP (PoC). Anaconda honors registries.conf.
%pre
mkdir -p /etc/containers/registries.conf.d
cat > /etc/containers/registries.conf.d/50-poc-insecure.conf <<'EOR'
[[registry]]
location = "10.0.2.2:30080"
insecure = true
EOR
%end

ostreecontainer --url=10.0.2.2:30080/oci-bootc/rocky-edge:10 --transport=registry

rootpw --lock
sshkey --username=root "ssh-ed25519 AAAA..."

%post
# Same drop-in on the installed system, so bootc upgrade can reach the registry.
mkdir -p /etc/containers/registries.conf.d
cp /etc/containers/registries.conf.d/50-poc-insecure.conf /etc/containers/registries.conf.d/
%end
reboot
```

Notice there is no `%packages` section. The image is the package list. The only per-node inputs are the hostname and the operator's SSH key, and the installer never asks anyone to type a password. The [kickstart flow](https://brandonrc.github.io/rocky-custom-baremetal-deployment/architecture/#kickstart-flow) in the docs shows every step in order.

We learned two things in this step.

**Use `ostreecontainer`, not the newer `bootc` kickstart command.** Both commands validate on Rocky 10.2, and the `bootc` command (which runs `bootc install to-filesystem` under the hood) is where the installer is headed. When we tried it on Rocky 10.2, the installed system had the wrong SELinux labels on the root SSH key and on `/etc/resolv.conf`. sshd could not read the key and NetworkManager could not write DNS, so we could not reach the node at all, and a `restorecon` in `%post` did not fix it. The same image installed with `ostreecontainer` (which runs `ostree container image deploy`) came up correctly labeled with SELinux enforcing. We did not chase the root cause any further. The lesson for a fleet is that you have to test the install path, not just the image.

**Mark the registry insecure in both places.** Anaconda pulls the image inside the installer environment, so the `registries.conf.d` drop-in has to exist there (that is the `%pre`). The installed system needs the same drop-in for `bootc upgrade`, so the `edge-site-config` RPM bakes it into the image, and the kickstart writes it into the installed `/etc` as well. In production you would terminate TLS at Artifact Keeper's Caddy front door and distribute the CA instead. The mechanics are the same.

Here is what happens on power-on:

| Milestone | Time |
|---|---|
| Installer boots, applies the kickstart, pulls and deploys the image, reboots | 65 s |
| SSH answers on the installed system | 25 s after first boot |
| RKE2 node Ready | 61 s after SSH |
| Demo workload Running | shortly after |

The verification script then runs over SSH. This is the output that made the whole exercise feel worth it:

```text
=== bootc status ===
spec:    image: 10.0.2.2:30080/oci-bootc/rocky-edge:10   transport: registry
booted:  version: 10.2-3   imageDigest: sha256:d4f3ec69bbd3...
=== getenforce ===
Enforcing
=== kubectl get nodes -o wide ===
NAME           STATUS   ROLES                VERSION          OS-IMAGE                        KERNEL-VERSION
edge-node-01   Ready    control-plane,etcd   v1.36.5+rke2r1   Rocky Linux 10.2 (Red Quartz)   6.12.0-211.61.1.el10_2.x86_64
=== workload image / imageID ===
default/nginx-demo-...  docker.io/library/nginx:alpine  docker.io/library/nginx@sha256:df221db8...
=== Artifact Keeper: oci-dockerhub-proxy ===
98 cached objects; images: library/nginx, rancher/hardened-calico, rancher/hardened-coredns,
  rancher/hardened-etcd, rancher/hardened-kubernetes, rancher/klipper-helm, rancher/rke2-runtime, ...
OK: nginx-demo runs nginx:alpine @ sha256:df221db8..., and that manifest is cached in oci-dockerhub-proxy
```

Every image the cluster pulled, from `rancher/hardened-calico` to the demo's `nginx:alpine`, is now cached in `oci-dockerhub-proxy`. You can see it in the Artifact Keeper UI right next to the RPMs and the operating system image that produced the node.

![The oci-dockerhub-proxy repository showing nginx and every rancher RKE2 system image cached after the first boot](images/rocky-linux-image-mode-artifact-keeper/03-dockerhub-proxy-cache.png)

## Step 5: Upgrade and roll back by moving a tag

Upgrading the fleet does not touch the nodes. We bump `edge-site-config` to release 4 (a new MOTD and a new label on the demo Deployment), rebuild the edge image, push it as `rocky-edge:10.2-4`, and then **promote** it by copying that tag onto the floating `rocky-edge:10` inside the registry. The copy only touches the manifest and takes about a second, because every blob is already in `oci-bootc`.

```bash
#| command-line
#| data-filter-output: (out)
skopeo copy docker://localhost:30080/oci-bootc/rocky-edge:10.2-4 docker://localhost:30080/oci-bootc/rocky-edge:10
bootc upgrade
(out)layers already present: 73; layers needed: 3 (7.8 MB)
(out)Deploying...done (2 seconds)
(out)Queued for next boot: 10.0.2.2:30080/oci-bootc/rocky-edge:10
(out)  Version: 10.2-4
(out)  Digest: sha256:c98876108f864a75e7a7f958fda049d72673652e8eae5849c63dac893bb38f16
```

The node downloads 7.8 MB, stages the new deployment, and reboots into it. After the reboot, `bootc status` shows 10.2-4 booted with 10.2-3 as the rollback, the MOTD reads `edge-site-config 1.0-4.el10: day-2 update via bootc upgrade (release 4)`, and RKE2 has re-applied the re-seeded manifests, so the Deployment rolled to the release-4 label. `bootc rollback` and a reboot put everything back, manifests included, because the seeding service runs from whichever image is booted. We ran it both directions to be sure. The [day-2 flow](https://brandonrc.github.io/rocky-custom-baremetal-deployment/architecture/#day-2-flow) is diagrammed in the docs.

This step gave us the failure we learned the most from. On an earlier build, the upgrade staged cleanly, the node rebooted, and it came back on the **old** image. The staged deployment was gone, and there was no error in the current journal. The clue was in the next boot's `ostree-boot-complete` output:

```text
ostree-finalize-staged.service failed on previous boot: Finalizing deployment:
Finalizing SELinux policy: Failed to execute child process "/usr/bin/bwrap"
(No such file or directory)
```

Here is what is going on. At shutdown, ostree finalizes the staged deployment. Because `rke2-selinux` adds policy modules, `/etc/selinux` differs from the image's copy, so ostree rebuilds the SELinux policy by running `semodule` inside bubblewrap (a small sandboxing tool). The RESF minimal base does not include the `bubblewrap` package, and no package declares a dependency on it, because libostree calls `/usr/bin/bwrap` by its hard-coded path.

**Why nothing caught it:** our base build never noticed, because the builder container had bubblewrap as a dependency of rpm-ostree, and the community image happens to ship it. We proved the diagnosis on the live node by temporarily overlaying `/usr` with `bootc usr-overlay` and installing the package; the next upgrade went through. The fix is the one extra package you saw in the Containerfile, plus a tmpfiles.d entry for `/var/log/journal` so that the journal survives reboots and the next person does not have to find this in a serial log like we did.

## Step 6: Sign everything, and verify everywhere

Everything up to this point would accept any image the registry handed out. That is fine for a lab and not acceptable for a fleet, so this step changes it. The rule is simple: Artifact Keeper stores the signatures and the public keys next to the artifacts, and every consumer verifies.

| Artifact | Signed by | Signature lives in | Verified by |
|---|---|---|---|
| `edge-site-config` RPM | our GPG key, `rpmsign` in the build container | the RPM header | dnf `gpgcheck=1` during the image build |
| `rpm-edge-site` repodata | Artifact Keeper's managed key | `repodata/repomd.xml.asc` | dnf `repo_gpgcheck=1` |
| Proxied Rocky and RKE2 RPMs | the vendors | RPM headers and upstream `repomd.xml.asc`, passed through the proxy | dnf `gpgcheck=1`, `repo_gpgcheck=1` |
| Base and edge images | our cosign key, by digest, after every push | `oci-bootc`, as a `sha256-<digest>.sig` tag | `policy.json` (the containers-image trust policy) on the build host, in the installer, and on the node |
| Public keys | n/a | the generic repository `raw-edge-keys` | fetched by the build and the kickstart, and shipped in the image |

Because uploads are write-once, the signed rebuild got new release numbers: `edge-site-config` 1.0-3 and 1.0-4, and images `rocky-edge:10.2-3` and `10.2-4`. The unsigned 10.2-1 and 10.2-2 stay in the registry as images the policy now refuses, which turned out to be a convenient test fixture. The [trust model](https://brandonrc.github.io/rocky-custom-baremetal-deployment/architecture/#trust-model) page has the long form of the table above.

### What Artifact Keeper does here

Two things happen natively. Artifact Keeper signs the repodata of a hosted RPM repository with a key you create through its signing API, and it serves `repomd.xml.asc` and `repomd.xml.key`. Its proxies also pass upstream `repomd.xml.asc` through byte for byte, so the Rocky and RKE2 repositories get `repo_gpgcheck=1` without any extra work. The repodata signatures expire after seven days and are re-signed on request, which is good to know if you mirror them. For images, Artifact Keeper stores whatever cosign pushes right next to the image, and its generic repository serves the public keys over anonymous HTTP. So image signing stays in your pipeline with cosign, and Artifact Keeper is the place the signatures and keys live. The [what Artifact Keeper does here](https://brandonrc.github.io/rocky-custom-baremetal-deployment/artifact-keeper/) page spells out each role.

### The cosign format problem

cosign 3 defaults to a new signature format attached through the OCI referrers API (a newer way of hanging metadata off an image). Artifact Keeper stored it correctly and `cosign verify` passed, but podman, skopeo, bootc and Anaconda all said `A signature was required, but no signature exists`. This was pretty confusing for a while. It turns out the containers-image stack still reads only the older format, the `sha256-<digest>.sig` tag. So we sign with the deprecated flags, and the deprecation warning is the real risk in this design:

```bash
#| command-line
#| data-filter-output: (out)
cosign sign --yes --key signing/keys/cosign.key --allow-http-registry --allow-insecure-registry --tlog-upload=false --new-bundle-format=false --use-signing-config=false localhost:30080/oci-bootc/rocky-edge@sha256:d4f3ec69bbd3...
(out)Flag --new-bundle-format has been deprecated, this will be the only supported format in future versions
(out)Pushing signature to: localhost:30080/oci-bootc/rocky-edge
```

`podman push --sign-by-sigstore-private-key` writes the same format natively. We have not switched to it yet, but it is the obvious next move. The [signing notes](https://brandonrc.github.io/rocky-custom-baremetal-deployment/findings-signing/) have the referrers output, the `cosign tree` view, and how the signatures show up in Artifact Keeper.

### The policy that enforces it

The node ships this in `/etc/containers/policy.json`, with the public key at `/etc/pki/containers/edge-cosign.pub` and a `registries.d` entry that turns on `use-sigstore-attachments` for the registry. This is abridged. The default is `reject`, with one exception for registry pulls of our image:

```json
{
  "default": [{"type": "reject"}],
  "transports": {
    "docker": {
      "10.0.2.2:30080/oci-bootc/rocky-edge": [{
        "type": "sigstoreSigned",
        "keyPath": "/etc/pki/containers/edge-cosign.pub",
        "signedIdentity": {"type": "exactRepository",
                           "dockerRepository": "localhost:30080/oci-bootc/rocky-edge"}
      }]
    }
  }
}
```

Two lines in there are not obvious. `signedIdentity` is required because cosign records a tag-less identity that the default matching rule never accepts. And it names `localhost:30080`, which is the signer's view of the registry, not `10.0.2.2:30080`, which is the node's view. In production you would give the registry one DNS name everywhere, and this collapses to `matchRepository`. The real file also keeps local transports such as `containers-storage` open, which the excerpt leaves out.

The kickstart writes the same three files in `%pre` before `ostreecontainer` runs, fetching the public key from Artifact Keeper, and the build host gets a user-level copy so that `podman build FROM` our base is checked too. This led to the finding of this step: **the kickstart's `--no-signature-verification` flag is not the switch.** With our `%pre` policy in place, an unsigned image is refused even with the flag added back. With the installer's stock policy, an unsigned image installs fine without it. The policy file is what enforces signatures, for the installer and for `bootc upgrade` alike. The build gate from Step 3 grew as well; it now fails on `gpgcheck=0` or a missing `gpgkey` in addition to foreign URLs.

### Proof, both directions

We tested each consumer with an image that is not signed:

```text
# podman build FROM the unsigned base, on the build host
Error: creating build container: unable to copy from source docker://localhost:30080/oci-bootc/rocky-bootc-base:unsigned:
Source image rejected: A signature was required, but no signature exists

# kickstart install of rocky-edge:unsigned-test, 40 seconds in
error: Performing deployment: Preparing import: Fetching manifest:
failed to invoke method OpenImage: A signature was required, but no signature exists

# bootc upgrade on the node after promoting the unsigned image to :10
error: Upgrading: Preparing import: Fetching manifest:
failed to invoke method OpenImage: A signature was required, but no signature exists
```

After the refused upgrade, the node stayed on its signed deployment, nothing was staged, and RKE2 never noticed anything happened. Images signed with the wrong key, unsigned RPMs, and repodata signed with a key the client does not hold all fail too. The [troubleshooting page](https://brandonrc.github.io/rocky-custom-baremetal-deployment/troubleshooting/) lists each error. Then we ran the signed images through the full install, upgrade and rollback again, this time with signatures checked at every hop. Promotion is still a plain tag copy. The signature is bound to the digest, so `rocky-edge:10` verifies the moment it points at a signed build.

What is still missing is TLS. Signatures protect the content end to end over plain HTTP, but tag-to-digest resolution, the unsigned EPEL metadata, and the first fetch of the public key all trust the network. Terminating TLS at Artifact Keeper's Caddy front door and distributing its CA is the next step. It touches every registry address in the pipeline, which is why we kept it separate. The [adapting page](https://brandonrc.github.io/rocky-custom-baremetal-deployment/adapting/) lists what changes.

## What we learned

**The registry-as-source-of-truth idea holds up.** By the end, Artifact Keeper held the base operating system packages, the Kubernetes packages, our site package, the base image, the edge image releases, their signatures, the public keys, and every container image the cluster pulled. Every pull we could trace was served by Artifact Keeper. "What is running on node 37?" is now `bootc status` on the node and a digest lookup in one registry. Promotion is a tag copy, rollback is built in, a site configuration change costs each node 7.8 MB, and every RPM and image (and all repodata whose publisher signs it) is verified at the point of use.

**Image mode fails differently than package mode.** Neither of our two real bugs showed up in `podman build`, `podman run`, or `bootc container lint`. A read-only `/opt` only matters once the root filesystem is actually read-only, and a missing finalize-time dependency only matters at the first upgrade. If you adopt image mode, your CI needs a stage that boots the image and upgrades it, not just builds it. The VM harness in the repo is that stage, and it is the part we would keep even if we changed everything else.

**Build the base from the recipe.** Composing the Rocky bootc base ourselves took one wrapper Containerfile and under five minutes, and it is the reason every package in the operating system can be traced to one registry. The image-mode details we hit along the way (a kickstart command to avoid on 10.2, a read-only `/opt`, a `bubblewrap` package to add) are all captured in the repo so you do not have to rediscover them.

**Artifact Keeper is what makes the one-place story work.** Its RPM proxies put the operating system and Kubernetes packages behind one address without mirroring anything by hand. Its hosted RPM repository generates and signs the metadata for our own package. Its OCI registry holds the base image, the edge image releases, and their cosign signatures, and its Docker Hub and quay.io proxies catch every workload image the cluster pulls. Its generic repository serves the public keys. Promotion is a tag copy inside the registry, and `bootc upgrade`, dnf, podman and the installer all verify against it.

From here, the hardening is deployment choices rather than new machinery: terminate TLS at Artifact Keeper's Caddy front door and distribute the CA; give the registry one DNS name so the signing policy is a plain `matchRepository`; install nodes by digest and let them track the tag; serve the installer kernel, initrd and stage2 from the registry too; and block egress at the edge. The [adapting page](https://brandonrc.github.io/rocky-custom-baremetal-deployment/adapting/) walks through each one.

Power-on to a working, signed, verified Kubernetes node is about three minutes. The slowest part of the install turned out to be downloading the installer's 750 MB stage2 from the mirror, so the harness caches and serves that locally too. The full numbers, stage by stage, are on the [timings page](https://brandonrc.github.io/rocky-custom-baremetal-deployment/timings/).

Everything, including the harness that produced these numbers, is in [github.com/brandonrc/rocky-custom-baremetal-deployment](https://github.com/brandonrc/rocky-custom-baremetal-deployment), with setup instructions and the complete findings on its [documentation site](https://brandonrc.github.io/rocky-custom-baremetal-deployment/). `make all` generates keys, builds, signs and pushes everything. `make vm-all` walks a node through install, verification, a refused unsigned upgrade, a real upgrade, and rollback.

## What's next for Artifact Keeper

Dogfooding Artifact Keeper this hard produced a handful of improvements, which is part of why we do these projects. The compose and documentation fixes are already in, so the quickstart you land on matches what this post describes. The [1.11.0 release](https://github.com/artifact-keeper/artifact-keeper/issues/4468) adds signature-aware views for OCI repositories, so cosign signatures show up on the image they belong to instead of as separate tags, corrects the storage accounting for shared layers, and adds server-side image signing, which turns the cosign step in Step 6 into a registry setting. The full list of what came out of this project is on the [Artifact Keeper notes](https://brandonrc.github.io/rocky-custom-baremetal-deployment/artifact-keeper-notes/) page.

---

If your deployments have grown complicated and you think one registry could simplify them, or you try this in a home lab, I would like to hear about it: bgeraci@openteams.com.
