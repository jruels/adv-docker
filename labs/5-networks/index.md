# Networks

In this lab we will see how overlay networks are created with docker and inspect the linux networking. Because swarm uses an overlay network, we will use the swarm commands to automatically generate our overlay networks.

## Current State of Linux Networks

View the current interfaces
```bash
sudo ip link
```

    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1460 qdisc pfifo_fast state UP mode DEFAULT group default qlen 1000
        link/ether 42:01:0a:8a:00:03 brd ff:ff:ff:ff:ff:ff
    3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default
        link/ether 02:42:88:ae:ec:2e brd ff:ff:ff:ff:ff:ff

Note there is a docker0 interface. This is the default host docker bridge.

## Create a swarm

Start swarm on your lab vm.

```bash
docker swarm init
```

List all your swarm nodes. Since we're only running on master you'll only see one node

```bash
docker node ls
```

    ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
    wzy6sxorlo54fe50sq25gi80e *   lab1                Ready               Active              Leader              18.03.1-ce


You can filter the nodes by their role. `manager` manages the swarm cluster.
```bash
docker node ls --filter role=manager
```

    ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
    wzy6sxorlo54fe50sq25gi80e *   lab1                Ready               Active              Leader              18.03.1-ce

`worker` is a swarm worker node. In this case we don't have any.

```bash
docker node ls --filter role=worker
```

    ID                  HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION


List the network on the manager. Notice that it has an overlay network called `ingress` and a bridge network called `docker_gwbridge`

```bash
docker network ls
```

    NETWORK ID          NAME                DRIVER              SCOPE
    f0d185a0ef87        bridge              bridge              local
    b3de6a68c312        docker_gwbridge     bridge              local
    34bb8861d8eb        host                host                local
    mgh6hwlyeg8p        ingress             overlay             swarm
    914aa364f861        none                null                local

The docker_gwbridge connects the ingress network to the Docker host’s network interface so that traffic can flow to and from swarm managers and workers. So the docker_gwbridge connects docker daemons in a swarm. If you create swarm services and do not specify a network, they are connected to the ingress network. It is recommended that you use separate overlay networks for each application or group of applications which will work together. In the next procedure, you will create two overlay networks and connect a service to each of them.

Let's take a look at the network devices we have on the linux host:

```bash
sudo ip link
```

    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1460 qdisc pfifo_fast state UP mode DEFAULT group default qlen 1000
        link/ether 42:01:0a:8a:00:03 brd ff:ff:ff:ff:ff:ff
    3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default
        link/ether 02:42:88:ae:ec:2e brd ff:ff:ff:ff:ff:ff
    8: docker_gwbridge: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default
        link/ether 02:42:53:a1:aa:e2 brd ff:ff:ff:ff:ff:ff
    10: veth7064f01@if9: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker_gwbridge state UP mode DEFAULT group default
        link/ether c2:4d:17:e6:27:bd brd ff:ff:ff:ff:ff:ff link-netnsid 1

Notice there is now a `docker_gwbridge` and `veth` shown. Note that the `veth` interface links to network namespace 1, `link-netnsid 1`. Let's take a look at these.

First let's explore the linux bridges.

```bash
sudo brctl show
```

    bridge name	bridge id		STP enabled	interfaces
    docker0		8000.024288aeec2e	no
    docker_gwbridge		8000.024253a1aae2	no		veth7064f01

Note that the veth interface is on the `docker_gwbridge` bridge.

veth interfaces come in pairs. Where is the other side? Notice the name of the interface `veth7064f01@if9`, that `@if9` is an indication that interface 9 is the veth pair to interface 10. Where is interface 10? In network namespace 1, but how do we find out more about it?

Let's try and learn about our namespaces

```bash
```

    [nothing]

Hrm, no output. Why not? Docker by default doesn't share namespace information for standard linux tools to read, but we can fix this with a simple symlink.

```bash
sudo ln -s /var/run/docker/netns /var/run/netns
```

Let's try that again

```bash
sudo ip netns
```

    1-a4xiy43kwu (id: 0)
    ingress_sbox (id: 1)

Here we see namespaces 0 and 1. Now we can use the `ip` command to learn about what's in these namespaces.

