---
layout: post
title:  "What is a container ?"
date:   2021-09-12 17:34:00 +0200
categories: rust
tags: rust tutorial learning container
series: Writing a container in Rust
serie_index: 1
---

The objective of this series of blog post is to understand what is a container, how does it work
and create one from scratch in Rust.
The implementation will be based on the amazing
[Linux containers in 500 lines of code][linux-containers-tutorial] tutorial, but rewritten in Rust.

# Overview

## Portability
From the [Docker documentation website][docker-website-whatisdocker]:
> A container is a standard unit of software that packages up code and all its dependencies so the
> application runs quickly and reliably from one computing environment to another.
> A Docker container image is a lightweight, standalone, executable package of software that
> includes everything needed to run an application: code, runtime, system tools, system libraries
> and settings.

Okay, so it's a way to make a software run without the compatibility hassle with the underlying
execution environment. This is especially usefull to be able to develop a service and ship it to
different servers, laptops or even embedded devices without all the problems it raises.

## Security
But wait there's more to it, containerized application allows an isolation of the execution
environment with the underlying operating system, in the same philosophy as a virtual machine.
The difference between a container and a virtual machine is explained on the
[Docker documentation][docker-website-whatisdocker] also:
> Containers and virtual machines have similar resource isolation and allocation benefits,
> but function differently because containers virtualize the operating system instead of hardware.

So we can run a software having administrator rights and performing any kind of operations without
harming our system. Of course this is theorical and the real security will depend on a lot of
factors, including the implementation, to avoid container evasion (the same way we would want to
avoid virtual machine evasion), a great post about it is
[Understanding Docker container escapes][understand-docker-container-escape].

# Softwares

## Docker

## Linux Containers (LXC)




[docker-website-whatisdocker]: https://www.docker.com/resources/what-container
[linux-containers-tutorial]: https://blog.lizzie.io/linux-containers-in-500-loc.html
[understand-docker-container-escape]: https://blog.trailofbits.com/2019/07/19/understanding-docker-container-escapes/
