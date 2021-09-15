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
# Getting started

So I guess you've heard that Rust's mascot is Ferris, the little cutie crab.
Well, let's put Ferris in a container ! :D

![Wanna eat Ferris ?](/assets/images/container_in_rust_part2/crabcan.png){: width="250" }

## Create the project
We'll create a Rust binary project called **Crabcan**, and the objective will be to separate the
different parts of the projects as distincly as possible to allow us to search in the code,
tweak it and understand it again after months of pause.

Run the command `cargo new --bin crabcan` to create the project
