---
title: "Running Kali Linux Containers on a Linux Server with Docker"
date: 2025-06-08T18:46:01+08:00
draft: false

categories: ['Tutorial']
tags: ['linux', 'docker', 'linux server']
toc: false
author: ""
---

Kali Linux ships official OCI images under `kalilinux/kali-rolling`, `kalilinux/kali-last-release`, and `kalilinux/kali-bleeding-edge`. They are deliberately minimal, containing little beyond the base userland, so tooling is layered on top through Kali's metapackages. That makes them well suited to disposable, reproducible assessment environments on a remote server: you get a clean toolchain per engagement, isolated from the host and from each other, without maintaining a full Kali VM. Use them only against systems you are authorized to test.

## Installing Docker Engine on the Host

The commands below target a Debian or Ubuntu host and use Docker's own apt repository rather than the distribution's `docker.io` package, which lags upstream. Conflicting packages should be removed first (`docker.io`, `docker-compose`, `podman-docker`, `containerd`, `runc`). If the host is itself Kali, point the repository at Debian, not at `kali-rolling`; Docker publishes no repository for Kali, and the `$VERSION_CODENAME` expansion would resolve to a codename that does not exist upstream. In that case hardcode the Debian stable codename.

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/$(. /etc/os-release && echo "$ID")/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
https://download.docker.com/linux/$(. /etc/os-release && echo "$ID") \
$(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
sudo docker run --rm hello-world
```

Membership in the `docker` group grants effective root on the host, since anyone who can talk to the daemon socket can bind-mount `/` into a container. On a shared server, either restrict the group to administrators or run the daemon rootless via `dockerd-rootless-setuptool.sh install`. Rootless mode constrains what a compromised container can reach, at the cost of some networking limitations that matter for scanning workloads, discussed below.

## Pulling and Running a Kali Container

The base image is roughly 100 MB compressed and contains almost no security tooling.

```bash
docker pull kalilinux/kali-rolling
docker run -it --rm kalilinux/kali-rolling /bin/bash
```

Inside, refresh the package index and install a metapackage appropriate to the use case. `kali-linux-headless` is the tool set of the default install minus the GUI applications and is the sensible choice for a server; `kali-linux-default` adds the desktop-oriented tools, and `kali-tools-web`, `kali-tools-passwords`, `kali-tools-wireless`, `kali-tools-information-gathering`, and similar per-category packages allow a narrower footprint. The headless set pulls several gigabytes, so prefer category metapackages or individual tools when disk or bandwidth is constrained.

```bash
apt update && apt -y install kali-linux-headless
```

Running with `--rm` discards the container's writable layer on exit, which is correct for throwaway work but destroys anything installed interactively. For anything you intend to keep, bake the tooling into an image instead of relying on container state.

## Building a Reproducible Image

A Dockerfile makes the environment declarative and cacheable. Collapsing `apt-get update` and `install` into a single `RUN` layer avoids stale-index failures, and cleaning `/var/lib/apt/lists` keeps the layer from carrying the package cache.

```dockerfile
FROM kalilinux/kali-rolling
ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update \
 && apt-get install -y --no-install-recommends \
      kali-tools-information-gathering kali-tools-web \
      kali-tools-vulnerability seclists \
      tmux vim-tiny iputils-ping dnsutils iproute2 openssh-client \
 && apt-get clean && rm -rf /var/lib/apt/lists/*

RUN useradd -m -s /bin/bash operator
USER operator
WORKDIR /home/operator
CMD ["/bin/bash"]
```

```bash
docker build -t kali-ops:latest .
```

Running as an unprivileged user inside the container is worth the minor friction. Tools that need raw sockets can be granted the specific capabilities they require at `docker run` time rather than running the whole session as root. Because `kali-rolling` is a rolling release, rebuilds are not bit-reproducible across dates. If you need that, pin by digest (`FROM kalilinux/kali-rolling@sha256:...`) or use `kali-last-release`, and rebuild on a schedule so the image tracks upstream security updates.

## Capabilities, Networking, and Persistence

Docker's default capability set omits `NET_RAW` only in some configurations and always omits `NET_ADMIN`, so SYN scans, ARP-level tooling, packet crafting, and interface manipulation commonly need explicit grants. Prefer granting only those two over `--privileged`, which disables nearly all isolation, exposes host devices, and should be avoided.

```bash
docker run -it --name engagement-01 \
  --cap-add NET_RAW --cap-add NET_ADMIN \
  -v "$PWD/engagements/01:/work" \
  kali-ops:latest
```

On the default bridge network, the container's traffic is NATed through the host, which is usually adequate for scanning remote targets over routed networks. Layer 2 work against the server's own segment, or tools that depend on seeing the host's interfaces directly, require `--network host`, which removes network namespace isolation entirely and gives the container the host's interfaces and ports. Treat that as a deliberate escalation. Rootless Docker complicates both cases, since it uses a userspace network stack (slirp4netns or pasta) that adds overhead and cannot forward raw packets the way the kernel bridge can, so a rootless daemon and raw-socket-heavy scanning are in tension.

Persistence belongs in bind mounts or named volumes, not in the container's writable layer. A bind mount to `/work` keeps scan output, wordlists, and notes on the host where they can be backed up independently of image rebuilds. Named volumes (`-v engagement01:/work`) are managed by Docker and avoid host UID mismatches, which otherwise surface as root-owned files on the host when the container runs as root.

## Long-Lived Containers with Compose

For a container that should survive disconnects and reboots, define it in Compose and attach with `docker exec`, using `tmux` inside so sessions persist independently of your SSH connection.

```yaml
services:
  kali:
    image: kali-ops:latest
    container_name: kali-ops
    hostname: kali-ops
    command: sleep infinity
    tty: true
    stdin_open: true
    restart: unless-stopped
    cap_add:
      - NET_RAW
      - NET_ADMIN
    volumes:
      - ./work:/home/operator/work
    mem_limit: 4g
    cpus: 2.0
```

```bash
docker compose up -d
docker exec -it kali-ops tmux new -As main
```

The `sleep infinity` command keeps PID 1 alive so the container acts as a persistent workstation, and the resource limits prevent a runaway scan or hash-cracking job from starving the host. Drop the limits on a dedicated box if the workload is compute-bound.

## Operational Hardening

Several host-level details deserve attention before exposing any of this beyond a private network. Published ports (`-p 8080:80`) are inserted into Docker's own iptables chains ahead of UFW and most firewalld rules, so a host firewall that appears to block a port may not block a published container port; bind explicitly to loopback (`-p 127.0.0.1:8080:80`) or manage the `DOCKER-USER` chain for filtering that Docker will not override. Avoid running an SSH daemon inside the container; `docker exec` over an SSH session to the host gives the same access with one authentication surface instead of two. Exploitation frameworks that open listeners, such as reverse-shell handlers, should publish only the specific port they need, bound to the specific interface they need. Keep the Docker daemon's TCP socket disabled, and if remote daemon access is required, use `docker context` over SSH rather than exposing 2375 or 2376.

Updating is a rebuild, not an in-place upgrade: `docker build --pull --no-cache -t kali-ops:latest .`, then `docker compose up -d` recreates the container from the new image while the bind-mounted work directory carries state across. Periodic `docker system prune` and `docker image prune` calls reclaim the disk consumed by superseded multi-gigabyte layers, which accumulate quickly with Kali tool sets.
