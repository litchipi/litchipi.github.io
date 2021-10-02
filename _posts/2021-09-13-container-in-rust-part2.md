date:   2021-09-30 15:50:35 +0200
### Patch for this step
The code for this step is available on github [litchipi/crabcan branch "step1"][code-step1]. \\
Apply the following patch to a freshly created project using `cargo new --bin`
<details>
	diff --git a/Cargo.toml b/Cargo.toml
	index 498f536..5810b7b 100644
	--- a/Cargo.toml
	+++ b/Cargo.toml
	@@ -7,3 +7,4 @@ edition = "2018"
	 # See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

	 [dependencies]
	+structopt = "0.3.23"
	diff --git a/src/cli.rs b/src/cli.rs
	new file mode 100644
	index 0000000..e6f4cf8
	--- /dev/null
	+++ b/src/cli.rs
	@@ -0,0 +1,34 @@
	+use std::path::PathBuf;
	+use structopt::StructOpt;
	+
	+#[derive(Debug, StructOpt)]
	+#[structopt(name = "crabcan", about = "A simple container in Rust.")]
	+pub struct Args {
	+    /// Activate debug mode
	+    // short and long flags (-d, --debug) will be deduced from the field's name
	+    #[structopt(short, long)]
	+    debug: bool,
	+
	+    /// Command to execute inside the container
	+    #[structopt(short, long)]
	+    pub command: String,
	+
	+    /// User ID to create inside the container
	+    #[structopt(short, long)]
	+    pub uid: u32,
	+
	+    /// Directory to mount as root of the container
	+    #[structopt(parse(from_os_str), short = "m", long = "mount")]
	+    pub mount_dir: PathBuf,
	+}
	+
	+pub fn parse_args() -> Args {
	+    let args = Args::from_args();
	+
	+    // If args.debug: Setup log at debug level
	+    // Else: Setup log at info level
	+
	+    // Validate arguments
	+
	+    args
	+}
	diff --git a/src/main.rs b/src/main.rs
	index e7a11a9..2bf9e3f 100644
	--- a/src/main.rs
	+++ b/src/main.rs
	@@ -1,3 +1,6 @@
	+mod cli;
	+
	 fn main() {
	-    println!("Hello, world!");
	+    let args = cli::parse_args();
	+    println!("{:?}", args);
	 }
</details>
## The logging crates
Now that we got from the user its input, let's set up a way to give him outputs.
Simple text is enough, but we want to separate debug informations from basic informations and
errors. For this, there's a lot of tools, but I chose the crates `log` and `env_logger` to perform
this task.

The [log crate][log-cratesio] is a very used tool to perform logging. It provides a `Log`
trait ([see the Book for traits explanation][rustbook-trait]) which defines all the function a logger has to have, and lets any other
crate implement these functions. I chose the [env_logger crate][env_logger-cratesio] to implement
these.

In `Cargo.toml`, we add the following dependencies:
``` toml
# ...
log = "0.4.14"
env_logger = "0.9.0"
```

## Setting up logging
Loggers have to be initialized with a level of verbosity. This will define wether to display
debug messages, or only errors, or nothing at all.
On our case, we want it to display normal informations by default, and increase verbosity to
debug messages when the `--debug` flag is passed through the commandline.

Let's initialize our logger in `src/cli.rs`:
``` rust
pub fn setup_log(level: log::LevelFilter){
    env_logger::Builder::from_default_env()
        .format_timestamp_secs()
        .filter(None, level)
        .init();
}
```
Yeah, a function is not really needed, but it's more readable isn't it ? \\
If you are into [Rust code optimisation][rustcodeopti], you may want to
[inline this function][rustbook-inlining].

Ok, now let's actually initialize logging right after getting the arguments from the commandline,
in the `parse_args` functions, let's replace the placeholders with this piece of code:
``` rust
if args.debug{
	setup_log(log::LevelFilter::Debug);
} else {
	setup_log(log::LevelFilter::Info);
}
```

## Logging

Now that everything is in place, let's actually log something in our terminal !
In the `main` function of `src/main.rs`, we can output the args gotten into a `info` message.
This is done using the `log::info!` [macro][rustbook-macros].
``` rust
log::info!("{:?}", args);
```
The `log` crate allows us to use `error!`, `warn!`, `info!`, `debug!` or `trace!` message levels.

After testing we get the output:
```
[2021-09-30T10:17:46Z INFO  crabcan] Args { debug: true, command: "/bin/bash", uid: 0, mount_dir: "./mountdir/" }
```

