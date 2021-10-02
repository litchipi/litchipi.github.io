---
layout: post
title:  "Creating a child process"
date:   2021-09-12 17:34:00 +0200
modified_date: 2021-09-14 12:00:00 +0200
categories: rust
tags: rust tutorial learning container docker
series: Writing a container in Rust
serie_index: 3
serie_url: /series/container_in_rust
---

# Basis of the container
Let's create the skeletton of the actual container. From the args we got from the commandline,
we're going to extract the configuration and initialize a `Container` struct that will have to
perform the container work.

## The configuration
In a new file `src/config.rs`, let's define the [struct][rustbook-struct] for the container
configuration.
``` rust
use crate::errors::Errcode;

use std::ffi::CString;
use std::path::PathBuf;

#[derive(Clone)]
pub struct ContainerOpts{
	pub path:       CString,
	pub argv:       Vec<CString>,

	pub uid:        u32,
	pub mount_dir:  PathBuf,
}
```
Let's analyse what we got there:
- **path**: The path of the binary / executable / script to execute inside the container
- **argv**: The full arguments passed (including the `path` option) into the commandline.
> These are required to perform an [execve syscall][syscall-execve] which we will use
> to contain our software in a process whose execution context is restricted.
- **uid**: The ID of the user inside the container. An ID of `0` means it's root (administrator).
> The user ID is visible on GNU/Linux by looking the file `/etc/passwd`, who has a format
> `username:x:uuid:guid:comment:homedir:shell`
- **mount_dir**: The path of the directory we want to use as a `/` root inside our container. \\
Configurations will be added later as we need them.

We are using CString because it'll be much easier to use to call the `execve` syscall later.
Also as the configuration will be shared with the child process to be created, we need to be able
to `clone` the struct (which contains data stored on the [heap][so-stackheap]),
that's why we add a `derive(Clone)` attribute to the struct. \\
(See [the chapter about ownership & data copy in the Book][rustbook-cloneandcopy])

Let's create the constructor of our configuration struct
``` rust
impl ContainerOpts{
	pub fn new(command: String, uid: u32, mount_dir: PathBuf) -> Result<ContainerOpts, Errcode> {
		let argv: Vec<CString> = command.split_ascii_whitespace()
			.map(|s| CString::new(s).expect("Cannot read arg")).collect();
		let path = argv[0].clone();
 
		Ok(
			ContainerOpts {
				path,
				argv,
				uid,
				mount_dir,
			}
		)
	}
}
```
Nothing too complicated here, we just get each arg from the command `String`, and creates a
`Vec<CString>` from them, cloning the first one, and creating the struct while returning an `Ok`
result.

## The container skeletton

Let's now create the Container `struct` that will perform the main tasks ahead.
``` rust
use crate::cli::Args;
use crate::errors::Errcode;
use crate::config::ContainerOpts;

pub struct Container{
	config: ContainerOpts,
}

impl Container {
	pub fn new(args: Args) -> Result<Container, Errcode> {
		let config = ContainerOpts::new(
			args.command,
			args.uid,
			args.mount_dir)?;
		Ok(Container {
			config,
			})
		}

	pub fn create(&mut self) -> Result<(), Errcode> {
		log::debug!("Creation finished");
		Ok(())
	}

	pub fn clean_exit(&mut self) -> Result<(), Errcode>{
		log::debug!("Cleaning container");
		Ok(())
	}
}
```
The `struct Container` is defined with a unique `config` field containing our configuration,
and implements 3 functions:
- `new` function that creates the `ContainerOpts` struct from the commandline arguments.
- `create` function that will handle the container creation process.
- `clean_exit` function that will be called before each exit to be sure we stay clean. \\
For now we let them very basic and will fill them later on.

Finally, we create a function `start` that will get the args from the commandline and handle
everything from the `struct Container` creation to the exit. \\
It returns a `Result` that will inform if an error happened during the process.

``` rust
pub fn start(args: Args) -> Result<(), Errcode> {
	let mut container = Container::new(args)?;
	if let Err(e) = container.create(){
		container.clean_exit()?;
		log::error!("Error while creating container: {:?}", e);
		return Err(e);
	}
	log::debug!("Finished, cleaning & exit");
	container.clean_exit()
}
```

The `?` you see at the end of the lines is used to propagate the errors. \\
`let mut container = Container::new(args)?;` is the equivalent of:
``` rust
let mut container = match Container::new(args) {
	Ok(el) => el,
	Err(e) => return Err(e),
}
```
However for this, the type in case of `Err` has to be the same. That's why having a unique
`Errcode` for all our errors in the project is handy, we can basically put `?` everywhere and
"cascade" any error back to the `start` function which will call `clean_exit` before logging the
error and exit the process with an error return code thanks to the `exit_with_retcode` function
we defined in the part about errors handling.

