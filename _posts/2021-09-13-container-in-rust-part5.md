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
`ssh crab@crabcan.local`, the website `http://localhost:80` to `http://crabcan.local` (
See [the official website for more informations](http://avahi.org/))

# Modifying the container mount point
