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

## Applying it to the container configuration
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

## ...
``` rust
+use crate::errors::Errcode;
+use nix::unistd::sethostname;
+pub fn set_container_hostname(hostname: &String) -> Result<(), Errcode> {
+    match sethostname(hostname){
+        Ok(_) => {
+            log::debug!("Container hostname is now {}", hostname);
+            Ok(())
+        },
+        Err(_) => {
+            log::error!("Cannot set hostname {} for container", hostname);
+            Err(Errcode::HostnameError(0))
+        }
+    }
+}
```

# Modifying the container mount point
