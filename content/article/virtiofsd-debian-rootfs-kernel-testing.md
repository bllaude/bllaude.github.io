---
title: "Set up rootfs of a Debian VM with virtiofsd for Linux Kernel Testing"
date: 2025-01-10T21:27:26+08:00
draft: false

categories: []
tags: ['linux', 'kernel', 'qemu', 'virtiofs', 'debian', 'testing']
toc: false
author: ""
---
This post builds a Debian 13 (trixie) rootfs with `mmdebstrap`, exports it with `virtiofsd`, boots it under QEMU with a locally built kernel, and arranges for modules, selftests and the kernel tree itself to appear in the guest without any image rebuild step. 

Debian is a good fit. Why? because its minimal bootstrap is small, its package set covers nearly every kernel-adjacent tool (`linux-perf`, `bpftool`, `trace-cmd`, `stress-ng`), and it is the userspace most existing kernel CI tooling, syzkaller included, already assumes.

## How the pieces fit

virtiofs is a FUSE dialect carried over virtio. In the guest, `fs/fuse/virtio_fs.c` registers a virtio driver that sends FUSE requests over virtqueues instead of `/dev/fuse`. On the host, `virtiofsd` is a vhost-user device backend: QEMU connects to its UNIX socket, passes it the guest memory mappings, and from then on the daemon reads requests out of the virtqueues and performs the filesystem operations itself, bypassing the QEMU main loop. This has two configuration consequences that account for most first-boot failures. Guest RAM has to be a shareable mapping (`memory-backend-memfd` or `memory-backend-file` with `share=on`), because the daemon must be able to map the buffers the guest posts. And the daemon's lifetime is bound to the QEMU connection, since the Rust implementation exits when its peer disconnects, so each VM launch needs a fresh daemon and a wrapper script is the natural unit of automation.

Using it as the root filesystem works because `init/do_mounts.c` accepts a non-block `root=` value when `rootfstype=` names a filesystem that does not need a block device; the same mechanism serves 9p and tmpfs roots. The value of `root=` is the virtiofs tag chosen on the QEMU command line, and `rootfstype=virtiofs` makes the kernel try that driver directly instead of probing for disks. Because mounting happens before any module can be loaded from the root filesystem, `CONFIG_VIRTIO_FS` and `CONFIG_FUSE_FS` must be built in rather than modular, a constraint that comes up again in the kernel configuration section.

## Host prerequisites

On a Debian 13 host the required packages are `qemu-system-x86`, `virtiofsd`, `mmdebstrap` and `qemu-utils`; `debootstrap` works equally well and its command line is given as an alternative below. Debian ships the Rust `virtiofsd`, and unlike the C daemon that used to live inside the QEMU tree its installed outside `$PATH`, under `/usr/libexec/`. Confirm the location on your system with `dpkg -L virtiofsd`, since the path has moved between releases and derivatives (Ubuntu in particular has shipped it under `/usr/lib/qemu/` or `/usr/libexec/` depending on release). Hardware virtualization via `/dev/kvm` is assumed; under TCG both kernel testing and virtiofs throughput are poor enough that the setup is not worth using.

```sh
sudo apt install qemu-system-x86 qemu-utils virtiofsd mmdebstrap
dpkg -L virtiofsd | grep -E 'libexec|bin'
ls -l /dev/kvm
```

## Building the rootfs directory

`mmdebstrap` builds a Debian tree into a directory and, unlike `debootstrap`, resolves and fetches packages in a single pass, which makes repeated rebuilds quick, and it can reuse a local apt cache through `--aptopt` or an apt-cacher in front of the mirror. Running it as real root preserves ownership and special files correctly, which matters because `virtiofsd` will later export those exact uids and modes to the guest; the unprivileged `--mode=unshare` variant produces a tree owned by a subuid range and is not appropriate here. The kernel and bootloader packages are intentionally excluded: the kernel under test is loaded directly by QEMU from the build tree, so a `linux-image` package in the guest would only add an unused `/lib/modules` directory that conflicts with the one produced by `modules_install`.

