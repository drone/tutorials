---
date: 2000-01-01T00:00:00+00:00
title: Publish multi-Architecture Debian packages for Rust
author: KN4CK3R
draft: true
image: /static/header-images/rust-deb.png
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

# Simplify Package Creation

The package creation should be automated to better fit into a continuous integration environment.
A simple shell script (call it `pack.sh`) can combine all needed steps and can be transformed into a Drone plugin later.

```sh
#! /bin/sh
set -e

mkdir /tmp/_package
cp -r $PLUGIN_APPLICATION_SOURCE /tmp/_package

mkdir /tmp/_package/DEBIAN
cp $PLUGIN_METAFILE_SOURCE /tmp/_package/DEBIAN/control
echo "Architecture: ${PLUGIN_ARCHITECTURE}" >> /tmp/_package/DEBIAN/control

dpkg-deb --build /tmp/_package

mv /tmp/_package.deb $PLUGIN_DESTINATION
```

The script performs all previous steps with input from some environment variables.

## The `PLUGIN_APPLICATION_SOURCE` Variable

This environment variable contains the path to the binaries included in the package.

## The `PLUGIN_METAFILE_SOURCE` Variable

This environment variable contains the path to the metadata file for the package.
Because of the different `Architecture` attribute for every platform this line must be omitted in the template and inserted on the fly.

## The `PLUGIN_ARCHITECTURE` Variable

This environment variable contains the architecture of the package which gets inserted into the metadata file.

## The `PLUGIN_DESTINATION` Variable

This environment variable contains the full path where the created package should be placed

## Usage

To use the script the following folder structure is assumed:

```
project
|-> package-assets
|---> armv7
|-----> bin
|-------> hello-world
|---> amd64
|-----> bin
|-------> hello-world
|-> package_metadata
```

Subsequently the script can create packages for both platforms:

```console
$ PLUGIN_APPLICATION_SOURCE=./package-assets/armv7 \
  PLUGIN_METAFILE_SOURCE=./package_metadata \
  PLUGIN_ARCHITECTURE=armv7 \
  PLUGIN_DESTINATION=./hello-world_1.0-1_armv7.deb \
  ./pack.sh
$ PLUGIN_APPLICATION_SOURCE=./package-assets/amd64 \
  PLUGIN_METAFILE_SOURCE=./package_metadata \
  PLUGIN_ARCHITECTURE=amd64 \
  PLUGIN_DESTINATION=./hello-world_1.0-1_amd64.deb \
  ./pack.sh
```

## Setup a Package Creation Image

