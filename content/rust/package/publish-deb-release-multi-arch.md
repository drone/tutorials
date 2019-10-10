---
date: 2000-01-01T00:00:00+00:00
title: Publish multi-Architecture Debian packages for Rust
author: KN4CK3R
draft: true
image: https://dummyimage.com/650x300/253d5f/ffffff
keywords:
- rust
- deb
- github
tags:
- rust
- github
- deb
- package
- arm
- arm64
- amd64
---

This guide will walk you through the steps required to build multi-architecture Rust projects and publish the binaries as Debian packages on GitHub.
A package is a collection of files that allow for applications or libraries to be distributed via a package management system.
This guide assumes you have a basic understanding of Rust, Docker, Debian packages and GitHub.

{{< alert info >}}
To cross compile Rust projects this tutorial uses the sample Rust project and the Cross-Compilation Docker image presented in the tutorial {{< ref "publish-github-release-multi-arch.md" >}}.
{{< / alert >}}

# Cross Compiling the Rust Project

The goal is to compile the Rust project for three different platforms: `amd64`, `aarch64` and `armv7`.
To reach this goal we use the Cross-Compilation Docker image from the previous tutorial which supports all these platforms.
The Docker image contains the Rust build chain and can be used like in the following example:

```console
# Build amd64 (default)
$ docker run --rm -v <path to your project>:/rust-project -w /rust-project rust-multiarch cargo build
# Build aarch64
$ docker run --rm -v <path to your project>:/rust-project -w /rust-project rust-multiarch cargo build --target=aarch64-unknown-linux-gnu
# Build armv7
$ docker run --rm -v <path to your project>:/rust-project -w /rust-project rust-multiarch cargo build --target=armv7-unknown-linux-gnueabihf
```

This creates three compiled binaries:

```
target/debug/hello-world
target/aarch64-unknown-linux-gnu/debug/hello-world
target/armv7-unknown-linux-gnueabihf/debug/hello-world
```

# Creating a Debian Package

To create a Debian package, you first need to decide the name of the package.
To be compatible with other packages it should contain the name of the project in lowercase followed by the version number and architecture:

```
<project>_<version>-<revision>_<platform>
```

Following this format a possible name for the package could be `hello-world_1.0-1_amd64`. Create a folder with this name:

```
$ mkdir hello-world_1.0-1_amd64
```

## Add Application Data

This folder will contain all files and meta data needed to build a simple package.
The files in this folder will be unpacked from the package on the target system.
So this folder is equal to the root `/` folder on a system.
The binary should be placed in the `/bin` folder, so the package needs this struture too:

```
$ mkdir -p hello-world_1.0-1_amd64/bin
$ cp target/debug/hello-world hello-world_1.0-1_amd64/bin
```

## Add Package Meta Data

The next step is to create the necessary meta data file to describe the package.

```
$ mkdir hello-world_1.0-1_amd64/DEBIAN
$ touch hello-world_1.0-1_amd64/DEBIAN/control
```

