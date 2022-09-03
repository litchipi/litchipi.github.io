---
layout: post
title:  "User namespaces and Linux Capabilties"
date:   2022-01-06 09:40:00 +0200
modified_date: 2022-08-23 9:01:22 +0200
categories: rust_container_tutorial
tags: rust tutorial learning container docker
series: Writing a container in Rust
serie_index: 6
serie_url: /series/container_in_rust
---

# User namespaces

User namespaces allows process to virtually be *root* inside their own namespace,
but not on the parent system.
It also allows the child namespace to have its own users / groups configuration.

The development of user namespaces started on Linux *2.6.23*, got released in Linux *3.8*, and still
continues to create a debate within the Linux kernel community about its efficiency, the
security issues it raises, etc...

> See [this great article][linux-user-namespaces-not-secure]
(link stolen from [original tutorial footnotes][original-tutorial])

The general user namespace configuration is the following:
- Child process tries to `unshare` its user resources
    - If it suceeds, then user namespace isolation is supported
    - If it fails, then user namespace isolation isn't supported
- Child process tells parent process if it supports user namespace isolation
- If it supported user namespace isolation, parent process maps UID / GID of the user namespace
- Parent process tells child process to continue
- Then child process switches his UID / GID to the one provided by the user as a parameter.

The whole code will be written in the `src/namespaces.rs` file, so we can add it as a module
in `src/main.rs`:

``` rust
// ...
mod namespaces;
```

And add a specific error for namespaces configuration in `src/errors.rs`:

``` rust
pub enum Errcode {
    // ...
    NamespacesError(u8),
}
```

## User namespaces supported ?

In our file `src/namespaces.rs`, we create two functions, one `userns` that will be executed by
the child process during its configuration, one `handle_child_uid_map` that will be called by the
container when it will perform UID / GID mapping.

``` rust
use std::os::unix::io::RawFd;
use nix::sched::{unshare, CloneFlags};

use crate::errors::Errcode;
use crate::ipc::{send_boolean, recv_boolean};

pub fn userns(fd: RawFd, uid: u32) -> Result<(), Errcode> {
    log::debug!("Setting up user namespace with UID {}", uid);
    let has_userns = match unshare(CloneFlags::CLONE_NEWUSER) {
        Ok(_) => true,
        Err(_) => false,
    };
    send_boolean(fd, has_userns)?;

    if recv_boolean(fd)? {
        return Err(Errcode::NamespacesError(0));
    }

    if has_userns {
        log::info!("User namespaces set up");
    } else {
        log::info!("User namespaces not supported, continuing...");
    }

    // Switch UID / GID with the one provided by the user

    Ok(())
}


use nix::unistd::Pid;
pub fn handle_child_uid_map(pid: Pid, fd: RawFd) -> Result<(), Errcode> {
    if recv_boolean(fd)? {
        // Perform UID / GID map here
    } else {
        log::info!("No user namespace set up from child process");
    }

    log::debug!("Child UID/GID map done, sending signal to child to continue...");
    send_boolean(fd, false)
}

```

The child container calls the **unshare** function
with the `CLONE_NEWUSER` flag, which perform the following:
> Unshare the user namespace, so that the calling process is moved into a new user
namespace which is not shared with any previously existing process.

