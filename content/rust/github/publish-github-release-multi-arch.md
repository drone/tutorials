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
- githug
- arm
- arm64
- amd64
---

This guide will walk you through the steps required to build multi-architecture Rust projects and publish the binaries on GitHub.
This guide assumes you have a basic understanding of Rust, Docker and GitHub.

{{< alert info >}}
To cross compile Rust projects this tutorial uses the sample Rust project and the Cross-Compilation Docker image presented in the tutorial [Publishing Rust Docker Images for ARM](...).
{{< / alert >}}

# Cross Compiling the Rust Project

The goal is to compile the Rust project for three different platforms: `amd64`, `aarch64` and `armv7`.
The Cross-Compilation Docker image from the previous tutorial supports only the current platform (`amd64` in this case) and `aarch64`.
To add support for `armv7` you need to install a compatible linker and the Rust Standard Library.
Add the highlighted lines to the `dockerfile` and build it.

{{< highlight text "linenos=table,hl_lines=5-6 8" >}}
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

To create a GitHub release, you first need to [commit our project to GitHub](https://help.github.com/en/articles/creating-a-repository-on-github) and create a git tag for the current version:

```console
$ git tag v1.0.0
$ git push origin v1.0.0
```

To create a release from this tag you could use the GitHub user interface or the GitHub API.
This tutorial shows the usage of the API with `curl`.

{{< alert info >}}
To use the GitHub API you need [a token for authorization](https://github.com/settings/tokens).
If you don't have one, create it now.
{{< / alert >}}

First you need to [create a release](https://developer.github.com/v3/repos/releases/#create-a-release) for the tag:

```console
$ curl -X POST \
  -H "Content-Type:application/json" \
  -H "Authorization: token <your api token>" \
  https://api.github.com/repos/<your GitHub username>/<the repository name>/releases \
  -d '{"tag_name":"<name of the tag>", "name":"<title of this release>", "body":"<description of this release>"}'
```

This performs as `POST` request to the GitHub API.
You need to provide three informations for this to work: a token for authorization, your username on GitHub with the name of the repository as parts of the url.
The last line provides the content for the release.
The `tag_name` must match the name of the tag created the step above.
`name` and `body` specify the title and the description of the release.

```console
$ curl -X POST -H "Content-Type:application/json" -H "Authorization: token f856c80e5929250b646b8ab30b8b065ab22fxxxx" https://api.github.com/repos/KN4CK3R/HelloWorld/releases -d '{"tag_name":"v1.0.0", "name":"Hello World v1.0.0", "body":"The description of this release."}'
{
  "url": "https://api.github.com/repos/KN4CK3R/HelloWorld/releases/20394456",
  "assets_url": "https://api.github.com/repos/KN4CK3R/HelloWorld/releases/20394456/assets",
  "upload_url": "https://uploads.github.com/repos/KN4CK3R/HelloWorld/releases/20394456/assets{?name,label}",
  "html_url": "https://github.com/KN4CK3R/HelloWorld/releases/tag/v1.0.0",
  "id": 20394456,
  "node_id": "MDc6UmVsZWFzZTIwMzk0NDU2",
  "tag_name": "v1.0.0",
  "target_commitish": "master",
  "name": "Hello World v1.0.0",
  "draft": false,
  "author": {
    "login": "KN4CK3R",
    "id": 1666336,
    "node_id": "MDQ6VXNlcjE2NjYzMzY=",
    "avatar_url": "https://avatars2.githubusercontent.com/u/1666336?v=4",
    "gravatar_id": "",
    "url": "https://api.github.com/users/KN4CK3R",
    "html_url": "https://github.com/KN4CK3R",
    "followers_url": "https://api.github.com/users/KN4CK3R/followers",
    "following_url": "https://api.github.com/users/KN4CK3R/following{/other_user}",
    "gists_url": "https://api.github.com/users/KN4CK3R/gists{/gist_id}",
    "starred_url": "https://api.github.com/users/KN4CK3R/starred{/owner}{/repo}",
    "subscriptions_url": "https://api.github.com/users/KN4CK3R/subscriptions",
    "organizations_url": "https://api.github.com/users/KN4CK3R/orgs",
    "repos_url": "https://api.github.com/users/KN4CK3R/repos",
    "events_url": "https://api.github.com/users/KN4CK3R/events{/privacy}",
    "received_events_url": "https://api.github.com/users/KN4CK3R/received_events",
    "type": "User",
    "site_admin": false
  },
  "prerelease": false,
  "created_at": "2019-10-01T11:15:27Z",
  "published_at": "2019-10-01T19:05:41Z",
  "assets": [

  ],
  "tarball_url": "https://api.github.com/repos/KN4CK3R/HelloWorld/tarball/v1.0.0",
  "zipball_url": "https://api.github.com/repos/KN4CK3R/HelloWorld/zipball/v1.0.0",
  "body": "The description of this release."
}
```

You must use the `upload_url` to upload the three assets to the newly created release:

```console
$ curl \
  -H "Authorization: token <your api token>" \
  -H "Content-Type: <mime type>" \
  --data-binary @<path to file> \
  <the upload url>?name=<file name>
```

The `Content-Type` header must contain the mime type of the assets.
(Hint: You can get the mime type with `file -b --mime-type /path/to/file`).
The url to call is the `upload_url` you got as response with an additional `name` parameter for the filename.

```console
$ curl -H "Authorization: token f856c80e5929250b646b8ab30b8b065ab22fxxxx" -H "Content-Type: application/x-pie-executable" --data-binary @assets/hello-world-aarch64 https://uploads.github.com/repos/KN4CK3R/HelloWorld/releases/20394456/assets?name=hello-world-aarch64
{
  "url":"https://api.github.com/repos/KN4CK3R/HelloWorld/releases/assets/15236291",
  "id":15236291,
  "node_id":"MDEyOlJlbGVhc2VBc3NldDE1MjM2Mjkx",
  "name":"hello-world-aarch64",
  "label":"",
  "uploader":{
    "login":"KN4CK3R",
    "id":1666336,
    "node_id":"MDQ6VXNlcjE2NjYzMzY=",
    "avatar_url":"https://avatars2.githubusercontent.com/u/1666336?v=4",
    "gravatar_id":"",
    "url":"https://api.github.com/users/KN4CK3R",
    "html_url":"https://github.com/KN4CK3R",
    "followers_url":"https://api.github.com/users/KN4CK3R/followers",
    "following_url":"https://api.github.com/users/KN4CK3R/following{/other_user}",
    "gists_url":"https://api.github.com/users/KN4CK3R/gists{/gist_id}",
    "starred_url":"https://api.github.com/users/KN4CK3R/starred{/owner}{/repo}",
    "subscriptions_url":"https://api.github.com/users/KN4CK3R/subscriptions",
    "organizations_url":"https://api.github.com/users/KN4CK3R/orgs",
    "repos_url":"https://api.github.com/users/KN4CK3R/repos",
    "events_url":"https://api.github.com/users/KN4CK3R/events{/privacy}",
    "received_events_url":"https://api.github.com/users/KN4CK3R/received_events",
    "type":"User",
    "site_admin":false
  },
  "content_type":"application/x-pie-executable",
  "state":"uploaded",
  "size":2583728,
  "download_count":0,
  "created_at":"2019-10-01T19:11:04Z",
  "updated_at":"2019-10-01T19:11:11Z",
  "browser_download_url":"https://github.com/KN4CK3R/HelloWorld/releases/download/v1.0.0/hello-world-aarch64"
}
$ curl -H "Authorization: token f856c80e5929250b646b8ab30b8b065ab22fxxxx" -H "Content-Type: application/x-pie-executable" --data-binary @assets/hello-world-armv7 https://uploads.github.com/repos/KN4CK3R/HelloWorld/releases/20394456/assets?name=hello-world-armv7
$ curl -H "Authorization: token f856c80e5929250b646b8ab30b8b065ab22fxxxx" -H "Content-Type: application/x-pie-executable" --data-binary @assets/hello-world-amd64 https://uploads.github.com/repos/KN4CK3R/HelloWorld/releases/20394456/assets?name=hello-world-amd64
```

If you have a look at the GitHub release page you will see the release with all three assets.

![Activate Repository](/screenshots/rust-github-release.jpg)

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

{{< highlight text "linenos=table,hl_lines=99" >}}
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

{{< highlight text "linenos=table,linenostart=5,hl_lines=3" >}}
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

{{< highlight text "linenos=table,linenostart=9,hl_lines=5-11" >}}
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

![Manage Secret](/screenshots/github-token-secret.jpg)

## Settings

Drone has a robust plugin system, including a plugin to create GitHub releases.
Add a new step to the pipeline and configure [the GitHub plugin](http://plugins.drone.io/drone-plugins/drone-github-release/).

{{< highlight text "linenos=table,linenostart=12,hl_lines=99" >}}
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

{{< highlight text "linenos=table,linenostart=12,hl_lines=2" >}}
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

{{< highlight text "linenos=table,linenostart=12,hl_lines=4-5" >}}
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

This attribute defines the path to the binaries you wish to upload.

{{< highlight text "linenos=table,linenostart=12,hl_lines=6" >}}
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

{{< highlight text "linenos=table,linenostart=12,hl_lines=7" >}}
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

{{< highlight text "linenos=table,linenostart=12,hl_lines=8-9" >}}
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
