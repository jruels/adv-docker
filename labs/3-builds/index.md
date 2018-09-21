## Dockerfile Optimization

Your boss Susan complains that even though everything is working, builds and deploys are taking longer than expected. Would you mind taking a look at these Dockerfiles to see if you can optimize them?

### Problem 1
Download: [1.Dockerfile](1.Dockerfile)

### Problem 2
Download: [2.Dockerfile](2.Dockerfile)

### Problem 3
Download: [3.Dockerfile](3.Dockerfile)

### Problem 4
Download: [4.Dockerfile](4.Dockerfile)

### Problem 5
This container isn't just larger than it needs to be, but a /var/lib file raised a flag with our security scanner. Think you might be able to reduce the footprint *and* pass the security scan?

You will need to clone the git repository with this command on your development box:

```bash
git clone https://github.com/jruels/docker-go-hello-world.git 
```

Build and run the file to see if it worked:

```bash
cd docker-go-hello-world
docker build -t helloworld .
docker run helloworld
```

Upon inspecting the image size, you'll see that it is quite large!

```bash
$ docker images | grep -i helloworld
helloworld                                                                latest                   ffead7b5dd8d        6 minutes ago        244MB
```

Let's see if we can incorporate mult-stage build concepts to slim this down significantly and make our application more portable.

Change your Dockerfile to look like this. We've labeled the first build stage as simply `builder`. This is because the actual
compilation of the go binary occurs in this layer. Additionally, we're going to introduce an additional alpine layer
that contains no go libraries of any kind. This layer will contain only the base alpine filesystem and your go binary.

```bash
FROM golang:1.9-alpine as builder

CMD ["/go/bin/hello"]

COPY main.go /go/src/github.com/chrishiestand/docker-go-hello-world/

RUN cd /go/src/github.com/chrishiestand/docker-go-hello-world && \
    go get && \
    CGO_ENABLED=0 GOOS=linux go build -a -o /go/bin/hello main.go

FROM alpine:latest
COPY --from=builder /go/bin/hello /usr/local/bin
CMD ["/usr/local/bin/hello"]
```

Now let's build this with a `slimmer` tag to compare the image sizes.

```bash
docker build -t helloworld:slimmer .
docker run helloworld:slimmer
```

```bash
$ docker images | grep -i hello
helloworld                                                                slimmer                  6a165b9caec9        About a minute ago   6.27MB
helloworld                                                                latest                   ffead7b5dd8d        6 minutes ago        244MB
```

By simply incorporating multi-stage builds, we've managed to drastically reduce the image's on-disk footprint by a factor of 40.

This multi-stage approach also benefits from portability. The docker build could be ported to any build system (Jenkins, TravisCI, CircleCI, etc)
and no consideration needs to be made as to whether the build system "supports" go builds.

### Extra

Finish early? Take a look at these Dockerfiles from a pro. Perhaps not perfectly optimized, but great examples: [https://github.com/jessfraz/dockerfiles](https://github.com/jessfraz/dockerfiles)
