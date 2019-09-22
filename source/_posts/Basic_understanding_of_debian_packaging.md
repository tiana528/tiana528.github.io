---
title: Basic understanding of debian packaging
date: 2019-02-07 20:25:32
tags: debian package
---

# Background

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

# Debian package management tools

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
    - dpkg --search foo*
- Find packages by package name.
    - dpkg --list foo*

      
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

# Common questions

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
    - ls /var/lib/dpkg/info/. | grep conffiles
- How to install packages in place without breaking existing running service?
    - As we discribed before, updating a package will first delete all files owned by the current installed version, then install the new version. There is a small time window during the package versionup that there is no available/executable package. This should be considered when performing environment update. One solution is rolling update, never let a service in the middle of update provides actual service. The other is always installing new packages and do not update existing packages.
- How to downgrade the version of a package?
    - apt get install foo=1.2 --allow-downgrades
- How to dealwith conffiles during package version up?
    - [Everything you need to know about conffiles: configuration files managed by dpkg
](https://raphaelhertzog.com/2010/09/21/debian-conffile-configuration-file-managed-by-dpkg/)


# Scenarios
Here we discuss some scenarios.


- version up packages and revert the version up
    - Use `--allow-downgrades` option to downgrade a version(revert a release) without prompting.
        - e.g. current version of package p1 is 1.2, then run `sudo apt-get install p1=1.1 --allow-downgrades`

- check broken installed packages
    - Run `sudo apt-get update`

- override a file installed by another debian packages
    - Never do this. It is not maintainable. However, sometimes, we want to move some files(responsibilities) from one package to another package, and in such case, we need to release two packages together in the specific order.
        - e.g. Package p1 install file f1, we want to let package p2 install the file, then implement it and release new version of p1 and p2. During deployment, we first install new version of p1, which will delete the file f1 installed by the previous version, then install new version of p2 to avoid confliction.

- installation of some packages fail in the middle
    - This leaves a `held package`, detect a held package by `sudo dpkg --get-selections | grep held`.
    - If the broken package is caused by ramdomly issue, a retry might help fix the issue, run `sudo dpkg --configure -a`.
    - If the broken package is caused by package script issue, and needs to be fixed by installing a new version, then perform cleanup and install new version. `sudo apt-get remove p1` and `sudo apt-get install p1=...`

- list up local installed debian packages
    - sudo dpkg -l

- search remote installable debian packages
    - sudo apt-cache search foo


- install cron job
    - It is very common to create a crob job in a debian package, just create a file under /etc/cron.d instead of using the crontab command. Note that it is not good to use crontab command because if multiple packages use the crontab command to install cron jobs, they will all edit the same file and is hard to maintenance. Instead, copying a file under /etc/cron.d enables each package manages its cron jobs independently and isolately.

# References

- [FAQ](https://www.debian.org/doc/manuals/debian-faq/ch-pkgtools.en.html) of dpkg tools
- How to use [apt-get](https://help.ubuntu.com/community/AptGet/Howto)?
