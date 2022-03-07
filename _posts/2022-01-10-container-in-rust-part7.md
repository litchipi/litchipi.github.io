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

When an application crashes because of a bug, the problem *has to be contained* in
such a way that the underlying operating system is not affected.

Imagine if your Tetris game is crashing while saving your highscore. Because
it didn't finish its operations on the disk, *it can corrupt it*, or leave it in
such a state that would cause troubles for any other application to use it afterward.

> One funny example of that principle was demonstrated live with the very famous
[Windows 98 live fail][windows_98_fail], which is also why nowadays live demonstrations
are replaced with pre-recorded demos

To avoid this case to happend, Linux separates software into 2 "lands",
**kernel**-land and **user**-land.

The kernel-land is *priviliged*, meaning it owns every bit of the machine (to some
exceptions like the processor-level context separation of [Arm TrustZone][arm_trust_zone]),
it can read and write into all the memory and uses drivers to interact with the
connected peripherals.

The user-land is *un-priviliged*, meaning that even an almighty *root* account
do not have full-control on the machine directly, but this level of power
allows to perform any kind of syscalls.

*Syscalls* are the way a user-land application can perform an action that is normally
done by the kernel.

> Syscalls are a *special assembly* machine code, each syscall has an "index"
> that is passed to a register, when executing the `syscall` command, it will
> switch into the kernel code responsible for handling such a syscall.

<img src="/assets/images/container_in_rust_part7/syscalls.png" alt="syscalls representation"
width="350">

Here is a representation of an application writing to the disk using the `write`
syscall.

> When using `write` in another langage like Python, it usually calls
> the `write` function of C as this function is directly translated into the
> corresponding syscall when compiled.

The driver is embedded in the kernel as a module, and it manages its internal
state and operations internally.
This way if anything goes wrong and another application wants to write,
it can reset its state, take care of the special operations that the underlying
physical device requires (I'm looking at you, [eMMC][emmc_state_graph] !)

> More on the linux kernel drivers [in this article][linux_kernel_driver_article]
(**BEWARE OF THE ARTICLE COLORS**, protect your eyes, seriously).

A complete list of all the syscalls that can be called with a Linux kernel is
available [here][linux_syscalls_list]

## Seccomp and syscalls restriction

As syscalls allows a user to control the system, it is necessary to restrict
any syscall that may allow a procssed inside our container to harm our underlying
operating system.

Enters **Seccomp** \\
As given by the [wikipedia page of seccomp][wikipedia_seccomp]:

> **seccomp** (short for *secure computing mode*) is a computer security facility
in the Linux kernel. \\
> **seccomp** allows a process to make a one-way transition into a "secure" state
where it *cannot make any system calls* except `exit()`, `sigreturn()`, `read()`
and `write()` to already-open file descriptors.
Should it attempt any other system calls, the kernel will terminate the process
with **SIGKILL** or **SIGSYS**. In this sense, it does **not virtualize** the system's
resources but **isolates** the process from them entirely.

> seccomp mode is enabled via the prctl(2) system call using the PR_SET_SECCOMP
argument, or (since Linux kernel 3.17) via the seccomp(2) system call.

It's kinda easy to see how **seccomp** is one of the backbone for a container
such as Docker.

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

[cgroups vulnerability][cgroups-vuln]

### Patch for this step

The code for this step is available on github [litchipi/crabcan branch “step14”][code-step14]. \\
The raw patch to apply on the previous step can be found [here][patch-step14]


[code-step13]: https://github.com/litchipi/crabcan/tree/step13/
[patch-step13]: https://github.com/litchipi/crabcan/commit/step13.diff

[code-step14]: https://github.com/litchipi/crabcan/tree/step14/
[patch-step14]: https://github.com/litchipi/crabcan/commit/step14.diff

[origtuto_syscalls]: https://blog.lizzie.io/linux-containers-in-500-loc.html#org8504d16 
[cgroups-vuln]: https://thehackernews.com/2022/03/new-linux-kernel-cgroups-vulnerability.html
[wikipedia_seccomp]: https://en.wikipedia.org/wiki/Seccomp
[windows_98_fail]: https://youtu.be/yeUyxjLhAxU
[arm_trust_zone]: https://blog.quarkslab.com/introduction-to-trusted-execution-environment-arms-trustzone.html
[syscalls_diagram]: ../_diagrams/containers_in_rust_part7/syscalls.png
[linux_kernel_driver_article]: http://www.haifux.org/lectures/86-sil/kernel-modules-drivers/kernel-modules-drivers.html
[emmc_state_graph]: https://gist.github.com/StoneCypher/be7f117881915e7df7bbc96c5c0a84d5
[linux_syscalls_list]: https://linuxhint.com/list_of_linux_syscalls
