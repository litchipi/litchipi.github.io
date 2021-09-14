---
layout: post
title:  "What is a container ?"
date:   2021-09-12 17:34:00 +0200
modified_date: 2021-09-14 12:00:00 +0200
categories: rust
tags: rust tutorial learning container docker
series: Writing a container in Rust
serie_index: 1
---

The objective of this series of blog post is to understand what is a container, how does it work
and create a software to create and manage containers, from scratch in Rust.
The implementation will be based on the amazing
[Linux containers in 500 lines of code][linux-containers-tutorial] tutorial, but rewritten in Rust.

First of all, to understand how we're going to build our containerization software, 
let's figure out what is a container and why we need it,
we'll examine then few examples of container software commonly used.

# Overview

A container is an isolated execution environment providing an abstraction between a software to be
executed and the underlying operating system. It can be seen as a software virtualisation process.

So we basically tell a container manager "*Hey, execute that thing inside a isolated box*", the
manager create a box that will look like a system in which the software can execute
(properly configured), and execute the software.

## Portability
Portability is the ability of a software to be executed on various execution environment without
any compatibility issue, containers are one of the solutions to allow that kind of behaviour and
is used to ship software without needing to modify the system to install it.
So one can execute a software without installing it, and configure it to work on the system;
it'll just require to run a container manager (installed on the system), and let it handle the
software.

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




# Common container software

Of course, a lot of different solutions exist to create, execute and manage containers, but we're
going to inspect the 2 most famous one, **Docker** and **LXC**.

## Docker

## Linux Containers (LXC)

[docker-website-whatisdocker]: https://www.docker.com/resources/what-container
[linux-containers-tutorial]: https://blog.lizzie.io/linux-containers-in-500-loc.html
[understand-docker-container-escape]: https://blog.trailofbits.com/2019/07/19/understanding-docker-container-escapes/