```sh
export ROOTFS=/srv/vm/debian-rootfs
sudo mkdir -p "$ROOTFS"

sudo mmdebstrap --variant=minbase --format=directory \
    --include=systemd-sysv,udev,dbus,kmod,iproute2,iputils-ping,openssh-server,\
procps,less,vim-tiny,ca-certificates,gdb,strace,linux-perf,bpftool,trace-cmd,\
build-essential,git,stress-ng \
    trixie "$ROOTFS" http://deb.debian.org/debian
```

The equivalent with `debootstrap` is `sudo debootstrap --variant=minbase --include=<same list> trixie "$ROOTFS" http://deb.debian.org/debian`, which is slower because it unpacks and configures sequentially but is available on every Debian-derived host. The `minbase` variant keeps the tree to a few hundred megabytes; the extra packages are a pragmatic debugging baseline: `kmod` provides `modprobe`, `insmod` and `depmod` for the modules installed from the build tree, `linux-perf` and `bpftool` allow tracing the kernel from the inside, `trace-cmd` drives ftrace, `gdb` and `strace` help reproduce failures in userspace, and `stress-ng` is a convenient load generator for shaking out races. `build-essential` lets out-of-tree modules and selftests be compiled in the guest against the exported kernel tree when cross-building on the host is inconvenient. A networking developer would add `tcpdump` and `iperf3`, a filesystem developer `xfsprogs`, `btrfs-progs` and `fio`.

## Making the tree bootable

A bootstrapped tree is not yet something systemd will bring up unattended on a serial console. The adjustments are small and can be made directly on the host through the directory, with `chroot` only for the steps that need a running `systemctl`. The root account needs an empty password and an autologin getty on `ttyS0`. Debian's serial getty comes from `serial-getty@.service`, which the getty generator instantiates automatically when the kernel command line carries `console=ttyS0`, so only a drop-in overriding `ExecStart` is required. Debian's `agetty` lives at `/sbin/agetty` (the `/sbin` to `/usr/sbin` merge makes this path valid either way).

```sh
sudo chroot "$ROOTFS" /bin/bash -eu <<'EOF'
passwd -d root
echo kvm-test > /etc/hostname
printf '127.0.0.1 localhost kvm-test\n' > /etc/hosts
: > /etc/fstab

mkdir -p /etc/systemd/system/serial-getty@ttyS0.service.d
cat > /etc/systemd/system/serial-getty@ttyS0.service.d/autologin.conf <<'UNIT'
[Service]
ExecStart=
ExecStart=-/sbin/agetty -o '-p -- \\u' --noclear --autologin root %I $TERM
UNIT

mkdir -p /etc/systemd/network
cat > /etc/systemd/network/20-wired.network <<'NET'
[Match]
Name=en* eth*
[Network]
DHCP=yes
NET
systemctl enable systemd-networkd.service ssh.service

mkdir -p /etc/systemd/journald.conf.d
printf '[Journal]\nStorage=volatile\n' > /etc/systemd/journald.conf.d/volatile.conf

mkdir -p /etc/ssh/sshd_config.d
printf 'PermitRootLogin yes\nPermitEmptyPasswords yes\n' > /etc/ssh/sshd_config.d/10-test.conf
EOF

# QEMU's user-mode network always offers its resolver at 10.0.2.3
echo 'nameserver 10.0.2.3' | sudo tee "$ROOTFS/etc/resolv.conf" >/dev/null
```

Emptying `/etc/fstab` stops systemd from waiting on mount entries that cannot exist in this topology, since the root filesystem is supplied by the host and there is nothing to `fsck` or remount. Keeping the journal volatile avoids journald writing continuously to a filesystem that is simultaneously being modified from the host, which amplifies small writes and occasionally produces confusing short-read behaviour while tests run. Writing a static `resolv.conf` sidesteps `systemd-resolved`, which Debian 13 has split into a separate package that `minbase` does not pull in; if you prefer the stub resolver, add `systemd-resolved` to the package list and symlink `/etc/resolv.conf` to its stub file instead. The sshd drop-in with empty passwords is acceptable only because the guest is reachable solely through the user-mode network forward bound to `127.0.0.1` shown later; for any broader exposure, install a public key in `/root/.ssh/authorized_keys` and remove the drop-in.

