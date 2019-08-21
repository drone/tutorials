---
date: 2000-01-01T00:00:00+00:00
title: Publishing Go Docker Images on ARM
author: bradrydzewski
image: https://community-cdn-digitalocean-com.global.ssl.fastly.net/assets/tutorials/images/large/go_image.png?1554493581
keywords:
- golang
- docker
tags:
- golang
- docker
- arm
---

Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam, quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo consequat.

# First Step

Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam, quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo consequat.

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

```
$ CGO_ENABLED=0 GOARCH=amd64 go build
$ docker build -t hello-world .
$ docker push hello-world
```

{{< highlight text "linenos=table,hl_lines=99" >}}
FROM arm64v8/alpine
EXPOSE 80
ADD hello-world /bin
ENTRYPOINT [ /bin/hello-world ]
{{< / highlight >}}

Duis aute irure dolor in reprehenderit in voluptate velit esse cillum dolore eu fugiat nulla pariatur. Excepteur sint occaecat cupidatat non proident, sunt in culpa qui officia deserunt mollit anim id est laborum.

# Second Step

Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam, quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo consequat.

{{< highlight text "linenos=table,hl_lines=99" >}}
kind: pipeline
type: docker
name: build

platform:
  os: linux
  arch: arm64

steps:
- name: test
  image: golang
  environment:
    CGO_ENABLED: "0"
  commands:
  - go test ./...
  - go build
{{< / highlight >}}


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

# Third Step

Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam, quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo consequat.

Duis aute irure dolor in reprehenderit in voluptate velit esse cillum dolore eu fugiat nulla pariatur. Excepteur sint occaecat cupidatat non proident, sunt in culpa qui officia deserunt mollit anim id est laborum.

# Fourth Step

Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam, quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo consequat.

Duis aute irure dolor in reprehenderit in voluptate velit esse cillum dolore eu fugiat nulla pariatur. Excepteur sint occaecat cupidatat non proident, sunt in culpa qui officia deserunt mollit anim id est laborum.
