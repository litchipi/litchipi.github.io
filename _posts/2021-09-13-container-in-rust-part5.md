---
layout: post
title:  "Defining the container environment"
date:   2021-09-12 17:34:00 +0200
modified_date: 2021-09-14 12:00:00 +0200
categories: rust
tags: rust tutorial learning container docker
series: Writing a container in Rust
serie_index: 5
serie_url: /series/container_in_rust
---

# Setting the container hostname
A hostname is what identifies our machine compared to every other living on the same network.

It is used by many different networking softwares, for example `avahi` is a software that streams
our hostname in the local network, allowing a command `ssh crab@192.168.0.42` to become
`ssh crab@crabcan.local`, the website `http://localhost:80` to `http://crabcan.local`, etc ...
> Check [the official website of avahi](http://avahi.org/) for more informations

In order to differentiate the operations performed by our software contained from the one 
performed by the host system, we will modify its hostname.

## Generate a hostname
First of all, create a new file called `src/hostname.rs`, in which we will write any code
related to the hostname. \\
Inside, set two arrays of pre-defined names and adjectives that we'll use together to
generate a stupid random hostname.

``` rust
const HOSTNAME_NAMES: [&'static str; 8] = [
    "cat", "world", "coffee", "girl",
    "man", "book", "pinguin", "moon"];

const HOSTNAME_ADJ: [&'static str; 16] = [
    "blue", "red", "green", "yellow",
    "big", "small", "tall", "thin",
    "round", "square", "triangular", "weird",
    "noisy", "silent", "soft", "irregular"];
```

We then generate a string using some randomness:

``` rust
use rand::Rng;
use rand::seq::SliceRandom;

pub fn generate_hostname() -> Result<String, Errcode> {
    let mut rng = rand::thread_rng();
    let num = rng.gen::<u8>();
    let name = HOSTNAME_NAMES.choose(&mut rng).ok_or(Errcode::RngError)?;
    let adj = HOSTNAME_ADJ.choose(&mut rng).ok_or(Errcode::RngError)?;
    Ok(format!("{}-{}-{}", adj, name, num))
}
```

> We obtain a hostname in the form *square-moon-64*, *big-pinguin-2*, etc ...

As we used a new `Errcode::RngError` to handle errors linked to the randomness functions, we add
this variant to our `Errcode` enum in `src/errors.rs`, along with another one that we'll use later,
the `Errcode::HostnameError(u8)` variant:
``` rust
pub enum Errcode {
    // ...
    HostnameError(u8),
    RngError
}
```

Also, we use the `rand` crate to have randomness in our hostname generation, so we have to add
it to the dependencies of `Cargo.toml`:
``` toml
[dependencies]
# ...
rand = "0.8.4"
```

## Adding to the configuration of the container
Now that we have a way to generate a `String` containing our "random" hostname, we can use it to set our container
configuration in `src/config.rs`:

``` rust
use crate::hostname::generate_hostname;

pub struct ContainerOpts{
    // ...
    pub hostname:   String,
}

impl ContainerOpts {
    // ...
    hostname: generate_hostname()?,
}
```

And finally, we can create in `src/hostname.rs` the function that will modify the actual hostname
of our *host namespace* with the new one, using the `sethostname` syscall:

``` rust
use crate::errors::Errcode;

use nix::unistd::sethostname;

pub fn set_container_hostname(hostname: &String) -> Result<(), Errcode> {
    match sethostname(hostname){
        Ok(_) => {
            log::debug!("Container hostname is now {}", hostname);
            Ok(())
        },
        Err(_) => {
            log::error!("Cannot set hostname {} for container", hostname);
            Err(Errcode::HostnameError(0))
        }
    }
}
```

> Check the [linux manual][man-sethostname] for more informations on the `sethostname` syscall

## Applying the configuration to the child process

For all the configuration we will apply to the child process, let's create a wrapping functions
setting everything, and add the `set_container_hostname` function call inside it, in `src/child.rs`:

``` rust
use crate::hostname::set_container_hostname;

fn setup_container_configurations(config: &ContainerOpts) -> Result<(), Errcode> {
    set_container_hostname(&config.hostname)?;
    Ok(())
}
```

And we then simply call the configuration function at the beginning of our child process:

``` rust
fn child(config: ContainerOpts) -> isize {
    match setup_container_configurations(&config) {
        Ok(_) => log::info!("Container set up successfully"),
        Err(e) => {
            log::error!("Error while configuring container: {:?}", e);
            return -1;
        }
    }
	// ...
}
```

Note that we cannot "recover" from any error hapenning in our child process, so we simply end it
with a `retcode = -1` along with a nice error message in case a problem occurs.

The final thing to do here is adding to `src/main.rs` the `hostname` module we just created:
``` rust
// ...
mod hostname;
```

## Testing
When testing, we can see the hostname we generated appear:
```
[2021-11-15T09:07:38Z INFO  crabcan] Args { debug: true, command: "/bin/bash", uid: 0, mount_dir: "./mountdir/" }
[2021-11-15T09:07:38Z DEBUG crabcan::container] Linux release: 5.13.0-21-generic
[2021-11-15T09:07:38Z DEBUG crabcan::container] Container sockets: (3, 4)
[2021-11-15T09:07:38Z DEBUG crabcan::container] Creation finished
[2021-11-15T09:07:38Z DEBUG crabcan::container] Container child PID: Some(Pid(26003))
[2021-11-15T09:07:38Z DEBUG crabcan::container] Waiting for child (pid 26003) to finish
[2021-11-15T09:07:38Z DEBUG crabcan::hostname] Container hostname is now weird-moon-191
[2021-11-15T09:07:38Z INFO  crabcan::child] Container set up successfully
[2021-11-15T09:07:38Z INFO  crabcan::child] Starting container with command /bin/bash and args ["/bin/bash"]
[2021-11-15T09:07:38Z DEBUG crabcan::container] Finished, cleaning & exit
[2021-11-15T09:07:38Z DEBUG crabcan::container] Cleaning container
[2021-11-15T09:07:38Z DEBUG crabcan::errors] Exit without any error, returning 0
```
And running it several times outputs different funny names :D
```
[2021-11-15T09:08:33Z DEBUG crabcan::hostname] Container hostname is now round-cat-221

[2021-11-15T09:08:48Z DEBUG crabcan::hostname] Container hostname is now silent-man-45

[2021-11-15T09:09:01Z DEBUG crabcan::hostname] Container hostname is now soft-cat-149
```

### Patch for this step

The code for this step is available on github [litchipi/crabcan branch “step9”][code-step9]. \\
The raw patch to apply on the previous step can be found [here][patch-step9]

# Modifying the container mount point


[man-sethostname]: https://linux.die.net/man/2/sethostname

[code-step9]: https://github.com/litchipi/crabcan/tree/step9/
[patch-step9]: https://github.com/litchipi/crabcan/commit/step9.diff