This script can be turned into a Drone plugin by creating a Docker image.
To do this simply create a [`Dockerfile`](https://docs.docker.com/engine/reference/builder/) with the following content:

{{< highlight docker "linenos=table,hl_lines=99" >}}
FROM debian
COPY pack.sh /bin/
RUN chmod +x /bin/pack.sh
ENTRYPOINT /bin/pack.sh
{{< / highlight >}}

Build the image with the name `package-build` to make it available for Docker.

```console
$ docker build -t package-build .
```

{{< alert info >}}
Drone plugins are Docker images that perform pre-defined tasks.
Choose from hundreds of community plugins or create your own.
{{< / alert >}}

### `FROM` Instruction

The `FROM` instruction set the base image for subsequent instructions.
This sets the official Debian image as the base which contains all the essential tools to build a package.

{{< highlight docker "linenos=table,hl_lines=1" >}}
FROM debian
COPY pack.sh /bin/
RUN chmod +x /bin/pack.sh
ENTRYPOINT /bin/pack.sh
{{< / highlight >}}

### `COPY` Instruction

The `COPY` instruction copies a file from the host into the image.

{{< highlight docker "linenos=table,hl_lines=2" >}}
FROM debian
COPY pack.sh /bin/
RUN chmod +x /bin/pack.sh
ENTRYPOINT /bin/pack.sh
{{< / highlight >}}

### `RUN` Instruction

The `RUN` instruction executes a command in the image context.
In this the copied script gets marked as executable.

{{< highlight docker "linenos=table,hl_lines=3" >}}
FROM debian
COPY pack.sh /bin/
RUN chmod +x /bin/pack.sh
ENTRYPOINT /bin/pack.sh
{{< / highlight >}}

### `ENTRYPOINT` Instruction

The `ENTRYPOINT` instruction tells docker what it should execute when the container gets started.

{{< highlight docker "linenos=table,hl_lines=4" >}}
FROM debian
COPY pack.sh /bin/
RUN chmod +x /bin/pack.sh
ENTRYPOINT /bin/pack.sh
{{< / highlight >}}

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
    - mkdir package-assets
    - mkdir -p package-assets/armv7/bin
    - mkdir -p package-assets/arm64/bin
    - mkdir -p package-assets/amd64/bin

- name: build
  image: rust-multiarch
  commands:
    - cargo build
    - cp target/debug/hello-world package-assets/amd64/bin
    - cargo build --target=aarch64-unknown-linux-gnu
    - cp target/aarch64-unknown-linux-gnu/debug/hello-world package-assets/arm64/bin
    - cargo build --target=armv7-unknown-linux-gnueabihf
    - cp target/armv7-unknown-linux-gnueabihf/debug/hello-world package-assets/armv7/bin

- name: create-armv7-package
  image: package-build
  settings:
    application_source: ./package-assets/armv7
    metafile_source: ./package_metadata
    architecture: armv7
    destination: ./assets/hello-world_1.0-1_armv7.deb

- name: create-arm64-package
  image: package-build
  settings:
    application_source: ./package-assets/arm64
    metafile_source: ./package_metadata
    architecture: arm64
    destination: ./assets/hello-world_1.0-1_arm64.deb

- name: create-amd64-package
  image: package-build
  settings:
    application_source: ./package-assets/amd64
    metafile_source: ./package_metadata
    architecture: amd64
    destination: ./assets/hello-world_1.0-1_amd64.deb

- name: publish
  image: plugins/github-release
  settings:
    api_key:
      from_secret: github_token
    files: assets/*
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
    - mkdir package-assets
    - mkdir -p package-assets/armv7/bin
    - mkdir -p package-assets/arm64/bin
    - mkdir -p package-assets/amd64/bin
{{< / highlight >}}

### The `build` Step

In this step the project gets compiled for the `amd64`, `arm64` and `armv7` platform.
After every build the resulting executable is copied into the `package-assets` folder.

{{< highlight yaml "linenos=table,linenostart=15" >}}
- name: build
  image: rust-multiarch
  commands:
    - cargo build
    - cp target/debug/hello-world package-assets/amd64/bin
    - cargo build --target=aarch64-unknown-linux-gnu
    - cp target/aarch64-unknown-linux-gnu/debug/hello-world package-assets/arm64/bin
    - cargo build --target=armv7-unknown-linux-gnueabihf
    - cp target/armv7-unknown-linux-gnueabihf/debug/hello-world package-assets/armv7/bin
{{< / highlight >}}

### The `create-xxx-package` Steps

These steps are used to build the packages for every platform.
The `settings` section contains all the data which is used in environment variables inside the shell script.

{{< highlight yaml "linenos=table,linenostart=25" >}}
- name: create-armv7-package
  image: package-build
  settings:
    application_source: ./package-assets/armv7
    metafile_source: ./package_metadata
    architecture: armv7
    destination: ./assets/hello-world_1.0-1_armv7.deb
{{< / highlight >}}

### The `publish` Step

Here the final packages get published as a GitHub release.
More informations about the configuration of this step can be found in the tutorial {{< ref "publish-github-release-multi-arch.md" >}}.

{{< highlight yaml "linenos=table,linenostart=49" >}}
- name: publish
  image: plugins/github-release
  settings:
    api_key:
      from_secret: github_token
    files: assets/*
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
