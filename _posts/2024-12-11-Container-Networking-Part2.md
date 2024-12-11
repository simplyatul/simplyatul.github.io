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

This is the second part of the series [Container Networking](https://simplyatul.github.io/blog/Container-Networking/). 
I will cover Network Namespaces and Linux Bridges in this blog post.

Linux Namespaces [wiki][1] says

```text
Namespaces are a feature of the Linux kernel that partition kernel resources 
such that one set of processes sees one set of resources, while another set 
of processes sees a different set of resources. 
```
So the process isolation is provided using namespaces and cgroups. This makes 
each process to see it's personal view of system (files, process, network 
interfaces, hostname).

Some more expert from [wiki][1]

```text
Since kernel version 5.6, there are 8 kinds of namespaces. 
Network namespaces virtualize the network stack. On creation, a network 
namespace contains only a loopback interface. Each network interface 
(physical or virtual) is present in exactly 1 namespace and can be moved 
between namespaces. 
```

[ip-netns](https://man7.org/linux/man-pages/man8/ip-netns.8.html) utility helps 
to create network namespaces. Let's see how.

## Network namespaces demo

Create a network namespaces

```bash
sudo ip netns add ns1
```
List them

```bash
ip netns list
ns1
```

Only loopback interface is available in ```ns1``` namespace.

```bash
sudo ip netns exec ns1 ip link
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```
Following diagram helps to visualize the setup

![cnd-1](https://github.com/simplyatul/simplyatul.github.io/blob/master/assets/images/cnd-1.png?raw=true)

For brevity, I will not list ```lo``` and ```enp0s3``` interfaces in rest of 
the diagrams.

Now, let's create a ```veth``` pair

```bash
sudo ip link add vethX type veth peer name vethY
sudo ip link set vethX up
sudo ip link set vethY up
```

![cnd-1](https://github.com/simplyatul/simplyatul.github.io/blob/master/assets/images/cnd-2.png?raw=true)

Attach one end of the ```veth``` pair to ```ns1``` namespace.

```bash
sudo ip link set vethX netns ns1
```

![cnd-1](https://github.com/simplyatul/simplyatul.github.io/blob/master/assets/images/cnd-3.png?raw=true)

This ensures processes in the default/root namespace could not see ```vethX``` 
interface 

```bash
ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 02:fd:4d:34:55:76 brd ff:ff:ff:ff:ff:ff
18: vethY@if19: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 5a:bc:4d:7e:76:b1 brd ff:ff:ff:ff:ff:ff link-netns ns1
```

And processes in ```ns1``` namespace could not see ```vethY``` interface.

```bash
sudo ip netns exec ns1 ip link
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
19: vethX@if18: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 5e:4b:13:90:7d:63 brd ff:ff:ff:ff:ff:ff link-netnsid 0
```

In similar fashion, we can create another namespace. Create a new ```veth``` 
pair. And then attach one end of ```veth``` pair to a namespace.

```bash
sudo ip netns add ns2
sudo ip link add vethP type veth peer name vethQ
sudo ip link set vethP up
sudo ip link set vethQ up
sudo ip link set vethP netns ns2
```
![cnd-1](https://github.com/simplyatul/simplyatul.github.io/blob/master/assets/images/cnd-4.png?raw=true)

Now the last part is to connect these two namespaces with a Bridge.

Bridge connects two LAN segments together. We will create a new bridge and 
attach ```vethY``` and ```vethQ``` to this bridge.

```bash
sudo ip link add name br0 type bridge
sudo ip link set dev br0 up
sudo ip link set vethY master br0
sudo ip link set vethQ master br0
```

![cnd-1](https://github.com/simplyatul/simplyatul.github.io/blob/master/assets/images/cnd-5.png?raw=true)

As we have looked in [last blog post](https://simplyatul.github.io/blog/Container-Networking-Part1/), any packet on bridge ```br0``` is seen on 
all the ```veths``` we have created. You can try it on yourself.

Now by this time, you might have started imagining how a container achieves the network isolation using Virtual Ethernet, Bridge and Linux Namespaces. We will take a 
look at docker container in [next blog post](https://simplyatul.github.io/blog/Container-Networking-Part3/).

Before you jump, let's clean up the setup

```bash
sudo ip link del br0
sudo ip link del vethY
sudo ip link del vethQ
sudo ip netns del ns1
sudo ip netns del ns2
```

[1]: https://en.wikipedia.org/wiki/Linux_namespaces