The `control` file is a simple text file which contains the meta data of the package.
A complete documentation for this file can be found [on the official Debian site](https://www.debian.org/doc/debian-policy/ch-controlfields.html#binary-package-control-files-debian-control).
For a simple package only a minimal content is needed:

{{< highlight text "linenos=table,hl_lines=1-2 5-8" >}}
Package: hello-world
Version: 1.0-1
Section: base
Priority: optional
Maintainer: Name <address@mail.com>
Description: Hello World
 A Drone tutorial application.
Architecture: amd64

{{< / highlight >}}

{{< alert info >}}
The last line needs to be empty.
{{< / alert >}}

The imported lines are highlighted.

### The `Package` Attribute

The `Package` attribute contains the name of the package.

### The `Version` Attribute

The `Version` attribute contains the application version of the package.

### The `Maintainer` Attribute

The `Maintainer` attribute should contain your name and email address.

### The `Description` Attribute

The `Description` attribute contains a short description of the application.
If there are multiple lines they have to start with a space character.

### The `Architecture` Attribute

The `Architecture` attribute contains the platform where the package can be used.

## Building the Debian Package

The following tree shows the folder structure for the package:

```
hello-world_1.0-1_amd64
|-> DEBIAN
|---> control
|-> bin
|---> hello-world
```

To build the package [`dpkg-deb`](http://man7.org/linux/man-pages/man1/dpkg-deb.1.html) can be used:

```console
$ dpkg-deb --build hello-world_1.0-1_amd64
dpkg-deb: building package 'hello-world' in 'hello-world_1.0-1_amd64.deb'.
```

After the build is finished a new file `hello-world_1.0-1_amd64.deb` is present in the current folder.
The following screenshots show the package details:

![Debian Package Info](/screenshots/rust-deb-package-info.png)

To install the package [`dpkg`](http://man7.org/linux/man-pages/man1/dpkg.1.html) can be used:

```console
$ sudo dpkg -i hello-world_1.0-1_amd64.deb
Selecting previously unselected package hello-world.
(Reading database ... 185161 files and directories currently installed.)
Preparing to unpack hello-world_1.0-1_amd64.deb ...
Unpacking hello-world (1.0-1) ...
Setting up hello-world (1.0-1) ...
$ hello-world
Hello, world!
```

Now the build process can be automated with Drone!

# Continuous Integration and Delivery

This section shows how to configure [Drone](https://drone.io), an Open Source continuous integration system, to automatically build your project for multiple architectures and publish it as debian packages every time you create a git tag.
You can install Drone on your own servers, or you can use the free Cloud offering.

## Activation

Fist, login into Drone and activate your repository:

![Activate Repository](/screenshots/drone-repo-activate.png)

## Configuration

Next, define a [Continuous Integration Pipeline](https://docs.drone.io/configure/pipeline/) for your project.
Drone looks for a special `.drone.yml` file within repositories for the pipeline definition:

{{< highlight yaml "linenos=table,hl_lines=99" >}}
kind: pipeline
type: docker
name: build

steps:
- name: prepare-workspace
  image: debian
  commands:
    - mkdir assets

- name: build
  image: rust-multiarch
  commands:
    - cargo build
    - cargo build --target=aarch64-unknown-linux-gnu
    - cargo build --target=armv7-unknown-linux-gnueabihf

- name: create-armv7-package
  image: debian
  commands:
    - mkdir -p /tmp/_package/bin
    - cp target/armv7-unknown-linux-gnueabihf/debug/hello-world
    - mkdir /tmp/_package/DEBIAN
    - cp package_metadata /tmp/_package/DEBIAN/control
    - echo "Architecture: armv7" >> /tmp/_package/DEBIAN/control
    - dpkg-deb --build /tmp/_package
    - mv /tmp/_package.deb assets/hello-world_1.0-1_armv7.deb

- name: create-arm64-package
  image: debian
  commands:
    - mkdir -p /tmp/_package/bin
    - cp target/aarch64-unknown-linux-gnu/debug/hello-world /tmp/_package/bin
    - mkdir /tmp/_package/DEBIAN
    - cp package_metadata /tmp/_package/DEBIAN/control
    - echo "Architecture: arm64" >> /tmp/_package/DEBIAN/control
    - dpkg-deb --build /tmp/_package
    - mv /tmp/_package.deb assets/hello-world_1.0-1_arm64.deb

- name: create-amd64-package
  image: debian
  commands:
    - mkdir -p /tmp/_package/bin
    - cp target/debug/hello-world /tmp/_package/bin
    - mkdir /tmp/_package/DEBIAN
    - cp package_metadata /tmp/_package/DEBIAN/control
    - echo "Architecture: amd64" >> /tmp/_package/DEBIAN/control
    - dpkg-deb --build /tmp/_package
    - mv /tmp/_package.deb assets/hello-world_1.0-1_amd64.deb

- name: publish
  image: plugins/github-release
  settings:
    api_key:
      from_secret: github_token
    files: assets/*.deb
    title: Release Title

trigger:
  event:
  - tag
{{< / highlight >}}

The above example defines the pipeline steps as a series of shell commands executed inside Docker containers.
To learn more about how to write the pipeline config see the [official reference](https://docs.drone.io/configure/pipeline/).

### The `prepare-workspace` Step

This step is used to prepare the initial workspace for the other steps.
It creates an `assets` folder for the final `.deb` files.
The `package-assets` folder is used for the package application data.

{{< highlight yaml "linenos=table,linenostart=6" >}}
- name: prepare-workspace
  image: debian
  commands:
    - mkdir assets
{{< / highlight >}}

### The `build` Step

In this step the project gets compiled for the `amd64`, `arm64` and `armv7` platform.
After every build the resulting executable is copied into the `package-assets` folder.

{{< highlight yaml "linenos=table,linenostart=11" >}}
- name: build
  image: rust-multiarch
  commands:
    - cargo build
    - cargo build --target=aarch64-unknown-linux-gnu
    - cargo build --target=armv7-unknown-linux-gnueabihf
{{< / highlight >}}

### The `create-xxx-package` Steps

These steps are used to build the packages for every platform.
The `commands` are identical to the manual steps.
The build directory for the package is outside of the project folder so a cleanup is not necessary.
The only interesting part is the `echo "Architecture: armv7" >> /tmp/_package/DEBIAN/control` which inserts the architecture definition into the `DEBIAN/contro` file on the fly.
Because of the different `Architecture` attribute for every platform this line must be omitted in the template file `package_metadata`.

{{< highlight yaml "linenos=table,linenostart=18" >}}
- name: create-armv7-package
  image: debian
  commands:
    - mkdir -p /tmp/_package/bin
    - cp target/armv7-unknown-linux-gnueabihf/debug/hello-world
    - mkdir /tmp/_package/DEBIAN
    - cp package_metadata /tmp/_package/DEBIAN/control
    - echo "Architecture: armv7" >> /tmp/_package/DEBIAN/control
    - dpkg-deb --build /tmp/_package
    - mv /tmp/_package.deb assets/hello-world_1.0-1_armv7.deb
{{< / highlight >}}

{{< alert info >}}
As you can see this step needs to be duplicated for every platform.
To make the handling easier it should be converted into a Drone plugin.
Drone plugins are Docker images that perform pre-defined tasks.
Choose from hundreds of community plugins or create your own.
{{< / alert >}}

### The `publish` Step

Here the final packages get published as a GitHub release.
More informations about the configuration of this step can be found in the tutorial {{< ref "publish-github-release-multi-arch.md" >}}.

{{< highlight yaml "linenos=table,linenostart=51" >}}
- name: publish
  image: plugins/github-release
  settings:
    api_key:
      from_secret: github_token
    files: assets/*.deb
    title: Release Title
{{< / highlight >}}

### The `trigger` Condition

This section forces the pipeline to run only if a git tag was created.

{{< highlight yaml "linenos=table,linenostart=57" >}}
trigger:
  event:
  - tag
{{< / highlight >}}

## Execution

The Pipeline will execute every time you push code to your repository.
You can login to Drone and inspect the Pipeline results:

![Build Pipeline](/screenshots/drone-build-rust-deb.png)
