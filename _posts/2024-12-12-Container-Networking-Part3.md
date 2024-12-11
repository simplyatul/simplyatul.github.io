---
title: "Container Networking: Part Two"
author_profile: false
classes: wide
categories:
  - Blog
tags:
  - Container
  - Linux Networking
  - docker
  - cloud
---

This is the third part of the series [Container Networking](https://simplyatul.github.io/blog/Container-Networking/). I will explain little bit of docker container 
networking in this blog post.

I followed steps mentioned at [this](https://docs.docker.com/engine/install/ubuntu/) to install docker on Ubuntu.

Post docker install, you can see ```docker0``` device in the list

```bash
$ ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 02:fd:4d:34:55:76 brd ff:ff:ff:ff:ff:ff
11: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default 
    link/ether 02:42:cf:b4:c1:a8 brd ff:ff:ff:ff:ff:ff
```

Let's create a busybox container

```bash
$ docker run --name bb -dt busybox
```

```bash
$ docker ps -a
CONTAINER ID   IMAGE     COMMAND   CREATED              STATUS              PORTS    NAMES
02429964e449   busybox   "sh"      About a minute ago   Up About a minute             bb
```
In device list, you see ```veth7b01920@if16``` interface is created with master as docker0

```bash
$ ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 02:fd:4d:34:55:76 brd ff:ff:ff:ff:ff:ff
11: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default 
    link/ether 02:42:cf:b4:c1:a8 brd ff:ff:ff:ff:ff:ff
17: veth7b01920@if16: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP mode DEFAULT group default 
    link/ether e2:15:50:88:11:03 brd ff:ff:ff:ff:ff:ff link-netnsid 2
```

Let's check the network namespace

```bash
$ ip netns list
```

And you won't see any network namespaces...How come??? This is because

```text
ip netns list command looks up network namespaces file in the 
/var/run/netns directory.

However, the Docker daemon doesn’t create a reference of
the network namespace file in the /var/run/netns directory
after the creation. Therefore, ip netns ls cannot resolve the
network namespace file

Ref: https://www.baeldung.com/linux/docker-network-namespace-invisible
```

If you want ```ip netns list``` should show the namespace name docker has 
created, then follow below steps

```bash
export container_name=bb
container_pid=$(sudo docker inspect -f '{{.State.Pid}}' $container_name)
echo $container_pid

sudo touch /var/run/netns/$container_name
sudo mount -o bind /proc/$container_pid/ns/net /var/run/netns/$container_name
```

Now you can see the namespace name

```bash
$ ip netns list
bb (id: 0)
```

```bash
sudo ip netns exec bb ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
16: eth0@if17: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default 
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
```

Following diagram will help you to visualize better.

![cnd-1](https://github.com/simplyatul/simplyatul.github.io/blob/master/assets/images/cnd-6.png?raw=true)

