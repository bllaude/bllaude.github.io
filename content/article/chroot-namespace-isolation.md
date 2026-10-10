---
title: "chroot with namespace isolation"
date: 2026-05-08T22:05:54+08:00
draft: false

categories: []
tags: []
toc: false
author: ""
---
This is aimed at being *'chroot alone isn’t isolation'*, building qc, a small C runtime that uses `pivot_root` plus mount, PID, UTS, IPC, network, user and cgroup namespaces to run unprivileged. Compiles cleanly with `-Wall -Wextra`

Follow project on [Github](https://github.com/bllaude/quick-containers)

<!--more-->

I'm also going to list what’s missing compared with a real runtime: networking, cgroup limits, seccomp and capabilities, overlayfs and a minimal init.

Sources: none; this is written from kernel man-page knowledge (`namespaces(7)`, `user_namespaces(7)`, `pivot_root(2)`, `clone(2)`).

## The code
```c
// qc.c — quick containers. cc -O2 -Wall -o qc qc.c
#define _GNU_SOURCE
#include <sched.h>
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/mman.h>
#include <sys/mount.h>
#include <sys/syscall.h>
#include <sys/wait.h>

#define die(m) do { perror(m); _exit(1); } while (0)
#define STACK_SZ (1024 * 1024)

struct cfg { const char *root; char **argv; int sync[2]; };

static void put(const char *path, const char *s) {
    int fd = open(path, O_WRONLY);
    if (fd < 0 || write(fd, s, strlen(s)) < 0) die(path);
    close(fd);
}

static int child(void *arg) {
    struct cfg *c = arg;
    char b;
    close(c->sync[1]);
    if (read(c->sync[0], &b, 1) != 1) die("sync");   // wait for id maps
    close(c->sync[0]);

    if (sethostname("quick", 5))                        die("sethostname");
    if (mount(NULL, "/", NULL, MS_REC | MS_PRIVATE, NULL)) die("make-private");
    if (mount(c->root, c->root, NULL, MS_BIND | MS_REC, NULL)) die("bind-root");
    if (chdir(c->root))                                 die("chdir");
    if (syscall(SYS_pivot_root, ".", "."))              die("pivot_root");
    if (umount2(".", MNT_DETACH))                       die("umount-old");
    if (chdir("/"))                                     die("chdir /");
    if (mount("proc", "/proc", "proc",
              MS_NOSUID | MS_NODEV | MS_NOEXEC, NULL))  die("mount-proc");

    clearenv();
    setenv("PATH", "/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin", 1);
    setenv("HOME", "/root", 1);
    execvp(c->argv[0], c->argv);
    die("exec");
    return 1;
}

int main(int argc, char **argv) {
    if (argc < 3) { fprintf(stderr, "usage: %s <rootfs> <cmd> [args...]\n", argv[0]); return 2; }
    struct cfg c = { .root = argv[1], .argv = &argv[2] };
    uid_t uid = getuid(); gid_t gid = getgid();
    if (pipe(c.sync)) die("pipe");

    char *stack = mmap(NULL, STACK_SZ, PROT_READ | PROT_WRITE,
                       MAP_PRIVATE | MAP_ANONYMOUS | MAP_STACK, -1, 0);
    if (stack == MAP_FAILED) die("mmap");

    int flags = CLONE_NEWUSER | CLONE_NEWNS | CLONE_NEWPID | CLONE_NEWUTS |
                CLONE_NEWIPC | CLONE_NEWNET | CLONE_NEWCGROUP | SIGCHLD;
    pid_t pid = clone(child, stack + STACK_SZ, flags, &c);
    if (pid < 0) die("clone");

    char path[64], map[64];
    snprintf(path, sizeof path, "/proc/%d/setgroups", pid); put(path, "deny");
    snprintf(path, sizeof path, "/proc/%d/uid_map", pid);
    snprintf(map, sizeof map, "0 %u 1\n", uid);            put(path, map);
    snprintf(path, sizeof path, "/proc/%d/gid_map", pid);
    snprintf(map, sizeof map, "0 %u 1\n", gid);            put(path, map);

    if (write(c.sync[1], "x", 1) != 1) die("sync-write");  // release the child
    close(c.sync[0]); close(c.sync[1]);

    int st;
    if (waitpid(pid, &st, 0) < 0) die("waitpid");
    return WIFEXITED(st) ? WEXITSTATUS(st) : 128 + WTERMSIG(st);
}
```