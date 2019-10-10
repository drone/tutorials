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

{{<alert "info">}}
The name "octocat" is used as an example, make sure to replace it with your own username where appropriate.
{{</alert>}}

# Prerequisites

- You will need to have [Docker](https://docs.docker.com/install/) installed. *(version 19.03 as of this tutorial)*

{{<alert "info">}}
<a href="https://golang.org">Go</a> 1.11 or higher is also used in the tutorial for convenience, but it can be done without it.
{{</alert>}}

# The Go Application

We will use a basic HTTP hello-world application for our example.

## Writing the application

First create an empty directory that you will throughout the tutorial. You will need to run every command from there.

Then create a new Go [module](https://blog.golang.org/using-go-modules) with:

```
go mod init github.com/octocat/hello-world
```

This will create the necessary `go.mod` file with the following content:
{{<highlight text "linenos=table,hl_lines=99">}}
module github.com/octocat/hello-world

go 1.11
{{</highlight>}}

We will also need the business logic of the application.
For that, create a `main.go` file with the following code:

{{<highlight text "linenos=table,hl_lines=99">}}
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
{{</highlight>}}

When it is run, it will start a HTTP Server that listens on port `8080`, and will simply respond with the `hello world` text for every request. It is enough for us for now just to make sure everything works fine.

## Testing the Application Without Docker

{{<alert "info">}}
Since we will use Docker to build the image, this step can be skipped.
{{</alert>}}

To build and run the application, use

```
go run .
```

It will start the server, then you should be able to visit `http://localhost:8080` in your browser, and see the `hello world` text.


{{<alert "warn">}}
If the port is already in use, the server will panic! You can pick a port other than 8080 if you wish, but make sure to use the same port throughout the tutorial.
{{</alert>}}

# Building with Docker

It is time to build a Docker image with our application in it.

The common practice for Go projects is to separate the build process of the application from the distribution image.
This approach works better with Continuos Integration and Delivery, allowing more caching opportunities.

However, you could also use [multi-stage](https://docs.docker.com/develop/develop-images/multistage-build/) build with a single Dockerfile if you want to .

{{<alert "info">}}
If you are not familiar with Dockerfiles, you can read more about them <a href="https://docs.docker.com/engine/reference/builder/">here</a>.
{{</alert>}}

## Building for ARM64

First we build the application using Docker with the official Go image.

We will need to set three environment variables for the Go compiler:

- `CGO_ENABLED=0`: This one disables [CGo](https://github.com/golang/go/wiki/cgo), it is only needed in our case because we will distribute with an [Alpine Linux](https://alpinelinux.org/) image, and it would run into problems with linking to `libc`.
- `GOOS=linux`: This one sets the target operating system to Linux.
- `GOARCH=arm64`: This one sets the target **ARM64** architecture.

{{<alert "info">}}
You can see the available GOOS and GOARCH combinations <a href="https://golang.org/doc/install/source#environment">here</a>.
{{</alert>}}

We also set the working directory of the image to `/app`, and mount the current directory there using Docker [volumes](https://docs.docker.com/storage/volumes/).

{{<alert "warn">}}
If you use cmd on Windows, you must use "-v %cd%:/app" to mount the current directory.
{{</alert>}}

So our final Docker command will be:

```
docker run \
  -e CGO_ENABLED=0 \
  -e GOOS=linux \
  -e GOARCH=arm64 \
  -w /app \
  -v ${PWD}:/app \
  golang:1.13 go build
```

This will produce a `hello-world` application in our project's directory.

Then we will need to distribute it in a Docker image.
In order to do that, create a `Dockerfile` with the following content:

{{<highlight text "linenos=table,hl_lines=99">}}
# We use the ARM64 version of Alpine.
FROM arm64v8/alpine

# We use the /app working directory for convenience.
WORKDIR /app

# Copy the already built binary file from the first stage.
COPY hello-world /app

# Our application uses the 8080 port
# to communicate, so we need to expose it.
EXPOSE 8080

# Finally we set the entry point of the image,
# so that the hello-world server runs whenever the container is started.
ENTRYPOINT [ "/app/hello-world" ]
{{</highlight>}}

{{<alert "warn">}}
Make sure you use a base image of the correct architecture, ARM64 in this case.
{{</alert>}}

With our `Dockerfile` in place, we can finally build the image with Docker.

{{<alert "info">}}
We also tag the image "hello-world" for convenience.
{{</alert>}}


```
docker build -t hello-world .
```

After the code has run, you should see the `hello-world` image when running `docker image ls`.

On linux, simply use:
```
docker image ls | grep hello-world
```

You should see a similar output:
```
hello-world latest 8fd45a7d5056 2 minutes ago 11.5MB
```

Now you can use the image on your ARM64 devices, and/or publish it to [Docker Hub](https://docs.docker.com/docker-hub/).

{{<alert "info">}}
You can also use the image with Docker Desktop for Windows or Mac even on different host architectures.
On Linux this requires some <a href="https://github.com/hypriot/qemu-register">initial setup</a>.
{{</alert>}}

You can test the image with the following command:

```
docker run -it -p 8080:8080 hello-world
```

Then you should be able to see the hello-world application in your [browser](http://localhost:8080).

## Building for AMD64

Switching target architecture with Go easy. We can use the examples from the [ARM64 build steps]({{< ref "#building-with-docker-for-arm64" >}}) with a few modifications.

We will need to change the `GOARCH` to `amd64`, and then use an AMD64-based image for distribution.

So our build command will look like this:

{{<highlight text "linenos=table,hl_lines=4">}}
docker run \
  -e CGO_ENABLED=0 \
  -e GOOS=linux \
  -e GOARCH=amd64 \
  -w /app \
  -v ${PWD}:/app \
  golang:1.13 go build
{{</highlight>}}

And then our `Dockerfile`:

{{<highlight text "linenos=table,hl_lines=2">}}
# We use the AMD64 version of Alpine. 
FROM amd64/alpine

WORKDIR /app

COPY hello-world /app

EXPOSE 8080

ENTRYPOINT [ "/app/hello-world" ]
{{</highlight>}}

Finally we build it with using:

```
docker build -t hello-world .
```

You can test the image with the following command:

```
docker run -it -p 8080:8080 hello-world
```

Then you should be able to see the hello-world application in your [browser](http://localhost:8080).

# Continuous Integration and Delivery

Even though the steps above are easy, it makes sense to automate them. 

In order to do that, you will need to set up a [Git](https://git-scm.com) repository. For the rest of the tutorial we will assume, that the project is already on [GitHub](https://github.com/).

In this section we configure [Drone](https://drone.io), an Open Source continuous integration system, to automatically build and test our code every time we push to GitHub. You can install Drone on your [own servers](https://docs.drone.io/installation/overview/), or you can use the [free Cloud offering](https://cloud.drone.io/).

## Activation

Fist, login into Drone and activate your repository:

![Activate Repository](/screenshots/drone-repo-activate.png)

## Configuration

Next, define a [Continuous Integration Pipeline](https://docs.drone.io/configure/pipeline/) for your project. Drone looks for special `.drone.yml` file within repositories for the Pipeline definition.

We will achieve the same result we did in [Our Docker build steps]({{< ref "#building-with-docker" >}}), but this time automatically.

First we need to set some basic information for Drone about our pipeline:

- The kind of the `.yml` file. In most cases this will be `pipeline`.
- The type of our pipeline. We want to use Docker containers for each step.
- The name of the pipeline. This can be anything you like.
- We also set the platform we want to execute the pipeline on. This depends on what platform your [Drone Runner](https://docs.drone.io/installation/runners/docker/) is on.

Then translate our steps [from earlier]({{< ref "#building-with-docker-for-arm64" >}}) into Drone Pipeline steps.

First we instruct Drone to build our Go application with the specified environment variables.

Then in an another step Drone will build the image with our `Dockerfile` and then push it to Docker hub using the official [Drone Docker plugin](http://plugins.drone.io/drone-plugins/drone-docker/).

Our final Pipeline will look like this:

{{<highlight text "linenos=table,hl_lines=99">}}
kind: pipeline
type: docker
name: build

steps:
  - name: build
    image: golang:1.13
    environment:
      CGO_ENABLED: "0"
      GOOS: "linux"
      GOARCH: "arm64"
    commands:
      - go build
  
  - name: distribute
    image: plugins/docker
    settings:
      repo: octocat/hello-world
      tags: v1.0.0
      username:
        from_secret: username
      password:
        from_secret: password
{{</highlight>}}

Drone will run the Pipeline each time code is pushed to the repository.

{{<alert "warn">}}
The tag "v1.0.0" is used for demonstration purposes. Make sure to change it to the tag you wish to use.
{{</alert>}}

## Secrets

For security reasons we do not want to store our credentials in our yaml. Instead you can securely store your secrets in the Drone database, adding your docker username and password via the repository settings screen.

![Manage Secret](/screenshots/drone-secrets.png)

## Creating Docker Manifests

Drone has support for [Docker Manifests](https://docs.docker.com/engine/reference/commandline/manifest/) via the [Drone Manifest plugin](http://plugins.drone.io/drone-plugins/drone-manifest/).

This means that we can also push manifests along with our image. To do that, we simply add an another step for our pipeline:

{{<highlight text "linenos=table,hl_lines=25-35">}}
kind: pipeline
type: docker
name: build

steps:
  - name: build
    image: golang:1.13
    environment:
      CGO_ENABLED: "0"
      GOOS: "linux"
      GOARCH: "arm64"
    commands:
      - go build
  
  - name: distribute
    image: plugins/docker
    settings:
      repo: octocat/hello-world
      tags: v1.0.0
      username:
        from_secret: username
      password:
        from_secret: password

  - name: manifest
    image: plugins/manifest
    settings:
      username:
        from_secret: username
      password:
        from_secret: password
      target: octocat/hello-world:v1.0.0
      template: octocat/hello-world:v1.0.0-OS-ARCH
      platforms:
        - linux/arm64
{{</highlight>}}

# Building for Multiple Architectures

The examples so far have only built a single image for a specific architecture. However with Drone we can easily build and publish multiple images targeting multiple platforms at the same time!

In order to showcase that feature we set up an example scenario with the following parameters and requirements:

- We have **two Drone Runners** already set up on different machines, one on **ARM64** and the other on **AMD64** architecture.
- We would like to build our application and images for **both platforms**.
- We want to build a **specific platform** image with its **corresponding runner**. For example, ARM64 images are only built on the ARM64 runner, and not on the other.
- We want to publish a **Docker manifest** along with our images.
- We want to do this **all at once** with a single `.drone.yml`.

{{<alert "info">}}
Building for a specific platform on the corresponding Runner is an artificial constraint for the sake of the tutorial.
<br><br>
The example application written in Go can be built for any architecture on any runner.
{{</alert>}}

We can achieve our goal by reusing what we have already done [earlier]({{< ref "#building-with-docker" >}}) in this tutorial with slight modifications.

First, we will need to create separate `Dockerfiles` for both architectures:

- `Dockerfile.arm64.linux` for our [ARM64]({{< ref "#building-for-arm64" >}}) image
- `Dockerfile.amd64.linux` for our [AMD64]({{< ref "#building-for-amd64" >}}) image

{{<alert "info">}}
You can name the Dockerfiles as you wish, but using a pattern like this you will end up with easily recognizable and descriptive names.
{{</alert>}}

Then we use Drone's [multiple pipeline](https://docs.drone.io/configure/pipeline/multiple/#multi-platform) feature to create and run three pipelines in a single `.drone.yml` file:

- `arm64-pipeline` will build and publish our **ARM64** image on the corresponding runner.
- `amd64-pipeline` will build and publish our **AMD64** image on the corresponding runner.
- `manifest-pipeline` will then publish a Docker manifest for our images. This can run anywhere, but only after the first two.

{{<alert "info">}}
You don't have to put "-pipeline" in the names, we just indicate here that they are well, pipelines.
{{</alert>}}

We will use our [already defined pipelines]({{< ref "#configuration" >}}). We can put all of them in the `.drone.yml`, separated by `---` and Drone will execute each one just fine. 

{{<alert "info">}}
You can read more about this YAML syntax <a href="https://stackoverflow.com/questions/50788277/why-3-dashes-hyphen-in-yaml-file/50788318">here</a>.
{{</alert>}}

Of course we need to modify them slightly, we need to name them differently, and also restrict each pipeline to each runner that has the same operating system and architecture with the `platform` option.

{{<alert "info">}}
The default GOOS and GOARCH values match the host operating system and architecture by default. Since we defined the runner platforms for each pipeline, we can omit them.
<br><br>
For the same reason we could also omit the platform-specific settings in our Dockerfiles, for example we could simply use the "alpine" image instead of "arm64v8/alpine".
{{</alert>}}

We also make sure that our `manifest-pipeline` will only run after our builds have completed. We do this with the `depends_on` setting.

Our three-pipeline `.drone.yml` will look like this:

{{<highlight text "linenos=table,hl_lines= 3 7 20 30 34 47 57 69-71 73-75">}}
kind: pipeline
type: docker
name: arm64-pipeline

platform:
  os: linux
  arch: arm64

steps:
  - name: build
    image: golang:1.13
    environment:
      CGO_ENABLED: "0"
    commands:
      - go build
  
  - name: distribute
    image: plugins/docker
    settings:
      dockerfile: "Dockerfile.arm64.linux"
      repo: octocat/hello-world
      tags: v1.0.0
      username:
        from_secret: username
      password:
        from_secret: password
---
kind: pipeline
type: docker
name: amd64-pipeline

platform:
  os: linux
  arch: amd64

steps:
  - name: build
    image: golang:1.13
    environment:
      CGO_ENABLED: "0"
    commands:
      - go build
  
  - name: distribute
    image: plugins/docker
    settings:
      dockerfile: "Dockerfile.amd64.linux"
      repo: octocat/hello-world
      tags: v1.0.0
      username:
        from_secret: username
      password:
        from_secret: password
---
kind: pipeline
type: docker
name: manifest-pipeline

steps:
  - name: manifest
    image: plugins/manifest
    settings:
      username:
        from_secret: username
      password:
        from_secret: password
      target: octocat/hello-world:v1.0.0
      template: octocat/hello-world:v1.0.0-OS-ARCH
      platforms:
        - linux/arm64
        - linux/amd64

depends_on:
  - amd64-pipeline
  - arm64-pipeline
{{</highlight>}}

This configuration meets all our requirements, and our application will be automatically built and published for different platforms!

# Next Steps

Now that you have Drone automatically building and publishing Docker images you can add more advanced Pipeline steps. For example, you can automatically update your Kubernetes pods to use your new images, or you can send Slack notifications to your team every time your Pipeline completes.
