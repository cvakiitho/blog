---
layout: post
title: "Go relative imports, and building"
categories: [programming, go]
tags: [ programming, go, docker]
---

This post was only made because, because docker docs example wasn't working for me:

[https://docs.docker.com/develop/develop-images/multistage-build/](https://docs.docker.com/develop/develop-images/multistage-build/)

It is reasonable to use multi-stage build for containers with go binaries, because image to build go is pretty large:

```
REPOSITORY               SIZE
go-scratch-binary        6.24MB
go-alpine-binary         14.5MB
go-builder               807MB
```

"Problem" is, our go microservice is living inside corporate github, and it's packages can't be accessed from golang image without installing certificates, or adding our microservice into GOPATH, without these, you will end up with `cannot find module providing package <corp.github.url>/package` even though the package is in the same folder.


## Modules to the rescue

Fortunately Go 1.11 introduced [Modules](https://github.com/golang/go/wiki/Modules)

with modules, relative imports inside go source code works like a charm: 

start with `go mod init <package_name>` inside folder with your go main.go

```
├── main.go
├── controllers 
│   ├── some_controller.go
│   ├── something.go
```


this creates `go.mod`, and `go.sum` file:

go.mod
```
module <package_name>

  go 1.12

  require (
          github....
```


main.go
```
package main

import (
  "<package>/controllers"
)
```


controllers/some_controller
```
package controllers

import (
  ...
)
```

with this structure, you can build your go module anywhere with something like this:

Dockerfile
```
FROM golang:alpine

# Go mod needs git
RUN apk update && apk add --no-cache git

RUN mkdir /app
ADD . /app/
WORKDIR /app
RUN go build
CMD ["/app/<package>"]
```

`docker build . -t go-build:latest && docker run -it go-build` => running binary

```
2019/07/17 08:54:47 @@@Hello world!!!@@@
2019/07/17 08:54:47 Listening...
```

But this results in ~300MB image =>

## Docker multistage build to the rescue


Dockerfile
```
FROM golang:alpine AS builder

# Go mod needs git
RUN apk update && apk add --no-cache git

RUN mkdir /app
ADD . /app/
WORKDIR /app
RUN go build
CMD ["/app/<package>"]

FROM alpine:latest AS alpine
COPY --from=builder /app/<package> /app/<package>
CMD ["/app/<package>"]
```

`docker build --target=alpine -t go-alpine:latest`

=> ~10 MB image(depends on Go dependencies)

We can go further, and build statically linked binary, and use scratch docker image:

*if you try to use standard binary with scratch image you'll end with*

`standard_init_linux.go:207: exec user process caused "no such file or directory"`

Dockerfile
```
FROM golang:alpine AS builder

# Go mod needs git
RUN apk update && apk add --no-cache git

RUN mkdir /app
ADD . /app/
WORKDIR /app

RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -ldflags="-w -s"
CMD ["/app/<package>"]

FROM alpine:latest AS alpine
COPY --from=builder /app/<package> /app/<package>
CMD ["/app/<package>"]

FROM scratch AS binary
COPY --from=builder /app/<package> /app/<package>
CMD ["/app/<package>"]
```

`docker build --target=binary -t go-binary:latest`

=> ~5MB image(depends on dependencies)