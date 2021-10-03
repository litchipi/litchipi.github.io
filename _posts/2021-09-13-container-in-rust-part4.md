---
layout: post
title:  "Birth of a child process"
date:   2021-09-12 17:34:00 +0200
modified_date: 2021-09-14 12:00:00 +0200
categories: rust
tags: rust tutorial learning container docker
series: Writing a container in Rust
serie_index: 4
serie_url: /series/container_in_rust
---

In this post, we're going to clone the parent process of our container into a child process.

Before that can happend, we'll set up the ground by preparing some inter-process communication
channels allowing us to interact with the child process we're going to create.

# Inter-process communication (IPC) with sockets
## Introduction to IPC
When it comes to inter-process communication (or IPC for short), the **Unix domain sockets** or
"same host socket communication" is the solution.
They differ from the "network socket communication" kind of sockets that are used to perform
network operations with remote hosts.

You can find a really nice article about sockets and IPC [here][ipctuto]
It consists in practice of a file (*Unix philosophy: everything is a file*)
In which we're going to read or write, to transfer information from one process to another.

For our tool, we don't need any fancy IPC, but we want to be able to transfer simple boolean
to / from our child process.

## Create a socketpair
Creating a pair of sockets will allow us to give one to the child process, and one to the
parent process.

![a pair of sockets](/assets/images/container_in_rust_part3/sockets.png)

This way we'll be able to transfer raw binary data from one process to the other the same way
we write binary data into a file stored in a filesystem.

Let's create a new file `src/cli.rs` containing everything related to IPC:
``` rust
use crate::errors::Errcode;

use std::os::unix::io::RawFd;
use nix::sys::socket::{socketpair, AddressFamily, SockType, SockFlag, send, MsgFlags, recv};

pub fn generate_socketpair() -> Result<(RawFd, RawFd), Errcode> {
	match socketpair(
		AddressFamily::Unix,
		SockType::SeqPacket,
		None,
		SockFlag::SOCK_CLOEXEC)
		{
			Ok(res) => Ok(res),
			Err(_) => Err(Errcode::SocketError(0))
	}
}
```

We create a `generate_socketpair` function in which we call the `socketpair` function, which is
the [standard Unix way of creating socket pairs][man-socketpair], but called from Rust.

- `AddressFamily::Unix`: We are creating a Unix domain socket
(see [all AddressFamily variants for details][AddressFamily-docs])

- `SockType::SeqPacket`: The socket will use a communication semantic with packets and fixed length datagrams.
(see [all SockType variants for details][SockType-docs])

- `None`: The socket will use the default protocol associated with the socket type.

- `SockFlag::SOCK_CLOEXEC`: The socket will be automatically closed after any syscall of the 
`exec` family. (see [Linux manual for `exec` syscalls][man-exec])

As we use a new `Errcode::SocketError` variant, let's add it to `src/errors.rs` now:
``` rust
 pub enum Errcode{
	// ...
	SocketError(u8),
 }
```

## Adding the sockets to the container config

When creating the configuration, let's generate our socketpair and add it to the `ContainerOpts`
data so the child process can access to it easily.
In the `src/config.rs` file:
``` rust
use crate::ipc::generate_socketpair;

// ...
use std::os::unix::io::RawFd;
#[derive(Clone)]
pub struct ContainerOpts{
	// ...
	pub fd:         RawFd,
	// ...
}
```

Also let's modify the `ContainerOpts::new` function so it returns the sockets along with the
constructed `ContainerOpts` struct, the parent container needs to get access to it.
``` rust
impl ContainerOpts{
	pub fn new(command: String, uid: u32, mount_dir: PathBuf) -> Result<(ContainerOpts, (RawFd, RawFd)), Errcode> {
		let sockets = generate_socketpair()?;
		// ...
		Ok((
			ContainerOpts {
				// ...
				fd: sockets.1.clone(),
			},
			sockets
		))
	}
 }
```

## Adding to the container, setting up the cleaning
In our container implementation, let's add a field in the `Container` struct to
be able to access the sockets more easily.
In the file `src/container.rs`:

``` rust
use nix::unistd::close;
use std::os::unix::io::RawFd;
// ...

pub struct Container{
	sockets: (RawFd, RawFd),
	config: ContainerOpts,
 }
 
impl Container {
	pub fn new(args: Args) -> Result<Container, Errcode> {
		let (config, sockets) = ContainerOpts::new(
			// ...
			)?;
		Ok(Container {
			sockets,
			config,
		})
	}
}
```

As sockets  requires some cleaning before exit, let's close them in the `clean_exit` function.
``` rust
pub fn clean_exit(&mut self) -> Result<(), Errcode>{
	// ...
	if let Err(e) = close(self.sockets.0){
		log::error!("Unable to close write socket: {:?}", e);
		return Err(Errcode::SocketError(3));
	}

	if let Err(e) = close(self.sockets.1){
		log::error!("Unable to close read socket: {:?}", e);
		return Err(Errcode::SocketError(4));
	}
	// ...
}
```

## Creating wrappers for IPC

To ease the use of the sockets, let's create two wrappers to ease the use.
We only want to transfer boolean, so let's create a `send_boolean` and `recv_boolean` function
in `src/ipc.rs`:
``` rust
pub fn send_boolean(fd: RawFd, boolean: bool) -> Result<(), Errcode> {
	let data: [u8; 1] = [boolean.into()];
	if let Err(e) = send(fd, &data, MsgFlags::empty()) {
		log::error!("Cannot send boolean through socket: {:?}", e);
		return Err(Errcode::SocketError(1));
	};
	Ok(())
}

pub fn recv_boolean(fd: RawFd) -> Result<bool, Errcode> {
	let mut data: [u8; 1] = [0];
	if let Err(e) = recv(fd, &mut data, MsgFlags::empty()) {
		log::error!("Cannot receive boolean from socket: {:?}", e);
		return Err(Errcode::SocketError(2));
	}
	Ok(data[0] == 1)
}
```
Here it's just some interacting with the `send` and `recv` functions from the `nix` crate, handling
data types conversion, etc... There's nothing much to say about it, but it's still interesting
how we can interact with functions that has a low-level C backend from Rust.

We won't use the wrappers for now, but they'll come handy later.

### Patch for this step

The code for this step is available on github [litchipi/crabcan branch “step7”][code-step7]. \\
The raw patch to apply on the previous step can be found [here][patch-step7]



# Cloning a process

[code-step7]: https://github.com/litchipi/crabcan/tree/step7/
[patch-step7]: https://github.com/litchipi/crabcan/commit/step7.diff

[ipctuto]: https://opensource.com/article/19/4/interprocess-communication-linux-networking
[man-socketpair]: https://man7.org/linux/man-pages/man2/socketpair.2.html
[man-exec]: https://man7.org/linux/man-pages/man3/exec.3.html

[AddressFamily-docs]: https://docs.rs/nix/0.22.1/nix/sys/socket/enum.AddressFamily.html
[SockType-docs]: https://docs.rs/nix/0.22.1/nix/sys/socket/enum.SockType.html
