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
Ok let's dive directly into our project. First of all let's get the arguments from the command line.
The objective is to get configurations from text-written flags while calling our tool.

**Command example**
```
crabcan --mount ./mountdir/ --uid 0 --debug --command "bash"
```
This command will call `crabcan` with the folder `mountdir` to mount as root for the container,
the UID number `0`, will output all `debug` messages, and will execute the command `bash` inside
the container.

## Introducing the `structopt` crate
The [structopt crate][structopt-cratesio] is a very usefull tool to parse arguments from the
commandline (using the `clap` crates as a backend).
The method is very straightforward, by defining a [struct][rustbook-struct] containing all the arguments:
```rust
#[derive(Debug, StructOpt)]
#[structopt(name = "example", about = "An example of StructOpt usage.")]
struct Opt {
	/// Activate debug mode
	// short and long flags (-d, --debug) will be deduced from the field's name
	#[structopt(short, long)]
	debug: bool

	// etc ...
}
```
A detailed use of structopt and all its power is available in its [documentation][structopt-docs].
One thing worth noticing is that the `/// text` part above an argument defined in the struct
will be used as a message inside the helper (when you type `crabcan --help` for example).

## Creating our argument parsing
We are going to create a new file `src/cli.rs` containing everything related to commandline.
For it to be used inside our project, we have to include it as a [module][rustbook-module] of the
project.

In `src/main.rs` we replace the content with the following:
```rust
mod cli;

fn main() {
    let args = cli::parse_args();
}
```
Basically we expect the `src/cli.rs` file to provide a function `parse_args` that will return the
struct containing all our configuration defined by the user through the commandline. \\
*Note that because `args` is not used, you will get a compiler warning.*

Now let's implement that function `parse_args` in `src/cli.rs`:
```rust
use std::path::PathBuf;
use structopt::StructOpt;

#[derive(Debug, StructOpt)]
#[structopt(name = "crabcan", about = "A simple container in Rust.")]
pub struct Args {
    /// Activate debug mode
    #[structopt(short, long)]
    debug: bool,

    /// Command to execute inside the container
    #[structopt(short, long)]
    pub command: String,

    /// User ID to create inside the container
    #[structopt(short, long)]
    pub uid: u32,

    /// Directory to mount as root of the container
    #[structopt(parse(from_os_str), short = "m", long = "mount")]
    pub mount_dir: PathBuf,
}

pub fn parse_args() -> Args {
    let args = Args::from_args();

    // If args.debug: Setup log at debug level
    // Else: Setup log at info level

    // Validate arguments

    args
}
```
So here we first import our necessary dependencies `structopt` but also `PathBuf` from the standard
library. \\
Then we define our `Args` struct, containing all the arguments and informations to be used for
argument parsing. Let's look what arguments we are expecting:
- `debug`: Will be used to display debug messages or just normal logs
- `command`: The command that will be executed inside the container (with arguments)
- `uid`: The user ID that will be created to run the software inside the container.
- `mount_dir`: The folder to use as a root `/` directory inside the container. \\
*Note that this argument will be passed as **mount** in the commandline*

These arguments are defined with the [macro attribute][rustbook-macroattribute]
`structopt(short, long)` to automatically create a short and long commandline argument from the
field name.
(The field `toto` will be defined as arguments `-t` and `--toto`).

Finally, we create the `parse_args` in which we gather the arguments from the commandline with the
`from_args` function of the struct
(which  was generated thanks to the `derive(StructOpt)` [macro attribute][rustbook-macroattribute]).

After setting some placeholders for arguments validation and logging initialisation, we return
the arguments.

One last thing, add the dependencies we just imported inside the `Cargo.toml` file:
```toml
# ...
[dependencies]
structopt = "0.3.23"
```

## Testing our code

Let's test our code with `cargo run`:
```
error: The following required arguments were not provided:
    --command <command>
    --mount <mount-dir>
    --uid <uid>

USAGE:
    crabcan [FLAGS] --command <command> --mount <mount-dir> --uid <uid>

For more information try --help
```
And this is it, our argument parsing works !
Now if we try `cargo run -- --mount ./ --uid 0 --command "bash" --debug`, we don't get any errors.
You can add a `println!("{:?}", args)` in our `src/main.rs` file to get a nice output:
```
Args { debug: true, command: "bash", uid: 0, mount_dir: "./" }
```

The code for this step is available on github [litchipi/crabcan branch "step1"][code-step1]

# Setup Logging

# Prepare errors handling

# Validate arguments

[rust-the-book]: https://doc.rust-lang.org/book/
[rustbook-struct]: https://doc.rust-lang.org/stable/book/ch05-01-defining-structs.html
[rustbook-module]: https://doc.rust-lang.org/stable/book/ch07-02-defining-modules-to-control-scope-and-privacy.html
[rustbook-macroattribute]: https://doc.rust-lang.org/stable/book/ch19-06-macros.html?#attribute-like-macros
[structopt-cratesio]: https://crates.io/crates/structopt
[structopt-docs]: https://docs.rs/structopt/latest/structopt/
[code-step1]: https://github.com/litchipi/crabcan/tree/step1