```bash
sudo ip netns exec ingress_sbox ip addr
```

    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
           valid_lft forever preferred_lft forever
    6: eth0@if7: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default
        link/ether 02:42:0a:ff:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
        inet 10.255.0.2/16 brd 10.255.255.255 scope global eth0
           valid_lft forever preferred_lft forever
    9: eth1@if10: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
        link/ether 02:42:ac:12:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 1
        inet 172.18.0.2/16 brd 172.18.255.255 scope global eth1
           valid_lft forever preferred_lft forever

Here we see the other side of the veth pair on the host network, and an additional veth interface to the other namespace. Let's run the same thing on that namespace. Be sure to use the namespace name from the `sudo ip netns` command above.

```bash
sudo ip netns exec 1-a4xiy43kwu ip addr
```

    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
           valid_lft forever preferred_lft forever
    2: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default
        link/ether 16:84:5c:da:a3:d6 brd ff:ff:ff:ff:ff:ff
        inet 10.255.0.1/16 brd 10.255.255.255 scope global br0
           valid_lft forever preferred_lft forever
    5: vxlan0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master br0 state UNKNOWN group default
        link/ether 56:ac:7b:37:b7:e0 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    7: veth0@if6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master br0 state UP group default
        link/ether 16:84:5c:da:a3:d6 brd ff:ff:ff:ff:ff:ff link-netnsid 1

You can tell from the `vxlan0` device that this is the namespace used for traffic encapsulation between containers.

### Create the services with the default overlay network

The service will publish port 80 to the outside world. All of the service task containers can communicate with each other without opening any ports.

```bash
docker service create \
  --name my-nginx-default-net \
  --publish target=80,published=80 \
  --replicas=2 \
  nginx
```

Run `docker service ls` to monitor and verify the service is up:

```bash
docker service ls
```

    ID                  NAME                   MODE                REPLICAS            IMAGE               PORTS
    p8qcmhmbxq2f        my-nginx-default-net   replicated          2/2                 nginx:latest        *:80->80/tcp

Inspect the ingress network. The output will be long, but notice the Containers and Peers sections. Containers lists all service tasks (or standalone containers) connected to the overlay network from that host. Note that the peer address is your host IP address.

```bash
docker inspect ingress
```

```json
[
    {
        "Name": "ingress",
        "Id": "mgh6hwlyeg8p7qwz7p77cgl7f",
        "Created": "2018-03-29T07:11:34.446364427Z",
        "Scope": "swarm",
        "Driver": "overlay",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "10.255.0.0/16",
                    "Gateway": "10.255.0.1"
...
```

Note the subnet and container IP addresses and MAC addresses.


Take a look at the vxlan namespace again. Notice anything new?

```bash
sudo ip netns exec 1-a4xiy43kwu ip addr
```

    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
           valid_lft forever preferred_lft forever
    2: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default
        link/ether 16:84:5c:da:a3:d6 brd ff:ff:ff:ff:ff:ff
        inet 10.255.0.1/16 brd 10.255.255.255 scope global br0
           valid_lft forever preferred_lft forever
    5: vxlan0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master br0 state UNKNOWN group default
        link/ether 56:ac:7b:37:b7:e0 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    7: veth0@if6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master br0 state UP group default
        link/ether 16:84:5c:da:a3:d6 brd ff:ff:ff:ff:ff:ff link-netnsid 1
    12: veth1@if11: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master br0 state UP group default
        link/ether 82:a9:a9:30:26:6b brd ff:ff:ff:ff:ff:ff link-netnsid 2
    16: veth2@if15: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master br0 state UP group default
        link/ether 36:a0:7e:48:59:4a brd ff:ff:ff:ff:ff:ff link-netnsid 3

Notice there are two more veth interfaces, both pointing to two new namespaces. These must be for our nginx containers.


## Create a custom overlay network

Create a new overlay network called `nginx-net`

```bash
docker network create -d overlay nginx-net
```

    v1qhhmk5qmotligdrvmvi2md4

If your master had any worker nodes, you wouldn't need to explicitly create the network there. The network will automatically be created when a node starts running a service task requiring that network.

Next try to create a new 2-replica nginx service connected to `nginx-net`.

```bash
docker service create \
  --name my-nginx-custom-overlay \
  --publish target=80,published=80 \
  --replicas=2 \
  --network nginx-net \
  nginx
```

    Error response from daemon: rpc error: code = InvalidArgument desc = port '80' is already in use by service 'my-nginx-default-net' (p8qcmhmbxq2fbc7k8co5hlfnc) as an ingress port

