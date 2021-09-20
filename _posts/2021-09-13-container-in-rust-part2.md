---
layout: post
title:  "Starting the project"
date:   2021-09-12 17:34:00 +0200
modified_date: 2021-09-14 12:00:00 +0200
categories: rust
tags: rust tutorial learning container docker
series: Writing a container in Rust
serie_index: 2
---

# Introduction
In this post, we will prepare the field for our implementation.

Every programmer is different and some of you may go  directly into Linux syscalls
to create an isolated box and so on ...
I prefer to always create a clean ground on which lay off my implementations, which
I find is much more practical to read/understand later on, and it's always a good thing
to practice clean implementations.

For the sake of this tutorial, that will also provide bonus tips and tools about topics
that may interest some less experiences Rust users, like **argument parsing**, **errors handling**,
**logging**, etc ...

## Create the project

So I guess you've heard that Rust's mascot is Ferris, the little cutie crab.
Well, let's put Ferris in a container ! :D

{:refdef: style="text-align: center;"}
![Wanna eat Ferris ?](/assets/images/container_in_rust_part2/crabcan.png){: width="250" }
{: refdef}

I will assume here that you already have `rustc` and `cargo` installed, if you don't, please
follow the instructions on [the Book](https://doc.rust-lang.org/book/ch01-01-installation.html)

### Cargo new
We'll create a Rust binary project called **Crabcan**, and the objective will be to separate the
different parts of the projects as distincly as possible to allow us to search in the code,
tweak it and understand it again after months of pause.

Run the command `cargo new --bin crabcan` to create the project

#### Cargo.toml
This will generate a `Cargo.toml` file in which we can describe our project, add dependencies and
tweak configurations of the rust compiler, a handy file to avoid having to avoid having to
create `rustc` commands by hand in a Makefile.

You can change the author name, e-mail and version of your project here, but we wont add any
dependencies yet.

#### src/main.rs
In the folder `src/` you will put, well, all your sources.
For now there's only a `main.rs` file with a `Hello World!` code inside.


# Parse the arguments
Ok let's dive directly into our project. First of all 

# Setup Logging

# Prepare errors handling

# Validate arguments

[rust-the-book]: https://doc.rust-lang.org/book/
