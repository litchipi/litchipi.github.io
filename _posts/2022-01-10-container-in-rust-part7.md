---
layout: post
title:  "Syscalls and ressources restriction"
date:   2022-01-10 07:48:30 +0200
categories: rust
tags: rust tutorial learning container docker
series: Writing a container in Rust
serie_index: 7
serie_url: /series/container_in_rust
---

# Syscalls restriction
## What are syscalls ?
## Seccomp and syscalls restriction

As given by the [wikipedia page of seccomp][wikipedia_seccomp]:

> **seccomp** (short for *secure computing mode*) is a computer security facility in the Linux kernel.

> **seccomp** allows a process to make a one-way transition into a "secure" state where it *cannot make any
system calls* except `exit()`, `sigreturn()`, `read()` and `write()` to already-open file descriptors.
Should it attempt any other system calls, the kernel will terminate the process with **SIGKILL** or **SIGSYS**.
In this sense, it does **not virtualize** the system's resources but **isolates** the process from them entirely.

> seccomp mode is enabled via the prctl(2) system call using the PR_SET_SECCOMP argument, or
(since Linux kernel 3.17) via the seccomp(2) system call.

It's kinda easy to see how **seccomp** is one of the backbone for a container such as Docker.

To effectively use this secure computing mode, you need to first create a seccomp **profile**
(also called *context*) which will be applied later on on the child process. \\
This profile describes **TODO**

## What syscalls to restrict ?

In this tutorial we won't look closely at each syscall we will refuse / allow as deep
descriptions with some examples of exploits are given in [the "syscalls" section of the original
tutorial][origtuto_syscalls].

Also, as pointed out in the original tutorial, 
good sources for syscalls restrictions are
[the docker documentation page](https://github.com/docker/docker.github.io/blob/master/engine/security/seccomp.md)
and [the seccomp profile of moby](https://github.com/moby/moby/blob/b248de7e332b6e67b08a8981f68060e6ae629ccf/profiles/seccomp/default.json).

The syscalls we will **refuse** in our container are:
```
keyctl
add_key
request_key
mbind
migrate_pages
move_pages
set_mempolicy
userfaultfd
perf_event_open
```

### Patch for this step

The code for this step is available on github [litchipi/crabcan branch “step13”][code-step13]. \\
The raw patch to apply on the previous step can be found [here][patch-step13]


# Ressources restriction
### Patch for this step

The code for this step is available on github [litchipi/crabcan branch “step14”][code-step14]. \\
The raw patch to apply on the previous step can be found [here][patch-step14]


[code-step13]: https://github.com/litchipi/crabcan/tree/step13/
[patch-step13]: https://github.com/litchipi/crabcan/commit/step13.diff

[code-step14]: https://github.com/litchipi/crabcan/tree/step14/
[patch-step14]: https://github.com/litchipi/crabcan/commit/step14.diff

[origtuto_syscalls]: https://blog.lizzie.io/linux-containers-in-500-loc.html#org8504d16 
