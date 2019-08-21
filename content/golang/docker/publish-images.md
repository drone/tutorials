---
date: 2000-01-01T00:00:00+00:00
title: Publishing Go Docker Images on Linux
author: bradrydzewski
image: https://community-cdn-digitalocean-com.global.ssl.fastly.net/assets/tutorials/images/large/go_image.png?1554493581
url: publish-golang-docker-images
keywords:
- golang
- docker
tags:
- golang
- docker
---

This guide will walk you through the steps required to build and publish Docker images for Go projects. This guide assumes you have a basic understanding of Go and Docker, and that you have Go and Docker installed.

# Setup a Go Project

Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam, quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo consequat.

First we setup a `go.mod` file for our project. The module file defines the project namespace and dependencies:

{{< highlight text "linenos=table,hl_lines=99" >}}
module github.com/octocat/hello-world

require (

)
{{< / highlight >}}

Next we create a simple Go program. For the purpose of this guide we create a simple http server that responds to an http request with `hello world`.

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



```
$ go build
$ ./hello-world
```



```
$ curl http://localhost:8080
hello world
```

# Setup a Dockerfile

{{< highlight text "linenos=table,hl_lines=99" >}}
FROM arm64v8/alpine
EXPOSE 80
ADD hello-world /bin
ENTRYPOINT [ /bin/hello-world ]
{{< / highlight >}}

# Build and Publish the Image


```
$ CGO_ENABLED=0 GOARCH=amd64 go build
$ docker build -t hello-world .
$ docker push hello-world
```



Duis aute irure dolor in reprehenderit in voluptate velit esse cillum dolore eu fugiat nulla pariatur. Excepteur sint occaecat cupidatat non proident, sunt in culpa qui officia deserunt mollit anim id est laborum.

# Continuous Integration

Once our project is ready and is pushed to GitHub we can automate the build and test processing using continuous integration.

In this section we configure [Drone](https://drone.io), an Open Source continuous integration system, to automatically build and test our code every time we push to GitHub. You can install Drone on your own servers, or you can use the free Cloud offering.

Fist, login into Drone and activate your repository:

__IMAGE HERE__

Next, define a continuous integration pipeline for your project. Drone looks for special `.drone.yml` file within repositories for the pipeline definition:

{{< highlight text "linenos=table,hl_lines=99" >}}
kind: pipeline
type: docker
name: build

platform:
  os: linux
  arch: arm64

steps:
- name: test
  image: golang:1.12
  environment:
    CGO_ENABLED: "0"
  commands:
  - go test ./...
  - go build
{{< / highlight >}}

In the above example we define our pipeline steps as a series of shell commands executed inside Docker containers.

{{< highlight text "linenos=table,linenostart=9,hl_lines=3 7-8" >}}
steps:
- name: test
  image: golang:1.12
  environment:
    CGO_ENABLED: "0"
  commands:
  - go test -v ./...
  - go build
{{< / highlight >}}

In the above example we configure the `go build` and `go test` commands to execute inside a `golang:1.12` container. This container is isolated from the host machine and is destroyed when the pipeline completes. 

The pipeline will execute every time we push code to our repository. We can login to Drone and inspect the pipeline results:

__IMAGE HERE__

# Continuous Delivery

Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam, quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo consequat.




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

Duis aute irure dolor in reprehenderit in voluptate velit esse cillum dolore eu fugiat nulla pariatur. Excepteur sint occaecat cupidatat non proident, sunt in culpa qui officia deserunt mollit anim id est laborum.
