---
layout: post
date:   2023-01-11 17:17:04 +0200
categories: nix
tags: nix web
title:  "Building web applications with Nix easily"
---

## Introduction

In this post I will present a little Nix library made to build web applications
the proper way.

The idea is to **always** keep it simple for the developper and have a nice API,
but still do the job.

As this is a first step only, and only tailored to my personnal needs,
for now it's limited. \\
The whole purpose of this project is *to gather contributions* for as many
backends and frontends possibilities as possible.

Throughout the presentation, I will assume that you are using **nix flakes** and
that the library is added to the inputs like so:

``` nix
{
    description = "My web app";

    inputs = {
        nixpkgs.url = github:NixOS/nixpkgs/22.11;
        flake-utils.url = github:numtide/flake-utils;
        nix_web_lib.url = github:litchipi/nix_web_lib;
    };

    outputs = inputs: with inputs; flake-utils.lib.eachDefaultSystem (system: let
        weblib = nix_web_lib.lib.${system};
    in {
        # ...
    });
}
```

In the examples I will use `weblib` directly, which is simply the library imported
for the system you use.

## Building the backend

Building the backend of an application is essentially the same as building any
pieces of software, the process is then no different. \\
The objective of the library is then to offer a nice API to build the all kinds
of possible backends flavours.

How to start the backend is up to the developper using the lib, and he'll be
able to pass whatever argument and environment variables he wants on this*.

As of today, **only Rust backend** is implemented (tested with `actix-web` framework)
using [cargo2nix](https://github.com/cargo2nix/cargo2nix)
so you need a `Cargo.nix` file inside the source directory. \\
It builds like so:

``` nix
backend = weblib.backend.rust.build {
    src = ./backend;
    bin_name = "my_binary_name";      # Default binary name is "backend"

    # Any argument to pass to "pkgs.rustBuilder.makePackageSet" function
    rustBuilderArgs = {
        rustChannel = "stable";
        rustVersion = "1.61.0";
    };
};
```

There is of course room for improvements as this is a very simple example that
may not be convenient enough for some use cases. But **you can contribute easily**
to make it better (see last section).

## Building the frontend

Where the library really makes it better, is to handle building the frontend
frameworks, which I found was really difficult to do because of the tendency
of Javascript to fetch things from Internet at compile time. (*ugh*)

Fortunately, I found [this answer][buildreactdiscourse] on discourse that
helped me build a basic React frontend, and then translate the process for
VueJS also.

This seam like a bit of a hack solution, but it works well so far and I've
been able to implement the API with a simple function:

``` nix
                    # Replace with "vue" if using VueJS
frontend_compiled_dir = weblib.frontend.react.build {
    src = ./frontend;
};
```

**Huge thanks to enobayram** for finding a solution to make this happend.

The workflow when developping the frontend would be to simply fire up
the backend locally and develop using your usual tools, but for the
final project to compile you'll need to have a `yarn.lock` file.

## Starting a database

Working with databases is not really the funniest thing, and when developping
and application it can be annoying to go against all kinds of database issues
when testing the behaviour.

To help with that, the idea is to provide to the library all sorts of utility
functions to help manage a database for testing locally.

For now only `postgresql` is implemented, and this is how easy it gets to
start the database locally:

``` nix
db_script = let
    dblib = weblib.database.postgresql;

    name = "my_web_app";
    args = {
        dir = "./testdb";
        host = "localhost";
        dbname = "${name}_db";
        user = "${name}_user";
        port = 5465;
    };
in ''
    set -e
    ${dblib.init_db args }            # Init
    ${dblib.pg_ctl args "start" }     # Start
    ${dblib.db_check_connected args } # Check started OK
    ${dblib.ensure_user_exists args } # Create user if doesn't exist
    ${dblib.ensure_db_exists args }   # Created DB if doesn't exist
    ${dblib.stop_on_interrupt args }  # Trap keyboard interrupt to stop db

    tail -f ${args.dir}/logs          # Display the logs
'';
```

## How can you contribute

For now, the usage of this library is very limited, however it is fairly easy
to contribute to implement much more frameworks !

Of course, you still have all the original ways to contribute, open an issue,
fork the repo, propose a PR, etc ...

But the repo is also made to *have examples of backends / frontends / database*,
and their associated flake to build them ! So if you want the library to add support
for another backend, or even one of the many Javascript framework for frontends,
you simply have to **add an example project**.

Then, you describe in the `README.md` how the
thing is supposed to be built *using the native tools* ! \\
From that, it'll be easier for me or any other contributor to take on the example
project and modify the library so it will work properly. \\
**No nix knowledge required**

Of course, in case of, you can contact me

[buildreactdiscourse]: https://discourse.nixos.org/t/how-to-use-nix-to-build-a-create-react-app-project/5200/10
