---
date: 2000-01-01T00:00:00+00:00
title: Publishing Rust Docker Images for ARM
author: KN4CK3R
draft: true
image: https://dummyimage.com/650x300/253d5f/ffffff
keywords:
- rust
- docker
tags:
- rust
- docker
- arm
---

This guide will walk you through the steps required to build and publish arm64 Docker images for Rust projects.
This guide assumes you have a basic understanding of Rust and Docker, and that you have Rust and Docker installed.

# Setup a Rust Project

To setup an example project this you can use the Rust package manager [`cargo`](https://doc.rust-lang.org/cargo/index.html):

```console
$ cargo init hello-world
```

This creates a folder named `hello-world` which contains the project.
It consists of a project file `Cargo.toml` and a source file `main.rs` in the `src` subdirectory.
The source file contains code which simply prints `Hello, world!` in the console:

{{< highlight text "linenos=table,hl_lines=99" >}}
fn main() {
    println!("Hello, world!");
}
{{< / highlight >}}

Use [`cargo`](https://doc.rust-lang.org/cargo/index.html) to compile the sourcecode to a standalone binary file, and then execute the application to test it:

```console
$ cargo build
$ ./target/debug/hello-world
```

The project should build without problems and display the message `Hello, world!` when run.

# Cross Compiling a Rust Project

The goal is now to build this project for the ARM platform.
Rust can build executables for many platforms.
To find out which platforms are available use the following command:

```console
$ rustc --print target-list
```

You can find more informations about the supported platforms [on the Rust website](https://forge.rust-lang.org/release/platform-support.html).
This tutorial uses `aarch64-unknown-linux-gnu` to compile the project for the ARM64 architecture.

To compile a project for a different platform two things are required: the Rust Standard Library for this platform and a compatible linker.
The standard library can be obtained with the [`rustup`](https://rustup.rs/) tool:

```console
$ rustup target add aarch64-unknown-linux-gnu
```

It is a little bit harder to find a compatible linker to use.
For `aarch64-unknown-linux-gnu` we have to install the `g++-aarch64-linux-gnu libc6-dev-arm64-cross` packages.

```console
$ apt-get install g++-aarch64-linux-gnu libc6-dev-arm64-cross
```

To tell [`cargo`](https://doc.rust-lang.org/cargo/index.html) to build for a specific platform you must pass the platform as a `--target` parameter and set the linker to use in an environment variable.
The variable name has the format `CARGO_TARGET_<plaform>_LINKER` where `<platform>` is the platform in UPPERCASE letters.

```console
$ export CARGO_TARGET_AARCH64_UNKNOWN_LINUX_GNU_LINKER=aarch64-linux-gnu-gcc
$ cargo build --target=aarch64-unknown-linux-gnu
# or
$ CARGO_TARGET_AARCH64_UNKNOWN_LINUX_GNU_LINKER=aarch64-linux-gnu-gcc cargo build --target=aarch64-unknown-linux-gnu
```

The build process creates a new folder `aarch64-unknown-linux-gnu` in the `target` folder which contains the compiled binary application.
If run it exits with an error message because the current system has not the correct platform:

```console
hello-world: cannot execute binary file: Exec format error
```

To simplify this setup process we can use the project [cross](https://github.com/rust-embedded/cross).
`cross` comes with a set of Docker images prepared for cross compilation.

```console
$ cargo install cross
$ cross build --target aarch64-unknown-linux-gnu
```

{{< alert info >}}
This does all the manual work but sadly at the time of writing `cross` can't be used in a Docker-in-Docker environment (see this [issue](https://github.com/rust-embedded/cross/issues/260)).
So currently it can't be used with Drone or any other Docker based CI/CD tool.
You can work around this problem by creating your own Docker image for cross compilation.
{{< / alert >}}

# Setup the Cross-Compilation Dockerfile

The Cross-Compilation [`dockerfile`](https://docs.docker.com/engine/reference/builder/) should contain a ready to use Rust Standard Library with a plattform specific linker to compile the project.
This image can later be used with Drone to automate the build process.
Create the following `dockerfile`:

{{< highlight docker "linenos=table,hl_lines=99" >}}
FROM rust:latest
RUN apt-get update \
 && apt-get install -y --no-install-recommends g++-aarch64-linux-gnu libc6-dev-arm64-cross \
 && rustup target add aarch64-unknown-linux-gnu
ENV CARGO_TARGET_AARCH64_UNKNOWN_LINUX_GNU_LINKER=aarch64-linux-gnu-gcc
{{< / highlight >}}

## `FROM` Instruction

The `FROM` instruction set the base image for subsequent instructions.
This sets the official Rust image as the base so you don't need to install the complete rust toolchain but only the cross compilation features.

{{< highlight docker "linenos=table,hl_lines=1" >}}
FROM rust:latest
RUN apt-get update \
 && apt-get install -y --no-install-recommends g++-aarch64-linux-gnu libc6-dev-arm64-cross \
 && rustup target add aarch64-unknown-linux-gnu
ENV CARGO_TARGET_AARCH64_UNKNOWN_LINUX_GNU_LINKER=aarch64-linux-gnu-gcc
{{< / highlight >}}

## `RUN` Instruction

The `RUN` instruction executes a command in the image context.
In this case it installs the linker you need to compile for the `aarch64` platform and instructs rustup to add the Rust Standard Library.

{{< highlight docker "linenos=table,hl_lines=2-4" >}}
FROM rust:latest
RUN apt-get update \
 && apt-get install -y --no-install-recommends g++-aarch64-linux-gnu libc6-dev-arm64-cross \
 && rustup target add aarch64-unknown-linux-gnu
ENV CARGO_TARGET_AARCH64_UNKNOWN_LINUX_GNU_LINKER=aarch64-linux-gnu-gcc
{{< / highlight >}}

## `ENV` Instruction

The `ENV` instruction sets an environment variable.
Here you specify which linker `cargo` should use.

{{< highlight docker "linenos=table,hl_lines=5" >}}
FROM rust:latest
RUN apt-get update \
 && apt-get install -y --no-install-recommends g++-aarch64-linux-gnu libc6-dev-arm64-cross \
 && rustup target add aarch64-unknown-linux-gnu
ENV CARGO_TARGET_AARCH64_UNKNOWN_LINUX_GNU_LINKER=aarch64-linux-gnu-gcc
{{< / highlight >}}

# Build and Use the Cross-Compilation Dockerfile

To build the `dockerfile` simply execute the command:

```console
$ docker build -t rust-aarch64 .
```

To compile a project using this image you instruct Docker to run a new container with the project mapped into it and execute the build with `cargo`.

```console
$ docker run --rm -v <path to your project>:/rust-project -w /rust-project rust-aarch64 cargo build --target=aarch64-unknown-linux-gnu
```

`--rm` tells Docker to clean up the container after exit.
`-v` mounts the local project folder to `/rust-project` in the container.
`-w` sets the current working directory.
The image to use is the previous `rust-aarch64` image.
`cargo build --target=aarch64-unknown-linux-gnu` executes the build for the specific platform.

# Setup the Application Dockerfile

Create a [`dockerfile`](https://docs.docker.com/engine/reference/builder/) for your project:

{{< highlight docker "linenos=table,hl_lines=99" >}}
FROM arm64v8/ubuntu
ADD target/aarch64-unknown-linux-gnu/debug/hello-world /bin
ENTRYPOINT [ "/bin/hello-world" ]
{{< / highlight >}}

## `ADD` Instruction

The `ADD` instruction copies the binary to the target directory in the image.
This adds the `hello-world` binary to the image.

{{< highlight docker "linenos=table,hl_lines=2" >}}
FROM arm64v8/ubuntu
ADD target/aarch64-unknown-linux-gnu/debug/hello-world /bin
ENTRYPOINT [ "/bin/hello-world" ]
{{< / highlight >}}

## `ENTRYPOINT` Instruction

The `ENTRYPOINT` instruction configures the default command that is executed when the container starts.
This is configured to execute the `hello-world` binary.

{{< highlight docker "linenos=table,hl_lines=3" >}}
FROM arm64v8/ubuntu
ADD target/aarch64-unknown-linux-gnu/debug/hello-world /bin
ENTRYPOINT [ "/bin/hello-world" ]
{{< / highlight >}}

# Build and Publish the Application Image

First you need to build the executable with the `rust-aarch64` image:

```console
$ docker run --rm -v <path to your project>:/rust-project -w /rust-project rust-aarch64 cargo build --target=aarch64-unknown-linux-gnu
```

Then build the Application Docker image:

```console
$ docker build -t hello-world .

Sending build context to Docker daemon  5.755MB
Step 1/3 : FROM arm64v8/ubuntu
 ---> 915beeae4675
Step 2/3 : ADD target/aarch64-unknown-linux-gnu/debug/hello-world /bin
 ---> 922062d00967
Step 3/3 : ENTRYPOINT [ "/bin/hello-world" ]
 ---> 07cacde6e87c
Successfully built 07cacde6e87c
Successfully tagged hello-world:latest
```

Create and run a Docker container using the new image.

```console
$ docker run --rm hello-world
Hello, world!
```

{{< alert info >}}
Docker Desktop uses qemu-static to run containers for different Linux architectures such as arm, mips, ppc64le and s390x.
{{< / alert >}}

# Continuous Integration

Once the project is ready and pushed to GitHub you can automate the build and test processing using continuous integration.

In this section you learn how to configure [Drone](https://drone.io), an Open Source continuous integration system, to automatically build and test your code every time we push to GitHub.
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
  image: rust-aarch64
  commands:
    - cargo build --target=aarch64-unknown-linux-gnu
{{< / highlight >}}

The above example defines the pipeline steps as a series of shell commands executed inside Docker containers.
To learn more about how to write the pipeline config see the [official reference](https://docs.drone.io/configure/pipeline/).

### The `image` Attribute

Defines the Docker image in which pipeline commands are executed.
Here the cross compile image is used.
Remember to push the image to a registry or import it on the machine the Drone runner runs.

{{< highlight yaml "linenos=table,linenostart=5,hl_lines=3" >}}
steps:
- name: build
  image: rust-aarch64
  commands:
    - cargo build --target=aarch64-unknown-linux-gnu
{{< / highlight >}}

### The `commands` Attribute

Defines the pipeline commands executed inside the Docker container.
The command to execute is the same you have used to build the project.

{{< highlight yaml "linenos=table,linenostart=9,hl_lines=5" >}}
steps:
- name: build
  image: rust-aarch64
  commands:
    - cargo build --target=aarch64-unknown-linux-gnu
{{< / highlight >}}

## Execution

The pipeline will execute every time you push code to your repository.
You can login to Drone and inspect the Pipeline results:

![Activate Repository](/screenshots/drone-build.png)

# Continuous Delivery

The final step is to automatically build and publish the Docker image to DockerHub.
In order to publish an image to DockerHub you need to provide your credentials.

## Secrets

For security reasons you do not want to store your credentials in the pipeline configuration file.
Instead you can securely store your secrets in the Drone database, adding your Docker username and password via the repository settings screen.

![Manage Secret](/screenshots/drone-secrets.png)

## Settings

Drone has a robust plugin system, including a plugin to build and publish images.
Add a new step to the Docker Pipeline and configure the [Docker plugin](http://plugins.drone.io/drone-plugins/drone-docker/).

{{< highlight yaml "linenos=table,linenostart=17,hl_lines=99" >}}
- name: publish
  image: plugins/docker
  settings:
    repo: hello-world
    tags: latest
    username:
      from_secret: username
    password:
      from_secret: password
{{< / highlight >}}

### The `image` Attribute

Defines the image used to build and publish the Docker image.

{{< alert info >}}
Drone plugins are Docker images that perform pre-defined tasks.
Choose from hundreds of community plugins or create your own.
{{< / alert >}}

{{< highlight yaml "linenos=table,linenostart=17,hl_lines=2" >}}
- name: publish
  image: plugins/docker
  settings:
    repo: hello-world
    tags: latest
    username:
      from_secret: username
    password:
      from_secret: password
{{< / highlight >}}

### The `username` Attribute

Provides the username used to authenticate with the Docker registry, source from the named secret.

{{< highlight yaml "linenos=table,linenostart=17,hl_lines=6-7" >}}
- name: publish
  image: plugins/docker
  settings:
    repo: hello-world
    tags: latest
    username:
      from_secret: username
    password:
      from_secret: password
{{< / highlight >}}

### The `password` Attribute

Provides the password used to authenticate with the Docker registry, sourced from the named secret.

{{< highlight yaml "linenos=table,linenostart=17,hl_lines=8-9" >}}
- name: publish
  image: plugins/docker
  settings:
    repo: hello-world
    tags: latest
    username:
      from_secret: username
    password:
      from_secret: password
{{< / highlight >}}

### The `repo` Attribute

Provides the Docker repository name.

{{< highlight yaml "linenos=table,linenostart=17,hl_lines=4" >}}
- name: publish
  image: plugins/docker
  settings:
    repo: hello-world
    tags: latest
    username:
      from_secret: username
    password:
      from_secret: password
{{< / highlight >}}

# Next Steps

Now that you have Drone automatically building and publishing Docker images you can add more advanced Pipeline steps.
For example, you can automatically update your Kubernetes pods to use your new images, or you can send Slack notifications to your team every time your Pipeline completes.