### Patch for this step
The code for this step is available on github [litchipi/crabcan branch "step2"][code-step2]. \\
Apply the following patch to the code got from previous step
<details>
	diff --git a/Cargo.toml b/Cargo.toml
	index 5810b7b..1767c57 100644
	--- a/Cargo.toml
	+++ b/Cargo.toml
	@@ -8,3 +8,5 @@ edition = "2018"

	 [dependencies]
	 structopt = "0.3.23"
	+log = "0.4.14"
	+env_logger = "0.9.0"
	diff --git a/src/cli.rs b/src/cli.rs
	index e6f4cf8..0d4a2ef 100644
	--- a/src/cli.rs
	+++ b/src/cli.rs
	@@ -25,10 +25,20 @@ pub struct Args {
	 pub fn parse_args() -> Args {
		 let args = Args::from_args();

	-    // If args.debug: Setup log at debug level
	-    // Else: Setup log at info level
	+    if args.debug{
	+        setup_log(log::LevelFilter::Debug);
	+    } else {
	+        setup_log(log::LevelFilter::Info);
	+    }

		 // Validate arguments

		 args
	 }
	+
	+pub fn setup_log(level: log::LevelFilter){
	+    env_logger::Builder::from_default_env()
	+        .format_timestamp_secs()
	+        .filter(None, level)
	+        .init();
	+}
	diff --git a/src/main.rs b/src/main.rs
	index 2bf9e3f..05a9d8d 100644
	--- a/src/main.rs
	+++ b/src/main.rs
	@@ -2,5 +2,5 @@ mod cli;

	 fn main() {
		 let args = cli::parse_args();
	-    println!("{:?}", args);
	+    log::info!("{:?}", args);
	 }
</details>

As a general practise it's good to take care of handling errors.
When it comes to Rust, this langage is far too powerfull concerning errors handling to ignore
them and not exploit them.

I am no-one to teach how to properly handle errors, but this part will give an exemple of how
errors can be managed in a large Rust project, and use Rust specific tools to handle them more
easily.

## The Errcode enum
Let's create a `src/errors.rs` file in which we'll define the following [enum][rustbook-enum]:
``` rust
// Allows to display a variant with the format {:?}
#[derive(Debug)]
// Contains all possible errors in our tool
pub enum Errcode{
}
```
Each time we will add a new error type, we'll add a variant to this enum.
The `derive(Debug)` allows the enum to be displayed using a `{:?}` format.

But we may want to display a more complete message for each variant, allowing us to not get lost
in codes and different numbers around our project.
For this, let's implement the `std::fmt::Display` [trait][rustbook-trait], defining the behaviour
of an object when attempting to display it in a regular `{}` format.
``` rust
use std::fmt;

#[allow(unreachable_patterns)]
// trait Display, allows Errcode enum to be displayed by:
//      println!("{}", error);
//  in this case, it calls the function "fmt", which we define the behaviour below
impl fmt::Display for Errcode {

    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        // Define what behaviour for each variant of the enum
        match &self{
            _ => write!(f, "{:?}", self) // For any variant not previously covered
        }
    }
}
```
> The `unreachable_patterns` attribute ensure that we won't get any warning from the compiler
> if the `match` statement describe all the variants.

## Linux return codes
Linux executable returns a number when they exit, which describe how everything went.
A return code of **0** means that there was no errors, and any other number describe an
error and what that error is (based on the return code value).
You can find [here][exitcode-meanings] a table of special return codes and their meaning.

We do not seek to perform bash automation scripts here with our tool, but we'll just set up a
way to return 0 if there was no errors, and 1 if there was an error. \\
In our `src/errors.rs` file, let's define a `exit_with_errcode` function:
``` rust
use std::process::exit;

// Get the result from a function, and exit the process with the correct error code
pub fn exit_with_retcode(res: Result<(), Errcode>) {
    match res {
        // If it's a success, return 0
        Ok(_) => {
            log::debug!("Exit without any error, returning 0");
            exit(0);
        },

        // If there's an error, print an error message and return the retcode
        Err(e) => {
            let retcode = e.get_retcode();
            log::error!("Error on exit:\n\t{}\n\tReturning {}", e, retcode);
            exit(retcode);
        }
    }
}
```

This function exit the process with a return code got from the `get_retcode` function implemented
by our `Errcode` enum. Let's implement it in the most easy and stupid way:
``` rust
impl Errcode{
    // Translate an Errcode::X into a number to return (the Unix way)
    pub fn get_retcode(&self) -> i32 {
        1 // Everything != 0 will be treated as an error
    }
}
```

## Result in Rust
When a piece of the code is not working properly, we can handle the error using a `Result` in Rust
([see the Book for detailed explanation][rustbook-results]). \\
`Result<T, U>` expects two types, one type `T` to return if it's a success, one type `U` to return
if there's an error. In our case, we want to return an `Errcode` if there's an error, and return
whatever we want if everything goes well.

Let's see how we can set this up in the `parse_args` function:
``` rust
pub fn parse_args() -> Result<Args, Errcode> {
	// ...
	Ok(args)
 }
```
If something goes wrong during the execution, we can simply write:
``` rust
return Err(Errcode::MyErrorType);
```

> The `Result` in Rust are very usefull and powerfull and it's generally a good idea to use it
> everywhere you want error handling as it's the standard Rust way to do.

Okay, but now we need to do something different in our `main` depending on how the function ended
with an error or a success. Let's use a `match` statement to define what to do in both cases:
``` rust
match cli::parse_args(){
	Ok(args) => {
		log::info!("{:?}", args);
		exit_with_retcode(Ok(()))
	},
	Err(e) => {
		log::error!("Error while parsing arguments:\n\t{}", e);
		exit(e.get_retcode());
	}
};
```
Here, in case the arguments parsing was successful, we log the args and call the
`exit_with_retcode` with an `Ok(())` value (it will simply exit with the return code 0).
That is where we're going to place our container starting point later.

In case there was an error, we log it (notice the `{}` format on our `Errcode` that will
call the `fmt` function of the `Display` trait we implemented earlier), and simply exit with the
return code associated.

One final step, we have to set `src/errors.rs` as a module of our project, and import the 
`exit_with_retcode` function in our `src/main.rs` file.
``` rust
mod errors;
 
use errors::exit_with_retcode;
```

After testing, we can get the following output:
```
[2021-09-30T13:47:45Z INFO  crabcan] Args { debug: true, command: "/bin/bash", uid: 0, mount_dir: "./mountdir/" }
[2021-09-30T13:47:45Z DEBUG crabcan::errors] Exit without any error, returning 0
```

### Patch for this step
The code for this step is available on github [litchipi/crabcan branch "step3"][code-step3]. \\
Apply the following patch to the code got from previous step
<details>
	diff --git a/src/cli.rs b/src/cli.rs
	index 0d4a2ef..ba58ec2 100644
	--- a/src/cli.rs
	+++ b/src/cli.rs
	@@ -1,3 +1,5 @@
	+use crate::errors::Errcode;
	+
	 use std::path::PathBuf;
	 use structopt::StructOpt;
	 
	@@ -22,7 +24,7 @@ pub struct Args {
		 pub mount_dir: PathBuf,
	 }
	 
	-pub fn parse_args() -> Args {
	+pub fn parse_args() -> Result<Args, Errcode> {
		 let args = Args::from_args();
	 
		 if args.debug{
	@@ -33,7 +35,7 @@ pub fn parse_args() -> Args {
	 
		 // Validate arguments
	 
	-    args
	+    Ok(args)
	 }
	 
	 pub fn setup_log(level: log::LevelFilter){
	diff --git a/src/errors.rs b/src/errors.rs
	new file mode 100644
	index 0000000..268022b
	--- /dev/null
	+++ b/src/errors.rs
	@@ -0,0 +1,48 @@
	+use std::fmt;
	+use std::process::exit;
	+
	+// Allows to display a variant with the format {:?}
	+#[derive(Debug)]
	+// Contains all possible errors in our tool
	+pub enum Errcode{
	+}
	+
	+impl Errcode{
	+    // Translate an Errcode::X into a number to return (the Unix way)
	+    pub fn get_retcode(&self) -> i32 {
	+        1 // Everything != 0 will be treated as an error
	+    }
	+}
	+
	+
	+#[allow(unreachable_patterns)]
	+// trait Display, allows Errcode enum to be displayed by:
	+//      println!("{}", error);
	+//  in this case, it calls the function "fmt", which we define the behaviour below
	+impl fmt::Display for Errcode {
	+
	+    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
	+        // Define what behaviour for each variant of the enum
	+        match &self{
	+            _ => write!(f, "{:?}", self) // For any variant not previously covered
	+        }
	+    }
	+}
	+
	+// Get the result from a function, and exit the process with the correct error code
	+pub fn exit_with_retcode(res: Result<(), Errcode>) {
	+    match res {
	+        // If it's a success, return 0
	+        Ok(_) => {
	+            log::debug!("Exit without any error, returning 0");
	+            exit(0);
	+        },
	+
	+        // If there's an error, print an error message and return the retcode
	+        Err(e) => {
	+            let retcode = e.get_retcode();
	+            log::error!("Error on exit:\n\t{}\n\tReturning {}", e, retcode);
	+            exit(retcode);
	+        }
	+    }
	+}
	diff --git a/src/main.rs b/src/main.rs
	index 05a9d8d..ededaaf 100644
	--- a/src/main.rs
	+++ b/src/main.rs
	@@ -1,6 +1,16 @@
	+use std::process::exit;
	+
	+mod errors;
	 mod cli;
	 
	+use errors::exit_with_retcode;
	+
	 fn main() {
	-    let args = cli::parse_args();
	-    log::info!("{:?}", args);
	+    match cli::parse_args(){
	+        Ok(_args) => exit_with_retcode(Ok(())),
	+        Err(e) => {
	+            log::error!("Error while parsing arguments:\n\t{}", e);
	+            exit(1);
	+        }
	+    };
	 }
</details>
Before diving into the real work, let's validatet the arguments passed from the commandline.
We will just check that the `mount_dir` actually exists, but this part can be extended with
additionnal checks, as we add more options, etc ... \\
Let's replace the placeholders in `src/cli.rs` with the actual arguments validation:
``` rust
pub fn parse_args() -> Result<Args, Errcode> {
	// ...
	if !args.mount_dir.exists() && !args.mount_dir.is_dir(){
		return Err(Errcode::ArgumentInvalid("mount"));
	}
	// ...
}
```
The condition checks if the path (a `PathBuf` type as we defined in our `Args` struct) exists
and if it's a directory.

If it isn't, we return a `Result::Err` with our `Errcode` enum with a custom variant
`ArgumentInvalid`, specifying that the fault was on argument `mount`. \\
In `src/errors.rs`, we will define this variant:
``` rust
pub enum Errcode{
	ArgumentInvalid(&'static str),
}
```
And we can add in the `match` statement of the `fmt` function the following:
``` rust
match &self{
	// Message to display when an argument is invalid
	Errcode::ArgumentInvalid(element) => write!(f, "ArgumentInvalid: {}", element),

	// ...
}
```

### Patch for this step
The code for this step is available on github [litchipi/crabcan branch "step4"][code-step4]. \\
Apply the following patch to the code got from previous step
<details>
	diff --git a/src/cli.rs b/src/cli.rs
	index ba58ec2..821dab6 100644
	--- a/src/cli.rs
	+++ b/src/cli.rs
	@@ -33,7 +33,9 @@ pub fn parse_args() -> Result<Args, Errcode> {
			 setup_log(log::LevelFilter::Info);
		 }
	 
	-    // Validate arguments
	+    if !args.mount_dir.exists() && !args.mount_dir.is_dir(){
	+        return Err(Errcode::ArgumentInvalid("mount"));
	+    }
	 
		 Ok(args)
	 }
	diff --git a/src/errors.rs b/src/errors.rs
	index 268022b..90a8248 100644
	--- a/src/errors.rs
	+++ b/src/errors.rs
	@@ -5,6 +5,7 @@ use std::process::exit;
	 #[derive(Debug)]
	 // Contains all possible errors in our tool
	 pub enum Errcode{
	+    ArgumentInvalid(&'static str),
	 }
	 
	 impl Errcode{
	@@ -24,6 +25,10 @@ impl fmt::Display for Errcode {
		 fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
			 // Define what behaviour for each variant of the enum
			 match &self{
	+
	+            // Message to display when an argument is invalid
	+            Errcode::ArgumentInvalid(element) => write!(f, "ArgumentInvalid: {}", element),
	+
				 _ => write!(f, "{:?}", self) // For any variant not previously covered
			 }
		 }
</details>

[rustbook-trait]: https://doc.rust-lang.org/book/ch10-02-traits.html
[rustbook-inlining]: https://nnethercote.github.io/perf-book/inlining.html
[rustbook-macros]: https://doc.rust-lang.org/book/ch19-06-macros.html
[rustbook-enum]: https://doc.rust-lang.org/book/ch06-01-defining-an-enum.html
[rustbook-results]: https://doc.rust-lang.org/book/ch09-02-recoverable-errors-with-result.html
[log-cratesio]: https://crates.io/crates/log
[env_logger-cratesio]: https://crates.io/crates/env_logger
[code-step2]: https://github.com/litchipi/crabcan/tree/step2
[code-step3]: https://github.com/litchipi/crabcan/tree/step3
[code-step4]: https://github.com/litchipi/crabcan/tree/step4
[rustcodeopti]: https://gist.github.com/jFransham/369a86eff00e5f280ed25121454acec1
[exitcode-meanings]: https://tldp.org/LDP/abs/html/exitcodes.html