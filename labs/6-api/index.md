## What plugins are running

Run `docker info` to see what plugins are running.

    Plugins:
     Volume: local
     Network: bridge host macvlan null overlay
     Log: awslogs fluentd gcplogs gelf journald json-file logentries splunk syslog

## Events and API

Open up two ssh connections to the lab machine.

In one window run:
```bash
docker events
```

Now in the other window run these commands against the docker engine API. For every command you run watch what docker events are shown in the first window.

### Pull an image
```bash
curl -s --unix-socket /var/run/docker.sock \
  -X POST "http:/v1.24/images/create?fromImage=alpine" | jq
```

### Run a container

Note that unlike the cli command `docker run`, this api command does not automatically pull if the image is not found locally.
```bash
curl -s --unix-socket /var/run/docker.sock -H "Content-Type: application/json" \
  -d '{"Image": "alpine", "Cmd": ["echo", "hello world"]}' \
  -X POST http:/v1.24/containers/create | jq
```

Copy the container ID from the previous output for use in the subsequent commands.

```bash
curl -s --unix-socket /var/run/docker.sock -X POST http:/v1.24/containers/d33fc3c3988ac638272a396d8081fd312a6388b120a1a2187dae42a6b54e1e6d/start

curl -s --unix-socket /var/run/docker.sock -X POST http:/v1.24/containers/d33fc3c3988ac638272a396d8081fd312a6388b120a1a2187dae42a6b54e1e6d/wait
```

Note that the container exited with no error and status code 0. Let's read the logs:

```bash
curl -s --unix-socket /var/run/docker.sock "http:/v1.24/containers/d33fc3c3988ac638272a396d8081fd312a6388b120a1a2187dae42a6b54e1e6d/logs?stdout=1"
```

    hello world



### Run a container in the background

Pull the image and create the container.
```bash

# pull first
curl -s --unix-socket /var/run/docker.sock \
  -X POST "http:/v1.24/images/create?fromImage=bfirsh/reticulate-splines" | jq


curl -s --unix-socket /var/run/docker.sock -H "Content-Type: application/json" \
  -d '{"Image": "bfirsh/reticulate-splines"}' \
  -X POST http:/v1.24/containers/create | jq
```

```json
{
  "Id": "bbeba6f57302a5a6d29e053c0ff0702738cf5e060857799cac9256b151e47d32",
  "Warnings": null
}
```

Start the container
```bash
curl -s --unix-socket /var/run/docker.sock -H "Content-Type: application/json" \
    -X POST http:/v1.24/containers/bbeba6f57302a5a6d29e053c0ff0702738cf5e060857799cac9256b151e47d32/start | jq
```

### List the containers
```bash
curl -s --unix-socket /var/run/docker.sock http:/v1.24/containers/json | jq
```

```json
[
  {
    "Id": "bbeba6f57302a5a6d29e053c0ff0702738cf5e060857799cac9256b151e47d32",
    "Names": [
      "/confident_mcclintock"
    ],
    "Image": "bfirsh/reticulate-splines",
    "ImageID": "sha256:b1666055931f332541bda7c425e624764de96c85177a61a0b49238a42b80b7f9",
    "Command": "/usr/local/bin/run.sh",
    "Created": 1525073257,
    "Ports": [],
    "Labels": {},
    "State": "running",
    "Status": "Up 9 seconds",
    "HostConfig": {
      "NetworkMode": "default"
    },
    "NetworkSettings": {
      "Networks": {
        "bridge": {
          "IPAMConfig": null,
          "Links": null,
          "Aliases": null,
          "NetworkID": "d6eab5bb355ec06ac3761e2876955a02625c7faf86ced4b5f1c0081575440c92",
          "EndpointID": "ffb2f46a8790b3dcc0f7e0db37e1949cfc34bb5d52521ddfd51e640649347e5e",
          "Gateway": "172.17.0.1",
          "IPAddress": "172.17.0.2",
          "IPPrefixLen": 16,
          "IPv6Gateway": "",
          "GlobalIPv6Address": "",
          "GlobalIPv6PrefixLen": 0,
          "MacAddress": "02:42:ac:11:00:02",
          "DriverOpts": null
        }
      }
    },
    "Mounts": []
  }
]
```

How does this compare to `docker ps`?

```bash
docker ps
```

    CONTAINER ID        IMAGE                       COMMAND                  CREATED             STATUS              PORTS               NAMES
    bbeba6f57302        bfirsh/reticulate-splines   "/usr/local/bin/run.…"   2 minutes ago       Up About a minute                       confident_mcclintock

### Stop a running containers
Be sure to use a container id from a previous step:
```bash
curl -s --unix-socket /var/run/docker.sock \
  -X POST http:/v1.24/containers/bbeba6f57302/stop
```

Print the container logs, again use the ID from above instead of the example id.

```bash
curl -s --unix-socket /var/run/docker.sock "http:/v1.24/containers/bbeba6f57302/logs?stdout=1"
```

    Reticulating spline 1...
    Reticulating spline 2...
    Reticulating spline 3...
    Reticulating spline 4...

### Use 2 different API clients

We can use `docker` commands interchangably with `curl` because they are simply api clients.

Run a container
```bash
docker run alpine echo hello world from docker cli
```

    hello world from docker cli

Store that container's ID in a variable:
```bash
CONTAINER_ID="$(docker ps -lq)"
```

Now read that container's logs with the container id:
```bash
curl -s --unix-socket /var/run/docker.sock "http:/v1.24/containers/$CONTAINER_ID/logs?stdout=1"
```

    hello world from docker cli

### List all images
```bash
curl -s --unix-socket /var/run/docker.sock http:/v1.24/images/json | jq
```

```json
[
  {
    "Containers": -1,
    "Created": 1525073851,
    "Id": "sha256:1163ab4068b74b3dfa75b7edd279edd5a2a3c611d7e5451c3d378986986f4dc7",
    "Labels": {},
    "ParentId": "sha256:3fd9065eaf02feaf94d68376da52541925650b81698c53c6824d92ff63f98353",
    "RepoDigests": null,
    "RepoTags": [
      "helloworld:latest"
    ],
    "SharedSize": -1,
    "Size": 4147781,
    "VirtualSize": 4147781
  },
...
```

### Commit a container
```bash
docker run -d alpine touch /helloworld

#Store that container's ID in a variable:
CONTAINER_ID="$(docker ps -lq)"

curl -s --unix-socket /var/run/docker.sock\
  -X POST "http:/v1.24/commit?container=$CONTAINER_ID&repo=helloworld"
```
