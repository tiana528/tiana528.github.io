---
title: Use debian packaging to help setup environments
date: 2019-02-07 20:25:32
tags: debian package
---

# background

It never becomes easy to setup/install/upgrade environments. This article aims at describing how to use/create debian packages to help setup the environments.

## What is a debian package

- Linux
    - [Linux](https://en.wikipedia.org/wiki/Linux) is a family of free and open-source software operating systems based on the Linux kernel.
- Linux kernel
    - [Linux kernel](https://en.wikipedia.org/wiki/Linux_kernel) is a free and open-source, monolithic, Unix-like operating system kernel.
- Linux distribution
    - roughly : Linux distribution = Linux kernel + software
    - A [Linux distribution](https://en.wikipedia.org/wiki/Linux_distribution) is an operating system made from a software collection, which is based upon the Linux kernel and, often, a package management system. There are more than 600 kinds of linux distributions.
- Debian
    - Debian is one of the Linux distributions, also called Debian GNU/Linux.
    - [Debian](https://en.wikipedia.org/wiki/Debian) is one of the earliest operating systems based on the Linux kernel, and containing more than 51000 [debian packges](https://www.debian.org/index.en.html)
- Debian package
    - A [Debian package](https://wiki.debian.org/Packaging) is a collection of files that allow for applications or libraries to be distributed via the Debian package management system. 
- Relationship between debian packages and linux distributions
    -  [Linux distributions](https://en.wikipedia.org/wiki/List_of_Linux_distributions) can be divided into several categories according to how packages are managed. PRM-based(.rpm file) distributions includes Red Hat Linux, CentOS, Fedora and so on. Debian-based(.deb file) distributions includes Ubuntu, GNU/Linux and so on.

# Debian package operation

## Create debian packages
It is possible to use the official tool(dpkg-deb) to create debian packages. But here we use [fpm](https://github.com/jordansissel/fpm) to create debian packages. It is very simple to use.

Refer [this](https://fpm.readthedocs.io/en/latest/installing.html) to install fpm.

### Example1
Create a debian package whose name is "hello_world", and what it do is installing a file "hello world" in the path of /tmp/hi.txt

It is very easy, just execute 3 commands.

```bash
mkdir -p working_folder/tmp
echo "hello world" > working_folder/tmp/hi.txt
fpm \
  --architecture=amd64 \
  --chdir=working_folder \
  --input-type=dir \
  --output-type=deb \
  --name=hello_world \
  --version="0.0.1" \
  --iteration="001" \
  --description="This is an example" \
  --deb-user="root" \
  --deb-group="root" \
  --deb-priority="extra" \
  "."
```
Then a file of hello-world_0.0.1-001_amd64.deb will be created in the current folder.
Use `sudo dpkg -i hello-world_0.0.1-001_amd64.deb` to install the deb file, and then you can see a file(/tmp/hi.txt) containing "hello world" is created.

## Example2
This is an example to create a symlink /etc/link which links to /tmp/hi.txt. This example shows the usage of `--after-install` to do something at the end of package installation.

Create a file hook-AfterInstall.sh.
```
#!/usr/bin/env bash
ln -s /tmp/hi.txt /etc/link
```
Then run 3 commands.
```bash
mkdir -p working_folder/tmp
echo "hello world" > working_folder/tmp/hi.txt
fpm \
  --architecture=amd64 \
  --chdir=working_folder \
  --input-type=dir \
  --output-type=deb \
  --name=hello_world \
  --version="0.0.1" \
  --iteration="002" \
  --description="This is an example" \
  --deb-user="root" \
  --deb-group="root" \
  --deb-priority="extra" \
  --after-install="hook-AfterInstall.sh" \
  "."
```
Then hello-world_0.0.1-002_amd64.deb is created in the current folder. Installing it will create a symlink.

## apt

## dpkg
dpkg plays with .deb file, following are several useful commands.
sudo dpkg -L hadoop-node-common

sudo dpkg --purge --force-depends hadoop-node-common

## common questions
- What are conffiles?
    - [conffiles](https://www.debian.org/doc/manuals/maint-guide/dother.en.html#conffiles)