> See [linux manual](https://man7.org/linux/man-pages/man2/unshare.2.html) for more details

If that call is successful, then user namespaces are supported. We use our previously defined
`send_boolean` and `recv_boolean` to transfer this information from the child process to the
parent process (which will be waiting for it in the `handle_child_uid_map` right after creating
the child process).

After the parent process performs (or not) the UID / GID map, it sends back a signal of success
and the child process can carry on, switching its UID and GID to the one the user provided.

## Inserting into the configuration flow

As the `userns` is part of the child process configuration, we add it to the
`setup_container_configurations` function in `src/child.rs`:

``` rust
use crate::namespaces::userns;

fn setup_container_configurations(config: &ContainerOpts) -> Result<(), Errcode> {
    // ...
    userns(config.fd, config.uid)?;
    // ...
}
```

> **Note** The order of configuration is really important. As we allow / restrict operations the
process can perform, we have to restrict the powers we know we won't use.

Right after we create the child process, the container must call the `handle_child_uid_map`
and wait for its signal to perform the operation.

In `src/container.rs`:

```rust
use crate::namespaces::handle_child_uid_map;

impl Container {
    // ...
    pub fn create(&mut self) -> Result<(), Errcode> {
       let pid = generate_child_process(self.config.clone())?;
       handle_child_uid_map(pid, self.sockets.0)?;
       // ...
    }
```

Finally, we want to close the socket once it's no longer used, so let's add in `src/child.rs`:

```rust
fn child(config: ContainerOpts) -> isize {
    match setup_container_configurations(&config) {
        // ...
    }

    if let Err(_) = close(config.fd){
        log::error!("Error while closing socket ...");
        return -1;
    }

    // ...
}
```

## Mapping the UID / GID

The file `/proc/<pid>/uidmap` is used by the Linux kernel to map the user IDs inside and outside
the namespace of a process. \\
The format is the following:
```
ID-inside-ns ID-outside-ns length
```
So if the file `/proc/<pid>/uidmap` contains `0 1000 5`, a user having the UID `0` *inside* the container
will have a UID `1000` *outside* the container. \\
In the same way, a UID of `1` *inside* maps to a UID of `1001` outside,
but a UID of `6` *inside* doesn't map to `1006` *outside* as only `5` UID are
allowed to be mapped (the *length*).

> Explanation shamelessly inspired from [this article][effect-writing-uidmap]

It happens exactly the same for GIDs as we only set up the group of the user,
using exactly the same number.

> For more informations about UID, GID and their relationship, look at [this article][uidvsgid]

As our child process has his root directory changed, we want our parent process
to take care of writing to the correct files. \\
Let's fill up the blanks in our `handle_child_uid_map` function:

``` rust
use std::fs::File;
use std::io::Write;
use nix::unistd::Pid;

const USERNS_OFFSET: u64 = 10000;
const USERNS_COUNT: u64 = 2000;

pub fn handle_child_uid_map(pid: Pid, fd: RawFd) -> Result<(), Errcode> {
    if recv_boolean(fd)? {
        if let Ok(mut uid_map) = File::create(format!("/proc/{}/{}", pid.as_raw(), "uid_map")) {
            if let Err(_) = uid_map.write_all(format!("0 {} {}", USERNS_OFFSET, USERNS_COUNT).as_bytes()) {
                return Err(Errcode::NamespacesError(4));
            }
        } else {
            return Err(Errcode::NamespacesError(5));
        }

        if let Ok(mut gid_map) = File::create(format!("/proc/{}/{}", pid.as_raw(), "gid_map")) {
            if let Err(_) = gid_map.write_all(format!("0 {} {}", USERNS_OFFSET, USERNS_COUNT).as_bytes()) {
                return Err(Errcode::NamespacesError(6));
            }
        } else {
            return Err(Errcode::NamespacesError(7));
        }
    } else {
        // ...
    }
    // ...
}
```

Note that we are mapping the UIDs and GIDs to more than 10000 so we are sure we don't
have any collision with an existing ident. \\
We map up to 2000 UIDs but that is not really important.

If we resume, from now on, whenever our contained process (matched by its **PID**)
claim he has (or sets himself) the UID `0`, the kernel will see it with the UID `10000`. \\
The same will happend for the GIDs.
> If the asked UID for the contained process is `0`, the contained process will
set itself as **root** in the scope of its isolated execution environment

Let's now tell our contained process to switch its UID to the one the user asked
for in the parameters in the `userns` function:

``` rust
use crate::errors::Errcode;

use nix::unistd::{Gid, Uid};
use nix::unistd::{setgroups, setresuid, setresgid};
use std::os::unix::io::RawFd;

pub fn userns(fd: RawFd, uid: u32) -> Result<(), Errcode> {
    // ...
    log::debug!("Switching to uid {} / gid {}...", uid, uid);
    let gid = Gid::from_raw(uid);
    let uid = Uid::from_raw(uid);

    if let Err(_) = setgroups(&[gid]){
        return Err(Errcode::NamespacesError(1));
    }

    if let Err(_) = setresgid(gid, gid, gid){
        return Err(Errcode::NamespacesError(2));
    }

    if let Err(_) = setresuid(uid, uid, uid){
        return Err(Errcode::NamespacesError(3));
    }

    Ok(())
}
```

We use the `setgroups` function (see [its linux manual page][man-setgroups]) to
set the list of groups the process is part of, for this simple example, let's
just add the GID of the process.

We use the `setresuid` and `setresgid` to set the UID and GID (respectively) of
the process.
This will set the *real user ID*, the *effective user ID*, and the *saved set-user-ID*.

The *real user ID* is **who you are** (who you logged into), \\
the *effective user ID* is **who you claim you are**
(used for temporary priviledges with *sudo*, or impersonate a user with *su*), \\
and the *saved set-user-ID* is **who you were before**
(in case of chained impersonations ...)

> For a detailed explanation, look at [this StackOverflow answer](https://stackoverflow.com/a/32456814)

So now, the contained process can be root in its isolated environment,
mapped by the system to a real UID >10000, and it can manage its users and
groups without polluting the parent system.

## About user namespaces security

On the [original tutorial footnotes][original-tutorial] (seriously, check it out.)
you can find out more informations, experimentations and documentation about
user namespaces.

One document I found really helpful when talking about hardening the container
security and isolation is [this PDF][hardening-linux-container-pdf], which
covers a lot of aspects that are out of the scope of this tutorial. \\
(User namespaces security are covered on section **5.5** on page **39**)

## Testing

When testing this out, this is the output we can get:

```
[2022-01-05T07:42:54Z INFO  crabcan] Args { debug: true, command: "/bin/bash", uid: 0, mount_dir: "./mountdir/" }
[2022-01-05T07:42:54Z DEBUG crabcan::container] Linux release: 5.13.0-22-generic
[2022-01-05T07:42:54Z DEBUG crabcan::container] Container sockets: (3, 4)
[2022-01-05T07:42:54Z DEBUG crabcan::hostname] Container hostname is now weird-moon-93
[2022-01-05T07:42:54Z DEBUG crabcan::mounts] Setting mount points ...
[2022-01-05T07:42:54Z DEBUG crabcan::mounts] Mounting temp directory /tmp/crabcan.SmpKbLNhLuJo
[2022-01-05T07:42:54Z DEBUG crabcan::mounts] Pivoting root
[2022-01-05T07:42:54Z DEBUG crabcan::mounts] Unmounting old root
[2022-01-05T07:42:54Z DEBUG crabcan::namespaces] Setting up user namespace with UID 0
[2022-01-05T07:42:54Z DEBUG crabcan::namespaces] Child UID/GID map done, sending signal to child to continue...
[2022-01-05T07:42:54Z DEBUG crabcan::container] Creation finished
[2022-01-05T07:42:54Z INFO  crabcan::namespaces] User namespaces set up
[2022-01-05T07:42:54Z DEBUG crabcan::namespaces] Switching to uid 0 / gid 0...
[2022-01-05T07:42:54Z DEBUG crabcan::container] Container child PID: Some(Pid(40182))
[2022-01-05T07:42:54Z DEBUG crabcan::container] Waiting for child (pid 40182) to finish
[2022-01-05T07:42:54Z INFO  crabcan::child] Container set up successfully
[2022-01-05T07:42:54Z INFO  crabcan::child] Starting container with command /bin/bash and args ["/bin/bash"]
[2022-01-05T07:42:54Z DEBUG crabcan::container] Finished, cleaning & exit
[2022-01-05T07:42:54Z DEBUG crabcan::container] Cleaning container
[2022-01-05T07:42:54Z DEBUG crabcan::errors] Exit without any error, returning 0
```

Everything seams to work as intended, we got our UID `0`, and it works with
different UIDs, so all good.

### Patch for this step

The code for this step is available on github [litchipi/crabcan branch “step11”][code-step11]. \\
The raw patch to apply on the previous step can be found [here][patch-step11]

# Linux Capabilties


## What are Linux capabilties

### Dividing the admin powers

Linux capabilities are a way to divide the almighty `administrator` role with
supreme powers over the machine into a set of limited powers controlling isolated
systems and processes of the system. \\
Those "isolated powers" include:
- Modifying owner / permission of a file (`CAP_CHOWN`)
- Using / setting the system time (`CAP_SYS_TIME`)
- Extensive use of the system ressources (`CAP_SYS_RESOURCE`)
- Perform raw I/O port operations (`CAP_SYS_RAWIO`)
- Lower a process priority (`CAP_SYS_NICE`)
- Reboot the system (`CAP_SYS_BOOT`)
- Modify the capabilities of another process (`CAP_SETPCAP`)

If we don't restrict the power of the contained process, it could change all
the settings we just set during its configuration.

> For more informations, details, explanations, and a list of all capabilities,
look at [the linux manual][man-capabilities]

### Acquiring capabilities

The Linux kernel derives a process capabilities from 4 categories of
capabilities:
- **Ambient**: Capabilities that are *granted* to a process, and *can be inherited*
by any child process created. \\
In short, **Ambient** capabilities are capabilities **both Permitted and Inheritable**.

- **Permitted**: Capabilities *granted* to a process. \\
Any capabilities dropped from the permitted set **can never be reacquired**.

- **Effective**: The actual **applied** permissions of a thread in the given context.

- **Inheritable**: The set of capabilities preserved across an `execve`. \\
When starting a new process from a file, adds capabilities from the **Inheritable**
set of the parent process to the **Permitted** set of the child process *only
if these capabilities are not restricted by the file permissions*

- **Bounding**: Used to define the capabilities to drop when performing an `execve`.

To know what kind of capabilities a newly created process have,
the Linux kernel uses these equations:

```
P'(ambient)     = (file is privileged) ? 0 : P(ambient)

P'(permitted)   = (P(inheritable) & F(inheritable)) | (F(permitted) & cap_bset) | P'(ambient)

P'(effective)   = F(effective) ? P'(permitted) : P'(ambient)

P'(inheritable) = P(inheritable) [i.e., unchanged]
```

where:
- `P` denotes the value of a thread capability set before the `execve`
- `P'` denotes the value of a thread capability set after the `execve`
- `F` denotes a file capability set
- `cap_bset` is the value of the capability **Bounding** set.

### Further readings

The [linux manual page for capabilities][man-capabilities] have a very
detailed explanation about the whole thing and is the best reference when searching
for informations on the subject.

The great **LWN** website got a list of all articles regarding capabilities [here](https://lwn.net/Kernel/Index/#Capabilities)

## Choosing what to restrict

To handle the capabilities, we will use the crate [capctl](https://crates.io/crates/capctl).
Let's add it to `Cargo.toml`:

``` toml
[dependencies]
# ...
capctl = "0.2.0"
```

We can create a file `src/capabilities.rs` in which we will handle everything
related to them.

Let's first define what we don't want. \\
For our contained process, this is the list of the capabilities we will restrict:

``` rust
use capctl::caps::Cap;
const CAPABILITIES_DROP: [Cap; 21] = [
    Cap::AUDIT_CONTROL,
    Cap::AUDIT_READ,
    Cap::AUDIT_WRITE,
    Cap::BLOCK_SUSPEND,
    Cap::DAC_READ_SEARCH,
    Cap::DAC_OVERRIDE,
    Cap::FSETID,
    Cap::IPC_LOCK,
    Cap::MAC_ADMIN,
    Cap::MAC_OVERRIDE,
    Cap::MKNOD,
    Cap::SETFCAP,
    Cap::SYSLOG,
    Cap::SYS_ADMIN,
    Cap::SYS_BOOT,
    Cap::SYS_MODULE,
    Cap::SYS_NICE,
    Cap::SYS_RAWIO,
    Cap::SYS_RESOURCE,
    Cap::SYS_TIME,
    Cap::WAKE_ALARM
];
```

> A detailed explanation of why these capabilities should be dropped
is available in [this section of the original tutorial](https://blog.lizzie.io/linux-containers-in-500-loc.html#org07e738c)

It may seams like we dropped everything, but we still retained some of them. \\
These capabilities still let some power to the contained process as long as it is
contained inside its namespace, even if they could be real threats to a
system (example with [CAP_DAC_OVERRIDE][cap_dac_override-security]
or [CAP_FOWNER][cap_fowner-security])

> A detailed list of the retained capabilities and an explanation of why these
capabilities can be kept is available in
[this section of the original tutorial](https://blog.lizzie.io/linux-containers-in-500-loc.html#orgc6d2b81)

## Restricting the process capabilities

First, create a new error variant in `src/errors.rs`:

``` rust
pub enum Errcode {
    // ...
    CapabilitiesError(u8),
}
```

Then, let's create a function `setcapabilities` who will perform the
restriction. \\
In `src/capabilities.rs`:

``` rust
use crate::errors::Errcode;
use capctl::caps::FullCapState;

pub fn setcapabilities() -> Result<(), Errcode> {
    log::debug!("Clearing unwanted capabilities ...");
    if let Ok(mut caps) = FullCapState::get_current() {
        caps.bounding.drop_all(CAPABILITIES_DROP.iter().map(|&cap| cap));
        caps.inheritable.drop_all(CAPABILITIES_DROP.iter().map(|&cap| cap));
        Ok(())
    } else {
        Err(Errcode::CapabilitiesError(0))
    }
}
```

We just need to add this capabilities restriction at the tail of our
configuration function for the contained process. \\
In `src/child.rs`:

``` rust
use crate::capabilities::setcapabilities;

pub fn setup_container_configurations(config: &ContainerOpts) -> Result<(), Errcode> {
    // ...
    setcapabilities()?;
    Ok(())
}
```

Finally, add the new module in `src/main.rs`:

``` rust
// ...
mod capabilities;
```

## Testing

When testing this out, this is the output we can get:

```
[2022-01-06T07:16:31Z INFO  crabcan] Args { debug: true, command: "/bin/bash", uid: 0, mount_dir: "./mountdir/" }
[2022-01-06T07:16:31Z DEBUG crabcan::container] Linux release: 5.13.0-22-generic
[2022-01-06T07:16:31Z DEBUG crabcan::container] Container sockets: (3, 4)
[2022-01-06T07:16:31Z DEBUG crabcan::hostname] Container hostname is now silent-book-0
[2022-01-06T07:16:31Z DEBUG crabcan::mounts] Setting mount points ...
[2022-01-06T07:16:31Z DEBUG crabcan::mounts] Mounting temp directory /tmp/crabcan.i1L9Dl0nU5ef
[2022-01-06T07:16:31Z DEBUG crabcan::mounts] Pivoting root
[2022-01-06T07:16:31Z DEBUG crabcan::mounts] Unmounting old root
[2022-01-06T07:16:31Z DEBUG crabcan::namespaces] Setting up user namespace with UID 0
[2022-01-06T07:16:31Z DEBUG crabcan::namespaces] Child UID/GID map done, sending signal to child to continue...
[2022-01-06T07:16:31Z DEBUG crabcan::container] Creation finished
[2022-01-06T07:16:31Z DEBUG crabcan::container] Container child PID: Some(Pid(2276734))
[2022-01-06T07:16:31Z DEBUG crabcan::container] Waiting for child (pid 2276734) to finish
[2022-01-06T07:16:31Z INFO  crabcan::namespaces] User namespaces set up
[2022-01-06T07:16:31Z DEBUG crabcan::namespaces] Switching to uid 0 / gid 0...
[2022-01-06T07:16:31Z DEBUG crabcan::capabilities] Clearing unwanted capabilities ...
[2022-01-06T07:16:31Z INFO  crabcan::child] Container set up successfully
[2022-01-06T07:16:31Z INFO  crabcan::child] Starting container with command /bin/bash and args ["/bin/bash"]
[2022-01-06T07:16:31Z DEBUG crabcan::container] Finished, cleaning & exit
[2022-01-06T07:16:31Z DEBUG crabcan::container] Cleaning container
[2022-01-06T07:16:31Z DEBUG crabcan::errors] Exit without any error, returning 0
```

Not very clear that it works, but it didn't fail and a bunch of tests later will
be able to ensure it works as intended.

### Patch for this step

The code for this step is available on github [litchipi/crabcan branch “step12”][code-step12]. \\
The raw patch to apply on the previous step can be found [here][patch-step12]


[linux-user-namespaces-not-secure]: https://medium.com/@ewindisch/linux-user-namespaces-might-not-be-secure-enough-a-k-a-subverting-posix-capabilities-f1c4ae19cad
[hardening-linux-container-pdf]: https://www.nccgroup.com/globalassets/our-research/us/whitepapers/2016/april/ncc_group_understanding_hardening_linux_containers-1-1.pdf
[original-tutorial]: https://blog.lizzie.io/linux-containers-in-500-loc.html
[effect-writing-uidmap]: https://ops.tips/notes/effect-of-writing-to-proc-pid-uid-map/
[uidvsgid]: https://www.cbtnuggets.com/blog/technology/system-admin/linux-file-permission-uid-vs-gid

[code-step11]: https://github.com/litchipi/crabcan/tree/step11/
[patch-step11]: https://github.com/litchipi/crabcan/commit/step11.diff

[code-step12]: https://github.com/litchipi/crabcan/tree/step12/
[patch-step12]: https://github.com/litchipi/crabcan/commit/step12.diff

[man-setgroups]: https://man7.org/linux/man-pages/man2/getgroups.2.html
[man-capabilities]: https://man7.org/linux/man-pages/man7/capabilities.7.html

[cap_dac_override-security]: https://book.hacktricks.xyz/linux-unix/privilege-escalation/linux-capabilities#cap_dac_override
[cap_fowner-security]: https://book.hacktricks.xyz/linux-unix/privilege-escalation/linux-capabilities#cap_fowner
