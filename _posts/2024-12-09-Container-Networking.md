---
title: "Container Networking: HowTo"
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

# Intro...
My first encounter with container was - I had performed load testing on an 
application using [JMeter][1]. So rather installing [JMeter][1] on multiple 
Virtual Machines (VMs), I ended deploying them on multiple containers on a single VM.

I still remember, I was curious to understand how a container can 
manage to keep applications isolated. I was interested in understanding 
networking part specifically.

When I searched, lot of terms coined around. Like [veth](https://man7.org/linux/man-pages/man4/veth.4.html), [bridges](https://en.wikipedia.org/wiki/Network_bridge), 
[Linux Namespaces](https://en.wikipedia.org/wiki/Linux_namespaces), [cgroups](https://en.wikipedia.org/wiki/Cgroups). I had lot of queries in my mind too.


- How a network packet traverse from Host to container and vice-versa?
- How container maintains separate network space for themselves?
- Post installing docker, I see ```docker0``` device in ```ip a``` command output. What is its use?
- Post creating container, I see ```eth0@if5``` device in container, whereas ```veth6e6a37e@if4``` device on Host. Are they related to ```docker0``` device?
- And many more...

After a lot of readings (yes, it took me some time), I understood few of 
the stuff which I would like to share in this blog series.

I assume reader has basic understanding of Linux, Virtualization and 
Containerization on a high level.

# Topics To Cover

I have broken down this blog post in following parts

1. [Part One](https://simplyatul.github.io/blog/Container-Networking-Part1/): Virtual Ethernet
2. [Part Two](https://simplyatul.github.io/blog/Container-Networking-Part2/): Network Namespaces and Network Bridges
3. [Part Three](https://simplyatul.github.io/blog/Container-Networking-Part3/): Container (docker) Networking

# Setup

I am going to use Vagrant VM on the Ubuntu host with [Oracle VirtualBox](https://www.virtualbox.org/wiki/Linux_Downloads) v7.0. The biggest advantage of using 
Vagrant VM is it provides a clean slate to play with. One can destroy, 
create, duplicate, share it very easily.

Here is the Vagrant File

```yaml
cat <<EOF >> Vagrantfile
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
 config.vm.box = "ubuntu/jammy64"
 config.vm.box_version = "20241002.0.0"

 config.vm.synced_folder "./shared-with-vm", "/shared-with-host"
 config.vm.hostname = "cnd" # Container Networking Demo

 config.vm.provider "virtualbox" do |vb|
    # Display the VirtualBox GUI when booting the machine
    # vb.gui = true
 
    # Customize the amount of memory on the VM:
    vb.memory = "2048"
    vb.cpus = 2
    vb.name = "Container Networking Demo"
  end
end
EOF
```

Create a directory ```shared-with-vm``` in the same location where 
Vagrantfile resides. This helps to share the data between host and VM

```bash
$ mkdir shared-with-vm
```

Start the VM and logged into it
```bash
$ vagrant up
$ vagrant ssh
```

Install few utilities

```bash
vagrant@cnd:~$ sudo apt update
vagrant@cnd:~$ sudo apt install -y net-tools vim curl git tree traceroute make dos2unix bind9-dnsutils tshark ethtool python3 python3-pip python3-scapy iputils-ping iproute2

# Note: you may need to restart few services. Do as it pops up
vagrant@cnd:~$ systemctl restart networkd-dispatcher.service unattended-upgrades.service
```

Last thing (optional, just to advertize my older blog post 
:stuck_out_tongue_winking_eye:), you can set the aliases. I do 
this [way](https://hackernoon.com/bash-aliases-take-them-with-you).

```bash
vagrant@cnd:~$ wget https://raw.githubusercontent.com/simplyatul/bin/master/setaliases.sh
vagrant@cnd:~$ source setaliases.sh 
```
All set, let us jump to [Part One](https://simplyatul.github.io/blog/Container-Networking-Part1/).

[1]: https://jmeter.apache.org/