---
date: 2000-01-01T00:00:00+00:00
title: Publishing Go Docker Images for Multiple Architectures
author: tamasf97
image: https://dummyimage.com/650x300/253d5f/ffffff
url: publish-golang-docker-images-for-multiple-architectures
featured: true
keywords:
- golang
- docker
tags:
- golang
- docker
- arm
- amd64
---

This tutorial will show you the steps of building a simple [Go](https://golang.org) application and then publishing it in a [Docker](https://www.docker.com/) image for multiple target architectures. The guide will focus on [ARM64](https://en.wikipedia.org/wiki/ARM_architecture#AArch64_features) and [AMD64](https://en.wikipedia.org/wiki/X86-64), but the method is the same for any other supported platform.

# Prerequisites

- You will need to have [Docker](https://docs.docker.com/install/) installed. *(version 19.03 as of this tutorial)*

{{< alert "info" >}}
<a href="https://golang.org">Go</a> 1.11 or higher is also used in the tutorial for convenience, but it can be done without it.
{{< / alert >}}

# The Go Application

We will use a basic HTTP hello-world application for our example.

## Writing the application

First, create the Go [module](https://blog.golang.org/using-go-modules) in an empty directory with 
```
go mod init github.com/octocat/hello-world
```

This will create the necessary `go.mod` file with the following content:
{{< highlight text "linenos=table,hl_lines=99" >}}
module github.com/octocat/hello-world

go 1.11
{{< / highlight >}}

We will also need the business logic of the application.
For that, create a `main.go` file with the following content:

{{< highlight text "linenos=table,hl_lines=99" >}}
package main

import (
    "fmt"
    "net/http"
)

func main() {
    http.HandleFunc("/", handler)
    http.ListenAndServe(":8080", nil)
}

func handler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintln(w, "hello world")
}
{{< / highlight >}}

When it is run, it will start a HTTP Server that listens on port `8080`, and will simply respond with the `hello world` text for every request. It is enough for us for now just to make sure everything works fine.

## Testing the Application Without Docker

{{< alert "info" >}}
Since we will use Docker to build the image, this step can be skipped.
{{< / alert>}}

To build and run the application, use

```
go run .
```

It will start the server, then you should be able to visit `http://localhost:8080` in your browser, and see the `hello world` text.


{{< alert "warn" >}}
If the port is already in use, the server will panic! You can pick a port other than 8080 if you wish, but make sure to use the same port throughout the tutorial.
{{< / alert>}}

# Building with Docker

It is time to build a Docker image with our application in it.

## Building for ARM64

We will create a two-stage `Dockerfile` with a stage that builds our application, then an another stage that will be based on `alpine`, and will run our already built executable. You can read about [multi-stage builds](https://docs.docker.com/develop/develop-images/multistage-build/) if you are not familiar with them.

{{< alert "info" >}}
If you are not familiar with Dockerfiles, you can read more about them <a href="https://docs.docker.com/engine/reference/builder/">here</a>.
{{< / alert >}}


{{< alert "info" >}}
We could omit the second stage, but a Docker image should not be larger than necessary, and we do not need the Go build environment inside the final image when distributing it.
{{< / alert >}}

Create a `Dockerfile` with the following content:

{{< highlight text "linenos=table,hl_lines=99" >}}
# Use the official Go image to build the application with.
FROM golang:1.11 as builder

# Create a working directory.
WORKDIR /hello

# Add the files necessary for building the application.
ADD main.go /hello
ADD go.mod /hello

# Then set the Go build environment variables.

# We need this because if it is omitted,
# the final binary will be linked to libc,
# and will have dependency issues on Alpine.
ENV CGO_ENABLED 0

# This is the target Operating system,
# we use "linux" here because it is the most commonly used,
# but we could also use "windows" (E.g. targetting Windows 10 IoT Core).
ENV GOOS linux

# Our target architecture, ARM64 in this case.
ENV GOARCH arm64

# Build the application,
# this will create a "hello-world" binary.
RUN go build

# Create the second stage, the final image.
# Make sure you use the correct target architecture
# of the base image, ARM64 in this case.
FROM arm64v8/alpine

# We use the same working directory for convenience.
WORKDIR /hello

# Copy the already built binary file from the first stage.
COPY --from=builder /hello/hello-world .

# Our application uses the 8080 port
# to communicate, so we need to expose it.
EXPOSE 8080

# Finally we set the entry point of the image,
# so that the hello-world server runs whenever the container is started.
ENTRYPOINT [ "/hello/hello-world" ]
{{< / highlight >}}

Each step has comments in the `Dockerfile`, make sure you read them, and understand what is happening.

With our `Dockerfile` in place, we can finally build the image with Docker.

{{< alert "info" >}}
We also tag the image "hello-world" for convenience.
{{< / alert >}}


```
docker build -t hello-world .
```

After the code has run, you should see the `hello-world` image when running `docker image ls`.

On linux, simply use:
```
docker image ls | grep hello-world
```

You should a similar output:
```
hello-world latest 8fd45a7d5056 2 minutes ago 11.5MB
```

Now you can use the image on your ARM64 devices, and/or publish it to [Docker Hub](https://docs.docker.com/docker-hub/).

{{< alert "info" >}}
Note that because the architecture of the image is ARM64, you cannot natively run it on AMD64 machines.
{{< / alert >}}

## Building for AMD64

Switching target architecture with Go and Docker is really easy. We can use the examples from the [ARM64 build steps]({{< ref "#building-with-docker-for-arm64" >}}), and then simply change the `arm64` values to `amd64`, or omit it where it is not needed since AMD64 is the most commonly used architecture.

Only two lines need to be changed in the `Dockerfile`:

The target architecture for the Go compiler:
```
# ENV GOARCH arm64
ENV GOARCH amd64
```

And also we need an AMD64-based Alpine image:
```
# FROM arm64v8/alpine
FROM alpine
```

After following the [ARM64 build steps]({{< ref "#building-with-docker-for-arm64" >}}) you can use the image on your AMD64 devices, and/or publish it to [Docker Hub](https://docs.docker.com/docker-hub/).

{{< alert "info" >}}
Note that because the architecture of the image is AMD64, you cannot run it natively on ARM64 machines.
{{< / alert >}}

This time, if you are on an AMD64 computer, you can also test the final image by using
```
docker run -it -p 8080:8080 hello-world
```

Then you should be able to see the hello-world application in your [browser](http://localhost:8080).

# Continuous Integration

Even though the steps above are easy, it makes sense to automate them. 

In order to do that, you will need to set up a [Git](https://git-scm.com) repository. For the rest of the tutorial we will assume, that the project is already on [GitHub](https://github.com/).

In this section we configure [Drone](https://drone.io), an Open Source continuous integration system, to automatically build and test our code every time we push to GitHub. You can install Drone on your own servers, or you can use the free Cloud offering.

## Activation

Fist, login into Drone and activate your repository:

![Activate Repository](/screenshots/drone-repo-activate.png)

## Configuration

Next, define a Continuous Integration Pipeline for your project. Drone looks for special `.drone.yml` file within repositories for the Pipeline definition:

{{< highlight text "linenos=table,hl_lines=99" >}}
kind: pipeline
type: docker
name: build

platform:
  os: linux
  arch: arm64

steps:
- name: test
  image: golang:1.11
  environment:
    CGO_ENABLED: "0"
  commands:
  - go test ./...
{{< / highlight >}}

In the above example we define our Pipeline steps as a series of shell commands executed inside Docker containers.

### The `platform` Section

Defines the target operating system and environment.

{{< highlight text "linenos=table,hl_lines=5-7" >}}
kind: pipeline
type: docker
name: build

platform:
  os: linux
  arch: arm64
{{< / highlight >}}


### The `name` Attribute

Defines the name of the Pipeline step.

{{< highlight text "linenos=table,linenostart=9,hl_lines=2" >}}
steps:
- name: test
  image: golang:1.11
  environment:
    CGO_ENABLED: "0"
  commands:
  - go test -v ./...
{{< / highlight >}}

### The `image` Attribute

Defines the Docker image in which Pipeline commands are executed.

{{< highlight text "linenos=table,linenostart=9,hl_lines=3" >}}
steps:
- name: test
  image: golang:1.11
  environment:
    CGO_ENABLED: "0"
  commands:
  - go test -v ./...
{{< / highlight >}}

### The `commands` Attribute

Defines the Pipeline commands executed inside the Docker container.

{{< highlight text "linenos=table,linenostart=9,hl_lines=7-8" >}}
steps:
- name: test
  image: golang:1.11
  environment:
    CGO_ENABLED: "0"
  commands:
  - go test -v ./...
{{< / highlight >}}

## Execution

The Pipeline will execute every time we push code to our repository. We can login to Drone and inspect the Pipeline results:

![Activate Repository](/screenshots/drone-build.png)

# Continuous Delivery

The final step is to automatically build and publish our Docker image to DockerHub. In order to publish and image to DockerHub we need to provide our credentials.

## Secrets

For security reasons we do not want to store our credentials in our yaml. Instead you can securely store your secrets in the Drone database, adding your docker username and password via the repository settings screen.

![Manage Secret](/screenshots/drone-secrets.png)

## Settings

Drone has a robust plugin system, including a plugin to build and publish images. Add a new step to the Docker Pipeline and configure the Docker plugin.

{{< highlight text "linenos=table,linenostart=17,hl_lines=99" >}}
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
Drone plugins are Docker images that perform pre-defined tasks. Choose from hundreds of community plugins or create your own.
{{< / alert >}}

{{< highlight text "linenos=table,linenostart=17,hl_lines=2" >}}
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

{{< highlight text "linenos=table,linenostart=17,hl_lines=6-7" >}}
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

{{< highlight text "linenos=table,linenostart=17,hl_lines=8-9" >}}
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

{{< highlight text "linenos=table,linenostart=17,hl_lines=4" >}}
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

Now that you have Drone automatically building and publishing Docker images you can add more advanced Pipeline steps. For example, you can automatically update your Kubernetes pods to use your new images, or you can send Slack notifications to your team every time your Pipeline completes.