## Configuring the kernel

The options below are the ones specific to this arrangement; start from `make defconfig && make kvm_guest.config` for the virtio base. Everything required to reach the root filesystem is built in, and anything under test that can be a module should be one, because modules are the cheapest iteration path: after `make modules_install` the new `.ko` files are present in the guest's `/lib/modules/$(uname -r)` the moment the host copy completes.

```
CONFIG_VIRTIO=y
CONFIG_VIRTIO_PCI=y
CONFIG_FUSE_FS=y
CONFIG_VIRTIO_FS=y
CONFIG_DAX=y
CONFIG_FUSE_DAX=y              # optional; only used if QEMU is given cache-size=
CONFIG_VIRTIO_NET=y
CONFIG_SERIAL_8250=y
CONFIG_SERIAL_8250_CONSOLE=y
CONFIG_DEVTMPFS=y
CONFIG_DEVTMPFS_MOUNT=y
CONFIG_TMPFS=y
CONFIG_TMPFS_POSIX_ACL=y
CONFIG_TMPFS_XATTR=y
CONFIG_CGROUPS=y               # systemd on trixie expects the unified cgroup v2 hierarchy
CONFIG_CGROUP_BPF=y
CONFIG_AUTOFS_FS=y
CONFIG_DEBUG_INFO_DWARF5=y
CONFIG_GDB_SCRIPTS=y
CONFIG_RANDOMIZE_BASE=n        # or pass nokaslr on the command line
```

The cgroup and autofs lines are worth calling out because Debian's current systemd is considerably less forgiving of a stripped-down kernel than the systemd in older releases: it requires cgroup v2 and will refuse or degrade noisily without it, and a missing `CONFIG_AUTOFS_FS` produces mount-unit failures that look unrelated to the kernel configuration. If boot reaches systemd and then stops at `Failed to mount` messages, `systemd-analyze` is not yet available, but the missing option is usually named in the first dozen lines of console output.

```sh
make -j"$(nproc)" bzImage modules
sudo make INSTALL_MOD_PATH="$ROOTFS" modules_install
```

`INSTALL_MOD_STRIP=1` reduces module size at the cost of debug info; keep the unstripped `vmlinux` and module objects on the host for debugging regardless. `modules_install` runs `depmod` against the target prefix, so `modprobe` works in the guest immediately. It also leaves `build` and `source` symlinks that point at the host's absolute kernel tree path, which dangle in the guest unless the tree is exported and mounted at the same path, as described later.

## Launching virtiofsd

The daemon has to be listening before QEMU starts, since QEMU connects to the socket while realizing the device. When exporting a rootfs it should run as root so that ownership, setuid bits, file capabilities and device nodes round-trip faithfully; the unprivileged user-namespace mode remaps ids and breaks anything that depends on real `chown` or `mknod` semantics.

```sh
sudo /usr/libexec/virtiofsd \
    --socket-path=/run/vm/vfsd-root.sock \
    --socket-group=kvm \
    --shared-dir="$ROOTFS" \
    --tag=myroot \
    --sandbox=namespace \
    --cache=auto \
    --xattr \
    --posix-acl \
    --announce-submounts
```

Each option has a reason. `--xattr` enables extended attribute passthrough, which Debian's boot path needs for file capabilities (`security.capability`, used by `ping` on systems that do not ship it setuid) and for tmpfiles and journal handling; without it those operations fail with `ENOTSUP` and produce errors that are misleading. `--posix-acl` covers the ACLs that `systemd-tmpfiles` and journald apply. `--cache` sets the coherency policy: `auto` revalidates attributes and data on a short timeout, which suits a workflow where the host modifies the tree while the guest runs, as `modules_install` against a live VM does; `never` gives strict coherency and is the right choice when chasing a suspected stale-cache bug; `always` is fastest but assumes only the guest writes. `--sandbox=namespace` puts the daemon in its own mount, pid and network namespaces with the shared directory as its root, so a malicious or buggy guest cannot traverse out of the export, while `chroot` is the weaker fallback where namespaces are unavailable. `--announce-submounts` makes mount points beneath the shared directory appear as distinct superblocks in the guest, avoiding inode number collisions. `--socket-group=kvm` lets a QEMU running as an ordinary member of the `kvm` group connect to a socket created by the root-owned daemon, so only the daemon needs elevated privileges, not the emulator.

