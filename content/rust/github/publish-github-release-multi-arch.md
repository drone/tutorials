---
date: 2000-01-01T00:00:00+00:00
title: Create multi-Architecture GitHub releases for Rust projects
author: KN4CK3R
draft: true
image: https://dummyimage.com/650x300/253d5f/ffffff
keywords:
- rust
- github
tags:
- rust
- github
- arm
- arm64
- amd64
---

This guide will walk you through the steps required to build multi-architecture Rust projects and publish the binaries on GitHub.
This guide assumes you have a basic understanding of Rust, Docker and GitHub.

{{< alert info >}}
To cross compile Rust projects this tutorial uses the sample Rust project and the Cross-Compilation Docker image presented in the tutorial {{< ref "rust-docker-arm64.md" >}}.
{{< / alert >}}

# Cross Compiling the Rust Project

The goal is to compile the Rust project for three different platforms: `amd64`, `aarch64` and `armv7`.
The Cross-Compilation Docker image from the previous tutorial supports only the current platform (`amd64` in this case) and `aarch64`.
To add support for `armv7` you need to install a compatible linker and the Rust Standard Library.
Add the highlighted lines to the [`Dockerfile`](https://docs.docker.com/engine/reference/builder/) and build it.

{{< highlight docker "linenos=table,hl_lines=5-6 8" >}}
FROM rust:latest
RUN apt-get update \
 && apt-get install -y --no-install-recommends g++-aarch64-linux-gnu libc6-dev-arm64-cross \
 && rustup target add aarch64-unknown-linux-gnu \
 && apt-get install -y --no-install-recommends g++-arm-linux-gnueabihf libc6-dev-armhf-cross \
 && rustup target add armv7-unknown-linux-gnueabihf
ENV CARGO_TARGET_AARCH64_UNKNOWN_LINUX_GNU_LINKER=aarch64-linux-gnu-gcc
ENV CARGO_TARGET_ARMV7_UNKNOWN_LINUX_GNUEABIHF_LINKER=arm-linux-gnueabihf-gcc
{{< / highlight >}}

```console
$ docker build -t rust-multiarch .
```

The usage is identical but with support for `armv7` now:

```console
# Build amd64 (default)
$ docker run --rm -v <path to your project>:/rust-project -w /rust-project rust-multiarch cargo build
# Build aarch64
$ docker run --rm -v <path to your project>:/rust-project -w /rust-project rust-multiarch cargo build --target=aarch64-unknown-linux-gnu
# Build armv7
$ docker run --rm -v <path to your project>:/rust-project -w /rust-project rust-multiarch cargo build --target=armv7-unknown-linux-gnueabihf
```

This creates three compiled binaries:

```console
target/debug/hello-world
target/aarch64-unknown-linux-gnu/debug/hello-world
target/armv7-unknown-linux-gnueabihf/debug/hello-world
```

To organize these assets create a new directory `assets` and copy (and rename) the binaries:

```console
$ mkdir assets
$ cp target/debug/hello-world assets/hello-world-amd64
$ cp target/aarch64-unknown-linux-gnu/debug/hello-world assets/hello-world-aarch64
$ cp target/armv7-unknown-linux-gnueabihf/debug/hello-world assets/hello-world-armv7
```

# Creating a GitHub Release

To create a GitHub release, you first need to [commit your project to GitHub](https://help.github.com/en/articles/creating-a-repository-on-github) and create a git tag for the current version:

```console
$ git tag v1.0.0
$ git push origin v1.0.0
```

On the `Releases` page click the `Draft a new release` button.
This forwards you to a form where you must type in all needed informations.
At the bottom you can attach the created assets.

![Create a new release](/screenshots/rust-github-draft.png)

To finish the process click the green `Publish release` button.
If you have a look at the GitHub release page again you will see the release with all three assets.

![Github Release](/screenshots/rust-github-release.png)

To make your live easier you can automate all of this with Drone in the next section.

# Continuous Integration

This section shows how to configure [Drone](https://drone.io), an Open Source continuous integration system, to automatically build your project for multiple architectures and publish it every time you create a git tag.
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
- name: build
  image: rust-multiarch
  commands:
    - mkdir assets
    - cargo build
    - cp target/debug/hello-world assets/hello-world-amd64
    - cargo build --target=aarch64-unknown-linux-gnu
    - cp target/aarch64-unknown-linux-gnu/debug/hello-world assets/hello-world-aarch64
    - cargo build --target=armv7-unknown-linux-gnueabihf
    - cp target/armv7-unknown-linux-gnueabihf/debug/hello-world assets/hello-world-armv7
- name: publish
  image: plugins/github-release
  settings:
    api_key:
      from_secret: github_token
    files: assets/*
    title: Release Title
  when:
    event: tag
{{< / highlight >}}

The above example defines the pipeline steps as a series of shell commands executed inside Docker containers.
To learn more about how to write the pipeline config see the [official reference](https://docs.drone.io/configure/pipeline/).

### The `image` Attribute

Defines the Docker image in which pipeline commands are executed.
Here the cross compilation image is used.
Remember to push the image to a registry or import it on the machine the Drone runner runs.

{{< highlight yaml "linenos=table,linenostart=5,hl_lines=3" >}}
steps:
- name: build
  image: rust-multiarch
  commands:
    - mkdir assets
    - cargo build
    - cp target/debug/hello-world assets/hello-world-amd64
    - cargo build --target=aarch64-unknown-linux-gnu
    - cp target/aarch64-unknown-linux-gnu/debug/hello-world assets/hello-world-aarch64
    - cargo build --target=armv7-unknown-linux-gnueabihf
    - cp target/armv7-unknown-linux-gnueabihf/debug/hello-world assets/hello-world-armv7
{{< / highlight >}}

### The `commands` Attribute

Defines the Pipeline commands executed inside the Docker container.
The commands to execute are the same you have used to build the project.
All assets are copied into the newly created assets folder.

{{< highlight yaml "linenos=table,linenostart=9,hl_lines=5-11" >}}
steps:
- name: build
  image: rust-multiarch
  commands:
    - mkdir assets
    - cargo build
    - cp target/debug/hello-world assets/hello-world-amd64
    - cargo build --target=aarch64-unknown-linux-gnu
    - cp target/aarch64-unknown-linux-gnu/debug/hello-world assets/hello-world-aarch64
    - cargo build --target=armv7-unknown-linux-gnueabihf
    - cp target/armv7-unknown-linux-gnueabihf/debug/hello-world assets/hello-world-armv7
{{< / highlight >}}

## Execution

The Pipeline will execute every time you push code to your repository.
You can login to Drone and inspect the Pipeline results:

![Build Pipeline](/screenshots/drone-build.png)

# Continuous Delivery

The final step is to automatically build the project and create a GitHub release when you have created a git tag.
In order to use the GitHub API you need to provide your credentials.

## Secrets

For security reasons you do not want to store your credentials in the pipeline configuration file.
Instead you can securely store your secrets in the Drone database, adding your GitHub token via the repository settings screen.

![Manage Secret](/screenshots/github-token-secret.png)

## Settings

Drone has a robust plugin system, including a plugin to create GitHub releases.
Add a new step to the pipeline and configure [the GitHub plugin](http://plugins.drone.io/drone-plugins/drone-github-release/).

{{< highlight yaml "linenos=table,linenostart=12,hl_lines=99" >}}
- name: publish
  image: plugins/github-release
  settings:
    api_key:
      from_secret: github_token
    files: assets/*
    title: Release Title
  when:
    event: tag
{{< / highlight >}}

### The `image` Attribute

Defines the image used to publish the GitHub release.

{{< alert info >}}
Drone plugins are Docker images that perform pre-defined tasks.
Choose from hundreds of community plugins or create your own.
{{< / alert >}}

{{< highlight yaml "linenos=table,linenostart=12,hl_lines=2" >}}
- name: publish
  image: plugins/github-release
  settings:
    api_key:
      from_secret: github_token
    files: assets/*
    title: Release Title
  when:
    event: tag
{{< / highlight >}}

### The `api_key` Attribute

Provides the GitHub API token used to authenticate, sourced from the named secret.

{{< alert info >}}
To use the GitHub API you need [a token for authorization](https://github.com/settings/tokens).
If you don't have one, create it now.
{{< / alert >}}

{{< highlight yaml "linenos=table,linenostart=12,hl_lines=4-5" >}}
- name: publish
  image: plugins/github-release
  settings:
    api_key:
      from_secret: github_token
    files: assets/*
    title: Release Title
  when:
    event: tag
{{< / highlight >}}

### The `files` Attribute

This attribute defines the path(s) to the binaries you wish to upload.
[Glob patterns](https://en.wikipedia.org/wiki/Glob_(programming)) are supported to find files.

{{< highlight yaml "linenos=table,linenostart=12,hl_lines=6" >}}
- name: publish
  image: plugins/github-release
  settings:
    api_key:
      from_secret: github_token
    files: assets/*
    title: Release Title
  when:
    event: tag
{{< / highlight >}}

### The `title` Attribute

Provides the title of the release.
A `note` attribute can be added too which contains the body text.
It is possible to use a simple string or file reference.
For example with a file it is easy to display the whole changelog for this release.

{{< highlight yaml "linenos=table,linenostart=12,hl_lines=7" >}}
- name: publish
  image: plugins/github-release
  settings:
    api_key:
      from_secret: github_token
    files: assets/*
    title: Release Title
  when:
    event: tag
{{< / highlight >}}

### The `event` Attribute

This attribute defines the trigger of this step.
In this case the step should only be executed when the build reason was a git tag commit.

{{< highlight yaml "linenos=table,linenostart=12,hl_lines=8-9" >}}
- name: publish
  image: plugins/github-release
  settings:
    api_key:
      from_secret: github_token
    files: assets/*
    title: Release Title
  when:
    event: tag
{{< / highlight >}}
