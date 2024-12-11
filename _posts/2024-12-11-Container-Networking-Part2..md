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
I will cover Network Namespaces in this blog post.

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
$ sudo ip netns add ns1
```
List them

```bash
$ ip netns list
ns1
```

Check loopback interface is available in a namespace

```bash
$ sudo ip netns exec ns1 ip link
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```
Following diagram helps to visualize the setup

<img src="{{ site.url }}{{ site.baseurl }}/assets/images/cnd-1.png" alt="">

### TODO DIAGRAM 1

For brevity, I will not list ```lo``` and ```enp0s3``` host interfaces in the 
rest of the diagrams.

Now, let's create veth pairs

### TODO DIAGRAM 2

```bash
$ sudo ip link add vethX type veth peer name vethY
$ sudo ip link set vethX up
$ sudo ip link set vethY up
```

Attach one of the ```veth``` pair to ```ns1``` namespace.

```bash
$ sudo ip link set vethX netns ns1
```

### TODO DIAGRAM 3

Notice that processes in the default/root namespace could not see ```vethX``` 
interface 

```bash
$ sudo ip link
```

And processes in ```ns1``` namespace could not see ```vethY``` 
interface.

```bash
$ sudo ip netns exec ns1 ip link
```

In similar fashion, we can create another namespaces and attach another pair 
of ```veth``` interfaces. It looks as

### TODO DIAGRAM 4


```bash
$ sudo ip netns add ns2
$ sudo ip link add vethP type veth peer name vethQ
$ sudo ip link set vethP up
$ sudo ip link set vethQ up
$ sudo ip link set vethP netns ns2
```

Now the last part is connect these two namespaces with Bridge.

Bridge connects two LAN segments together. We will create a new bridge and 
attach ```vethY``` and ```vethQ``` to this bridge.

### TODO DIAGRAM 5

As we looked in last blog post, any packet on bridge ```br0``` is seen on 
all the ```veths``` we have created.

```bash
$ cat <<EOF >> bridge.sh
#!/bin/bash

ip link add name br0 type bridge
ip link set dev br0 up
ip link set vethY master br0
ip link set vethQ master br0
EOF

$ chmod +x bridge.sh

$ sudo ./bridge.sh
```

## Clean up

```bash

```
### BACKUP

Now you might have started imagining how a container achieves the application 
isolation. It will get clarified soon.

Also you might already thought that we can connect two namespaces as well.
And yes, you are correct. It looks something like below diagram



Here are the commands

```bash

```


[1]: https://en.wikipedia.org/wiki/Linux_namespaces