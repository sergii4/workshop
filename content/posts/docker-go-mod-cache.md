---
title: "Docker, go modules and private repos"
date: 2020-03-29T20:14:27+03:00
tags: ["docker", "go", "git"]
draft: false
---
![go modules img](/img/go-mod.png)
Go modules change the way we work with dependency not only locally but in Docker(CI) as well.

First problem we face is caching dependency. It resolves quite simple as docker layer:
> When building an image, Docker steps through the instructions in your Dockerfile, executing each in the order specified. As each instruction is examined, Docker looks for an existing image in its cache that it can reuse, rather than creating a new (duplicate) image.

```
# We want to populate the module cache based on the go.{mod,sum} files.
COPY go.mod go.sum ./

#This is the ‘magic’ step that will download all the dependencies that are specified in
# the go.mod and go.sum file.

# Because of how the layer caching system works in Docker, the go mod download
# command will _ only_ be re-run when the go.mod or go.sum file change
# (or when we add another docker instruction this line)
RUN go mod download 
```

Looks like we are done. But only if you do not use private repo. If you do you see something like that: `authentication failed` or `repository not found`.
One additional step can help us:
```
COPY go.mod go.sum ./

RUN git config --global url."https://${GITHUB_ACCESS_TOKEN}:x-oauth-basic@github.com/PrivateRepo".insteadOf \
        "https://github.com/PrivateRepo"


RUN go mod download
```

You might want to use `--local` config, but it leads to:
>fatal: could not read Username for 'https://github.com': terminal prompts disabled

But with `--global` it works fine