What does this happen? It's because of the mesh network created by the ingress publish mode. The default publish mode of ingress, which is used when you do not specify a mode for the --publish flag, means that if you browse to port 80 on a node you will be connected to port 80 on one of the 2 service tasks, even if no tasks are currently running on the node you browse to. If you want to publish the port using host mode, you can add mode=host to the --publish output. However, you should also use --mode global instead of --replicas=5 in this case, since only one service task can bind a given port on a given node.

Let's try that again but we'll publish on a different node port.

```bash
docker service create \
  --name my-nginx-custom-overlay \
  --publish target=80,published=81 \
  --replicas=2 \
  --network nginx-net \
  nginx
```

    neqfm1pwytzd6lozeapblefdu
    overall progress: 2 out of 2 tasks
    1/2: running   [==================================================>]
    2/2: running   [==================================================>]
    verify: Service converged

Run docker service ls to monitor the progress of service bring-up, which may take a few seconds. Verify both our swarm services are running on the ports we expect.

```bash
docker service ls
```

    ID                  NAME                      MODE                REPLICAS            IMAGE               PORTS
    neqfm1pwytzd        my-nginx-custom-overlay   replicated          2/2                 nginx:latest        *:81->80/tcp
    p8qcmhmbxq2f        my-nginx-default-net      replicated          2/2                 nginx:latest        *:80->80/tcp

Inspect the nginx-net network. The output will be long, but notice the Containers and Peers sections. Containers lists all service tasks (or standalone containers) connected to the overlay network from that host. Note that the peer address is your host IP address.

```bash
docker inspect nginx-net
```

```json
[
    {
        "Name": "nginx-net",
        "Id": "v1qhhmk5qmotligdrvmvi2md4",
        "Created": "2018-03-29T19:42:24.157642282Z",
        "Scope": "swarm",
        "Driver": "overlay",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "10.0.0.0/24",route -
                    "Gateway": "10.0.0.1"...
```

Inspect the service using docker service inspect my-nginx-custom-overlay and notice the information about the ports and endpoints used by the service.

```bash
docker inspect my-nginx-custom-overlay
```

```json
[
    {
        "ID": "neqfm1pwytzd6lozeapblefdu",
        "Version": {
            "Index": 119
        },
        "CreatedAt": "2018-03-29T19:42:23.988245708Z",
        "UpdatedAt": "2018-03-29T19:42:23.990449163Z",
        "Spec": {
            "Name": "my-nginx-custom-overlay",...
```

Again look at the interfaces on the linux host. You should see an additional veth interface for every container running on the host.

```bash
sudo ip link
```

    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1460 qdisc pfifo_fast state UP mode DEFAULT group default qlen 1000
        link/ether 42:01:0a:8a:00:03 brd ff:ff:ff:ff:ff:ff
    3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default
        link/ether 02:42:e4:ab:d5:6b brd ff:ff:ff:ff:ff:ff
    8: docker_gwbridge: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default
        link/ether 02:42:6b:84:81:c5 brd ff:ff:ff:ff:ff:ff
    177: vethc0c9556@if176: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker_gwbridge state UP mode DEFAULT group default
        link/ether 82:69:4b:1c:63:95 brd ff:ff:ff:ff:ff:ff link-netnsid 1
    183: vethe03bbf1@if182: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker_gwbridge state UP mode DEFAULT group default
        link/ether ba:6b:01:d1:e8:1e brd ff:ff:ff:ff:ff:ff link-netnsid 2
    185: veth32eb6a9@if184: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker_gwbridge state UP mode DEFAULT group default
        link/ether be:5d:1c:eb:4c:96 brd ff:ff:ff:ff:ff:ff link-netnsid 3
    189: veth28670f9@if188: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker_gwbridge state UP mode DEFAULT group default
        link/ether ea:eb:15:35:a7:fe brd ff:ff:ff:ff:ff:ff link-netnsid 5
    196: veth735e768@if195: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker_gwbridge state UP mode DEFAULT group default
        link/ether 4e:c1:3a:c0:2b:93 brd ff:ff:ff:ff:ff:ff link-netnsid 6

## Cleanup

Note: Even though overlay networks are automatically created on swarm worker nodes as needed, they are not automatically removed.

Clean up the service and the networks with the following commands.

```bash
docker service rm my-nginx-custom-overlay
docker service rm my-nginx-default-net
docker network rm nginx-net
docker swarm leave --force
```