## Linking to the main function
One last thing, we need to call the `start` function from our `main` function. \\
In `src/main.rs`, add the following at the beginning of the file:
``` rust
mod container;
mod config;
```
And replace the `exit_with_retcode(Ok(()))` with `exit_with_retcode(container::start(args))`.

After testing, we get the following output:
```
[2021-10-02T14:16:41Z INFO  crabcan] Args { debug: true, command: "/bin/bash", uid: 0, mount_dir: "./mountdir/" }
[2021-10-02T14:16:41Z DEBUG crabcan::container] Creation finished
[2021-10-02T14:16:41Z DEBUG crabcan::container] Finished, cleaning & exit
[2021-10-02T14:16:41Z DEBUG crabcan::container] Cleaning container
[2021-10-02T14:16:41Z DEBUG crabcan::errors] Exit without any error, returning 0
```

### Patch for this step

The code for this step is available on github [litchipi/crabcan branch “step5”][code-step5].
Apply the following patch to the code got from previous step
<details>
	diff --git a/src/config.rs b/src/config.rs
	new file mode 100644
	index 0000000..4df3571
	--- /dev/null
	+++ b/src/config.rs
	@@ -0,0 +1,31 @@
	+use crate::errors::Errcode;
	+
	+use std::ffi::CString;
	+use std::path::PathBuf;
	+#[derive(Clone)]
	+pub struct ContainerOpts{
	+    pub path:       CString,
	+    pub argv:       Vec<CString>,
	+
	+    pub uid:        u32,
	+    pub mount_dir:  PathBuf,
	+}
	+
	+impl ContainerOpts{
	+    pub fn new(command: String, uid: u32, mount_dir: PathBuf)
	+            -> Result<ContainerOpts, Errcode> {
	+
	+        let argv: Vec<CString> = command.split_ascii_whitespace()
	+            .map(|s| CString::new(s).expect("Cannot read arg")).collect();
	+        let path = argv[0].clone();
	+
	+        Ok(
	+            ContainerOpts {
	+                path,
	+                argv,
	+                uid,
	+                mount_dir,
	+            },
	+        )
	+    }
	+}
	diff --git a/src/container.rs b/src/container.rs
	new file mode 100644
	index 0000000..a0c168f
	--- /dev/null
	+++ b/src/container.rs
	@@ -0,0 +1,40 @@
	+use crate::cli::Args;
	+use crate::errors::Errcode;
	+use crate::config::ContainerOpts;
	+
	+pub struct Container{
	+    config: ContainerOpts,
	+}
	+
	+impl Container {
	+    pub fn new(args: Args) -> Result<Container, Errcode> {
	+        let config = ContainerOpts::new(
	+                args.command,
	+                args.uid,
	+                args.mount_dir)?;
	+        Ok(Container {
	+            config,
	+        })
	+    }
	+
	+    pub fn create(&mut self) -> Result<(), Errcode> {
	+        log::debug!("Creation finished");
	+        Ok(())
	+    }
	+
	+    pub fn clean_exit(&mut self) -> Result<(), Errcode>{
	+        log::debug!("Cleaning container");
	+        Ok(())
	+    }
	+}
	+
	+pub fn start(args: Args) -> Result<(), Errcode> {
	+    let mut container = Container::new(args)?;
	+    if let Err(e) = container.create(){
	+        container.clean_exit()?;
	+        log::error!("Error while creating container: {:?}", e);
	+        return Err(e);
	+    }
	+    log::debug!("Finished, cleaning & exit");
	+    container.clean_exit()
	+}
	diff --git a/src/main.rs b/src/main.rs
	index 32c66c0..05856d7 100644
	--- a/src/main.rs
	+++ b/src/main.rs
	@@ -2,6 +2,8 @@ use std::process::exit;
	 
	 mod errors;
	 mod cli;
	+mod container;
	+mod config;
	 
	 use errors::exit_with_retcode;
	 
	@@ -9,7 +11,7 @@ fn main() {
		 match cli::parse_args(){
			 Ok(args) => {
				 log::info!("{:?}", args);
	-            exit_with_retcode(Ok(()))
	+            exit_with_retcode(container::start(args))
			 },
			 Err(e) => {
				 log::error!("Error while parsing arguments:\n\t{}", e);
</details>









# Checking the Linux kernel version

# Inter-process communication (IPC) with sockets

# Cloning a process

[rustbook-struct]: https://doc.rust-lang.org/book/ch05-01-defining-structs.html
[syscall-execve]: https://man7.org/linux/man-pages/man2/execve.2.html
[rustbook-cloneandcopy]: https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html#ways-variables-and-data-interact-clone
[so-stackheap]: https://stackoverflow.com/questions/79923/what-and-where-are-the-stack-and-heap
[tuto_contained.c]: https://blog.lizzie.io/linux-containers-in-500-loc.html#org39f5223
