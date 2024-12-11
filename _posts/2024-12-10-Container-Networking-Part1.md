---
title: "Container Networking: Part One"
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

This is the first part of the series [Container Networking](https://simplyatul.github.io/blog/Container-Networking/). 
I will cover Virtual ethernet devices in this blog post.

Generally, any machine has loopback and ethernet network interfaces. You can check the available interfaces using

```bash
ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 02:fd:4d:34:55:76 brd ff:ff:ff:ff:ff:ff
```

```enp0s3``` is an interface of type ethernet. You can communicate with machines outside the VM using this interface.

Let's create Virtual Ethernet device. [veth man page](https://man7.org/linux/man-pages/man4/veth.4.html) says

```text
The veth devices are Virtual Ethernet devices.  They can act as
tunnels between network namespaces to create a bridge to a
physical network device in another namespace, but can also be
used as standalone network devices.
```

We will see namespaces and bridges later, but let us see how we can create 
```veth``` interface(s) and their usage.

## Creating veth pair

```bash
sudo ip link add vethX type veth peer name vethY
```

```bash
ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 02:fd:4d:34:55:76 brd ff:ff:ff:ff:ff:ff
3: vethY@vethX: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default qlen 1000
    link/ether 5a:bc:4d:7e:76:b1 brd ff:ff:ff:ff:ff:ff
4: vethX@vethY: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default qlen 1000
    link/ether 5e:4b:13:90:7d:63 brd ff:ff:ff:ff:ff:ff
```

By default, they are down. You need to make them up

```bash
sudo ip link set vethX up
sudo ip link set vethY up
```

```bash
ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 02:fd:4d:34:55:76 brd ff:ff:ff:ff:ff:ff
3: vethY@vethX: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 5a:bc:4d:7e:76:b1 brd ff:ff:ff:ff:ff:ff
4: vethX@vethY: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 5e:4b:13:90:7d:63 brd ff:ff:ff:ff:ff:ff

```
The ```veth``` pair is very special. Packets transmitted on one device in the 
pair are immediately received on the other device.

## Traffic on veth pair
To send the packet using ```veth```, we will use [Scapy](https://scapy.net/) 
tool. It is Python-based interactive packet manipulation program and library.

Code to send single packet looks like

```bash
cat <<EOF >> onepkt.py
#! /usr/bin/env python3

import sys
from scapy.all import *


if __name__ == '__main__':
    usage_string = """Usage:
  onepkt.py <src-mac> <dst-mac> <iface> <msg>
    where <msg> is unique identifier for this message"""

    # total arguments
    if (len(sys.argv) != 5):
        sys.exit("Incorrect usage - num args.\n"+usage_string)

    src_mac = sys.argv[1]
    dst_mac = sys.argv[2]
    iface = sys.argv[3]
    msg = sys.argv[4]

    src_ip = "1.1.1.1"
    dst_ip = "2.2.2.2"
    sport = 1111
    dport = 2222


    pkt = Ether(dst=dst_mac, src=src_mac) / IP (src=src_ip, dst=dst_ip) / TCP(sport=sport, dport=dport) / msg

    pkt.show()

    sendp(pkt, iface=iface)

EOF
```
Code Credits: https://github.com/eric-keller/npp-linux-01-intro/blob/main/demo3/onepkt.py

In one terminal window, start the ```tshark``` on ```vethY```

```bash
sudo tshark -T fields -e eth -i vethY
Running as user "root" and group "root". This could be dangerous.
Capturing on 'vethY'
 ** (tshark:9349) 10:26:21.105768 [Main MESSAGE] -- Capture started.
 ** (tshark:9349) 10:26:21.105897 [Main MESSAGE] -- File: "/tmp/wireshark_vethY4R8LY2.pcapng"

```

Now run the following python code to send the single packet to ```vethX``` device

```bash
sudo python3 ./onepkt.py 22:11:11:11:11:11 22:22:22:22:22:22 vethX 123
###[ Ethernet ]###
  dst       = 22:22:22:22:22:22
  src       = 22:11:11:11:11:11
  type      = IPv4
###[ IP ]###
     version   = 4
     ihl       = None
     tos       = 0x0
     len       = None
     id        = 1
     flags     =
     frag      = 0
     ttl       = 64
     proto     = tcp
     chksum    = None
     src       = 1.1.1.1
     dst       = 2.2.2.2
     \options   \ 
###[ TCP ]###     
        sport     = 1111
        dport     = 2222
        seq       = 0   
        ack       = 0
        dataofs   = None
        reserved  = 0
        flags     = S
        window    = 8192
        chksum    = None
        urgptr    = 0
        options   = []
###[ Raw ]### 
           load      = '123'

.
Sent 1 packets.

```

On the ```tshark``` window, you will see the packet has received.

```bash
sudo tshark -T fields -e eth -i vethY
Running as user "root" and group "root". This could be dangerous.
Capturing on 'vethY'
 ** (tshark:9349) 10:26:21.105768 [Main MESSAGE] -- Capture started.
 ** (tshark:9349) 10:26:21.105897 [Main MESSAGE] -- File: "/tmp/wireshark_vethY4R8LY2.pcapng"

Ethernet II, Src: 22:11:11:11:11:11 (22:11:11:11:11:11), Dst: 22:22:22:22:22:22 (22:22:22:22:22:22)

```

The ```veth``` pairs play the critical role establishing Container Networking. 
We will see it in [next part](https://simplyatul.github.io/blog/Container-Networking-Part2/) of the blog.

## More on veth

If you have many ```veth``` pairs, then given a ```veth``` device how can you 
identify the peer? ```ethtool``` utility comes to rescue. ```ethtool``` tells 
us the peer's index.

```bash
ethtool -S vethX | grep peer
     peer_ifindex: 3
```
```peer_ifindex: 3``` indicates index of peer device. The index is show in 
```ip a``` command.

```bash
ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 02:fd:4d:34:55:76 brd ff:ff:ff:ff:ff:ff
3: vethY@vethX: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 5a:bc:4d:7e:76:b1 brd ff:ff:ff:ff:ff:ff
4: vethX@vethY: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 5e:4b:13:90:7d:63 brd ff:ff:ff:ff:ff:ff
```

Although device name ```vethX@vethY``` tells the peer name, but when devices 
are created by container softwares (like docker, podman, etc), the device names
are not that straight forward.

You can get detailed, prettier information of a ```veth``` device using

```bash
ip -d -j -p link show vethX
[ {
        "ifindex": 4,
        "link": "vethY",
        "ifname": "vethX",
        "flags": [ "BROADCAST","MULTICAST","UP","LOWER_UP" ],
        "mtu": 1500,
        "qdisc": "noqueue",
        "operstate": "UP",
        "linkmode": "DEFAULT",
        "group": "default",
        "txqlen": 1000,
        "link_type": "ether",
        "address": "5e:4b:13:90:7d:63",
        "broadcast": "ff:ff:ff:ff:ff:ff",
        "promiscuity": 0,
        "min_mtu": 68,
        "max_mtu": 65535,
        "linkinfo": {
            "info_kind": "veth"
        },
        "inet6_addr_gen_mode": "eui64",
        "num_tx_queues": 2,
        "num_rx_queues": 2,
        "gso_max_size": 65536,
        "gso_max_segs": 65535
    } ]

```

With this, we come to an end of this blog post. Next in the series is [Network 
Namespaces and Bridges](https://simplyatul.github.io/blog/Container-Networking-Part2/)