## Launching QEMU

The command line must express shared guest memory, the vhost-user-fs device bound to the daemon's socket, direct kernel boot parameters and a serial console. The size of the memory backend has to equal `-m`; a mismatch makes QEMU abort with a vhost-user error at device realization.

```sh
qemu-system-x86_64 \
    -enable-kvm -cpu host -smp 4 -m 4G \
    -object memory-backend-memfd,id=mem,size=4G,share=on \
    -numa node,memdev=mem \
    -chardev socket,id=char0,path=/run/vm/vfsd-root.sock \
    -device vhost-user-fs-pci,chardev=char0,tag=myroot \
    -kernel arch/x86/boot/bzImage \
    -append 'root=myroot rootfstype=virtiofs rw console=ttyS0 nokaslr init=/lib/systemd/systemd' \
    -netdev user,id=n0,hostfwd=tcp:127.0.0.1:2222-:22 \
    -device virtio-net-pci,netdev=n0 \
    -nographic -no-reboot \
    -s
```

`root=myroot` must equal `tag=` exactly, and `rw` is required because a non-block root defaults to read-only and systemd will not boot usefully in that state. The explicit `init=/lib/systemd/systemd` documents the intended init and keeps the guest from falling back to `/sbin/init` in a tree that happens to lack the `systemd-sysv` symlink. `-s` opens the gdbstub on TCP 1234; with `nokaslr` set, `gdb vmlinux` followed by `target remote :1234` lines up symbols with the running kernel, and sourcing `vmlinux-gdb.py` adds the kernel's helpers such as `lx-dmesg` and `lx-ps`. `-no-reboot` turns a panic into a QEMU exit rather than an endless reboot loop, which is what scripted bisection wants.

## A launch wrapper

Since the daemon dies when the guest disconnects, the two processes should start and stop together. This wrapper makes the edit, build, boot cycle a single command and cleans up the socket on every exit path.

```sh
#!/usr/bin/env bash
set -euo pipefail
ROOTFS=${ROOTFS:-/srv/vm/debian-rootfs}
RUN=/run/vm
SOCK=$RUN/vfsd-root.sock
sudo install -d -m 0775 -g kvm "$RUN"
sudo rm -f "$SOCK"

sudo /usr/libexec/virtiofsd --socket-path="$SOCK" --socket-group=kvm \
    --shared-dir="$ROOTFS" --tag=myroot --sandbox=namespace \
    --cache=auto --xattr --posix-acl --announce-submounts &
VFSD=$!
trap 'sudo kill $VFSD 2>/dev/null || true; sudo rm -f "$SOCK"' EXIT
while [[ ! -S $SOCK ]]; do sleep 0.1; done

qemu-system-x86_64 "$@" \
    -enable-kvm -cpu host -smp 4 -m 4G \
    -object memory-backend-memfd,id=mem,size=4G,share=on -numa node,memdev=mem \
    -chardev socket,id=char0,path="$SOCK" -device vhost-user-fs-pci,chardev=char0,tag=myroot \
    -kernel arch/x86/boot/bzImage \
    -append 'root=myroot rootfstype=virtiofs rw console=ttyS0 nokaslr init=/lib/systemd/systemd' \
    -netdev user,id=n0,hostfwd=tcp:127.0.0.1:2222-:22 -device virtio-net-pci,netdev=n0 \
    -nographic -no-reboot
```

## Exporting the kernel tree and selftests

