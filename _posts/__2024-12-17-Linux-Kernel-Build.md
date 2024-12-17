---
title: "Building Linux Kernel"
author_profile: false
classes: wide
categories:
  - Blog
tags:
  - linux
  - grub
  - linux kernel
  - kernel
  - ubuntu
---

In this blog post, I will list down the steps how to build the Linux Kernel.
And then how to boot from the newly built kernel.

I assume reader has basic understanding of Linux OS.

## VM Setup

I am using Oracle Virtualbox v7 on the Ubuntu (v24.04) x86_64 Host. Download [Ubuntu 20.04 iso](https://hr.releases.ubuntu.com/20.04.6/ubuntu-20.04.6-desktop-amd64.iso). In Virtualbox, create a new VM machine by providing the iso image. Select ```Skip Unattended Installation``` option. This is optional, but I prefer to choose this to observe what's going on.

Minimum resources to allocate are - 
- 2 vCPUs 
- 2 GB RAM
- 100 GB o Hard Disk space.

Once finished, Ubuntu OS will boot and provide option to install it. During installation, choose ```minimal installation``` option and do not install ```extra third party softwares```. This is because we simply don't need these things :grinning:.

Post starting the VM, check the kernel version. I see following at my end.

```bash
uname -r 
5.15.0-126-generic

uname -a
Linux kernel0 5.15.0-126-generic #136~20.04.1-Ubuntu SMP Thu Nov 14 16:38:05 UTC 2024 x86_64 x86_64 x86_64 GNU/Linux
```

Update and upgrade the kernel

```bash
sudo apt update
sudo apt upgrade
```

Reboot the VM once above is completed.

Install SSH Server and enable it

```bash
sudo apt install -y openssh-server 
systemctl enable --now ssh
```

Now install few useful tools

```bash
wget -4 https://raw.githubusercontent.com/simplyatul/vagrant-vms/main/tools-0-install.sh
chmod +x tools-0-install.sh
sudo ./tools-0-install.sh
```
Then install the packages required to build kernel

```bash
wget -4 https://raw.githubusercontent.com/simplyatul/vagrant-vms/refs/heads/main/tools-0-kernel-dev.sh
chmod +x tools-0-kernel-dev.sh
sudo ./tools-0-kernel-dev.sh
```

Let us now checkout stable kernel v6.8.12

```bash
git clone -4 --depth=1 --branch v6.8.12 git://git.kernel.org/pub/scm/linux/kernel/git/stable/linux-stable.git

cd linux-stable
```

to build the kernel, ```.config``` file is required. This file contains kernel configurations. I used my host's config file for this

```bash
# Copy /boot/config-6.8.0-48-generic from my host to VM's /tmp directory

cp /tmp/config-6.8.0-48-generic .config
```

Now do ```menuconfig```

```bash
make menuconfig

# Make following changes
# General Setup -> Kernel .config support => Make it *
# General Setup -> Kernel .config support -> Enable access to .config through /proc/config.gz => Select it
# General Setup -> Local version - append to kernel release => Enter -at-bl0 (Or put some identifier of your choice)

# Exit and Save
```

Disable following options

```bash
scripts/config --disable SYSTEM_TRUSTED_KEYS
scripts/config --disable SYSTEM_REVOCATION_KEYS

# Note => while "make" command is running, if any certificate related question arises, then simply hit Enter.
```

All config done. Time to build the kernel

```bash
time make -j6 2>&1 | tee build-0.log
```

If all goes well, you should see following command at end

```bash
Kernel: arch/x86/boot/bzImage is ready  (#2)
```


```bash
```


```bash
```


```bash
```

```bash
```

## Notes
- I used -4 option in few of the commands above. This instruct to use IPv4. This is because I was facing few issues with IPv6.