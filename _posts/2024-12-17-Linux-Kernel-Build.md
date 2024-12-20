---
title: "A Guide to Building Linux Kernel 6.x"
last_modified_at: 2024-12-20T18:55:00+05:30
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

This blog post provides a step-by-step tutorial on compiling and booting from 
Linux Kernel 6.x. We will use a virtual environment for safety. This blog 
post is ideal for developers and curious learners.

I assume reader has basic understanding of Linux OS.

## Setting up a playground

I am using [Oracle VirtualBox v7.0](https://www.virtualbox.org/wiki/Linux_Downloads) on the Ubuntu (v24.04) x86_64 Host. Download [Ubuntu 20.04 iso](https://hr.releases.ubuntu.com/20.04.6/ubuntu-20.04.6-desktop-amd64.iso). In Virtualbox, create a new VM machine by providing the iso image. Select ```Skip Unattended Installation``` option. This is optional, but I prefer to choose this to observe what's going on.

Minimum resources to allocate are - 
- 2 vCPUs 
- 2 GB RAM
- 100 GB o Hard Disk space.

Once finished, Ubuntu OS will boot and provide option to install it. During installation, choose ```minimal installation``` option and do not install ```extra third party softwares```. This is because we simply don't need these things atm.

Post starting the VM, check the kernel version. I see following at my end.

```bash
uname -r 
5.15.0-126-generic

uname -a
Linux kernel0 5.15.0-126-generic #136~20.04.1-Ubuntu SMP Thu Nov 14 16:38:05 UTC 2024 x86_64 x86_64 x86_64 GNU/Linux
```

Do Update and upgrade

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
Then install the packages required to build the kernel

```bash
wget -4 https://raw.githubusercontent.com/simplyatul/vagrant-vms/refs/heads/main/tools-0-kernel-dev.sh
chmod +x tools-0-kernel-dev.sh
sudo ./tools-0-kernel-dev.sh
```
## Kernel code config
Let us now checkout stable kernel version

```bash
git clone -4 --depth=1 --branch v6.8.12 git://git.kernel.org/pub/scm/linux/kernel/git/stable/linux-stable.git

cd linux-stable
```

To build the kernel, ```.config``` file is required. This file contains kernel configurations. I used my host's config file for this

```bash
# Copy /boot/config-6.8.0-48-generic from my host PC to VM's /tmp directory

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

## Kernel build
Gr8, all config is done. Time to build the kernel

```bash
time make -j6 2>&1 | tee build-0.log
```

If all goes well, you should see a statement like below at the end

```bash
Kernel: arch/x86/boot/bzImage is ready  (#2)
```
Check the vmlinux file has created 

```bash
ls -lh vmlinux
-rwxrwxr-x 1 kernel0 kernel0 469M Dec 17 16:57 vmlinux
```

## Kernel install

On Ubuntu, you can run following to install newly built kernel
```bash
sudo make -d modules_install install 2>&1 | tee make-install-0.log
```
Above command installs the kernel to ```/boot/```, installs modules to 
```/lib/modules/X.Y.Z/``` (where X.Y.Z is 6.8.12-at-bld0 in our case), and 
updates file ```/boot/grub/grub.conf```.

Now update following configs in ```/etc/default/grub``` file

```bash
GRUB_TIMEOUT_STYLE=hidden
GRUB_TIMEOUT=0
```
to

```bash
#GRUB_TIMEOUT_STYLE=hidden
GRUB_TIMEOUT=15
```

This ensures you see boot option for 15 sec before it automatically boots.

At end, update the grub

```bash
sudo update-grub2
```

Post reboot, you should see newly installed kernel is in action

```bash
uname -r
6.8.12-at-bld0

uname -a
Linux kernel0 6.8.12-at-bld0 #2 SMP PREEMPT_DYNAMIC Tue Dec 17 16:56:45 IST 2024 x86_64 x86_64 x86_64 GNU/Linux
```
Note the version number ```6.8.12-at-bld0``` has suffix ```-at-bld0``` which 
we had entered during kernel code config step above. Every time you build 
the new kernel, ensure to update this suffix.

## Notes
- I used ```-4``` option in few of the commands above. This instruct to use IPv4. This is because I was facing few issues with IPv6.

## References
1. https://kernelnewbies.org/KernelBuild
2. Book: [Linux Kernel Programming][1] (Second Edition) By Kaiwan N. Billimoria
3. [Kernel Workspace Setup](http://www.packtpub.com/sites/default/files/downloads/9781803232225_Online_Chapter.pdf)

[1]:https://www.oreilly.com/library/view/linux-kernel-programming/9781803232225/?_gl=1*1k3qty*_ga*MjA5Mzk2NjgzMC4xNzM0NDUzNDQ0*_ga_092EL089CH*MTczNDQ1MzQ0My4xLjEuMTczNDQ1MzQ0Ny41Ni4wLjA.