A second `virtiofsd` instance with its own socket and tag can export the kernel source and build directory. Adding a second `-chardev` and `-device vhost-user-fs-pci,tag=ksrc` pair to QEMU and mounting it in the guest at the same absolute path as on the host repairs the dangling `build` and `source` symlinks, which in turn makes out-of-tree module builds and `perf` with kernel debuginfo work in the guest. The mount is declared with a systemd unit whose file name is the escaped form of the mount point, which `systemd-escape -p --suffix=mount /home/user/linux` computes.

```ini
# $ROOTFS/etc/systemd/system/home-user-linux.mount
[Unit]
Description=Kernel tree from host
[Mount]
What=ksrc
Where=/home/user/linux
Type=virtiofs
[Install]
WantedBy=multi-user.target
```

Selftests are better built on the host and installed into the rootfs, which keeps the guest free of build state and makes runs reproducible. Because they run against the same Debian userspace every time, a failure that appears only in the guest is attributable to the kernel under test rather than to drift in the test environment.

```sh
make -C tools/testing/selftests TARGETS="net bpf mm" \
     INSTALL_PATH="$ROOTFS/opt/kselftest" install
```

Inside the guest, `/opt/kselftest/run_kselftest.sh -c net` executes the collection against the kernel that was just booted. Logs written beneath the shared directory are readable on the host while the VM is still running, and KUnit suites built as modules follow the same pattern: `modprobe` the module and read the TAP stream from `dmesg`.

## Debian-specific details worth knowing

Packages can be installed into the live guest with `apt`, using the user-mode network, and the resulting changes to `/var/lib/dpkg` and `/var/cache/apt` land directly in the host's rootfs directory, so a package added once in a throwaway session persists for every later boot; for a clean baseline, take a reflink copy (`cp -a --reflink=always`) of the pristine tree on btrfs or XFS and point the daemon at the copy for experiments. Because apt and dpkg write heavily, `--cache=auto` with a volatile journal is the least surprising combination, and `/var/cache/apt/archives` can be bind-mounted from the host to avoid refetching packages. Debian's `unattended-upgrades` and `apt-daily` timers are not present in a `minbase` tree, but if you add them, mask them: a background `apt` run during a timing-sensitive test is a classic source of unexplained variance. Finally, `dpkg` triggers such as `ldconfig` and `update-initramfs` run inside the guest when packages are installed there; `update-initramfs` has nothing to do without a `linux-image` package and will not be invoked, which is the intended outcome of leaving kernel packages out.

## Failure modes worth recognizing

A boot that stops at `VFS: Unable to mount root fs on unknown-block(0,0)` means either `CONFIG_VIRTIO_FS=y` and `CONFIG_FUSE_FS=y` are not built in or `root=` does not match the QEMU `tag`; look for `virtio_fs` lines in the kernel log before assuming the first cause. A vhost-user error from QEMU at startup is almost always a missing `share=on` on the memory backend, a size mismatch between `-m` and the backend, or a socket that did not yet exist because the daemon was still initializing, which is what the wait loop in the wrapper exists to prevent. `Permission denied` when QEMU opens the socket points at `--socket-group` not matching a group the invoking user belongs to. A boot that reaches systemd and then drops to emergency mode with capability or permission errors usually means the daemon ran without `--xattr` or without root privileges. Stale file contents in the guest after a `modules_install` on a running VM indicate `--cache=always` and should be `auto` or `never`. When the cause is unclear, start the daemon with `--log-level debug`, which prints every FUSE opcode and shows quickly whether a guest-visible failure originates in the kernel's virtiofs client or in the daemon's sandboxed view of the filesystem.


The loop reduces to `make`, `make modules_install` when modules changed, and a relaunch of the wrapper, with boot to a root prompt in a few seconds because there is no initramfs to unpack and no block device to probe. The userspace is regenerated from a single `mmdebstrap` command and a short set of files, inspected and repaired from the host without attaching a disk, and snapshotted with a reflink copy. The approach is a poor fit only where the root filesystem is itself the subject of the test (virtiofs, FUSE, or anything depending on block-level semantics of the root device), in which case keep a conventional disk image for root and use virtiofs for the source tree and selftest exports.