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


# Create debian packages
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

# Debian package operation

## dpkg
- See help
    - dpkg --help
    - dpkg --force-help
- Print out the control file and other information for a specified package
    - dpkg --info foo.deb
- Install a package onto the file system of the hard disk
    - dpkg --install foo_VVV-RRR.deb
- List the files in a package deb file
    - dpkg -c foo.deb
- Remove a package (but not its configuration files)
    - dpkg --remove foo
- Remove a package (including its configuration files)
    - dpkg --purge foo
- Remove a package no matter that it is depended by other existing packages
    - dpkg --purge --force-depends foo
- List all files 'owned' by a package
    - dpkg --listfiles foo
- Find packages by file name.
    - dpkg --search <pattern>
- Find packages by package name.
    - dpkg --list <pattern>

      
## aptitude
aptitude provides the functionality of apt-get

- Install a new package
    - apt-get install foo
- Install a new package by force
    - apt-get -f install
- Synchronize the package index files from their sources
    - apt-get update
- Remove packages without configurations
    - apt-get remove foo
- Remove packages with configurations
    - apt-get purge foo
- Search packages
    - apt-cache search foo
        - Lists packages whose name or description contains foo
        - This queries and displays available information about installed and installable packages.

## dpkg-deb
- Determine what files are contained in a Debian archive file
    - dpkg-deb --contents foo.deb
## common questions
- What are [conffiles](https://www.debian.org/doc/manuals/maint-guide/dother.en.html#conffiles)?
    - conffiles are just configuration files of a package. When you upgrade a package, you'll be asked whether you want to keep your old configuration files or not.
    - Files under the /etc directory are marked as conffiles automatically.
    - If your program uses configuration files but also rewrites them on its own, it's best not to make them conffiles because dpkg will then prompt users to verify the changes all the time.
    - If the program you're packaging requires every user to modify the configuration files in the /etc directory, there are two popular ways to arrange for them to not be conffiles, keeping dpkg quiet:
        - Create a symlink under the /etc directory pointing to a file under the /var directory generated by the maintainer scripts.
        - Create a file generated by the maintainer scripts under the /etc directory.

- What packages are already installed?
    - dpkg --list
    - dpkg --list 'foo*'

- How to check detail of installed packages
    - dpkg --status foo
- How to display the files on an installed package?
    - dpkg --listfiles foo
- How can I find out what package produced a particular file?
    - dpkg --search foo
        - This will search all of the files having the file extension of .list in the directory /var/lib/dpkg/info/.

- Can multiple versions of the same package exist on the same system?
    - No.

- What happens when upgrade a binary debian package?
    - It will first delete all files created by the old version except conffiles, then install the new version. So there might be a small period that there will be no available/executable package.

- What are conffiles of an installed packages?

- How to install packages in place without breaking existing running service?
    - Put version in the package name 
- How to downgrade the version of a package?
- How to dealwith conffiles during package version up?
    ls /var/lib/dpkg/info/. | grep conffiles
# References

- [FAQ](https://www.debian.org/doc/manuals/debian-faq/ch-pkgtools.en.html) of dpkg tools

- How to use [apt-get](https://help.ubuntu.com/community/AptGet/Howto)?