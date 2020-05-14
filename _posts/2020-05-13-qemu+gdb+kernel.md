---
layout:     post
title:      setup ubuntu qemu guest with custom kernel and debug it with gdb
subtitle:   ubuntu host and guest, qemu vm, x86-64 kernel 
date:       2020-05-13
author:     jiayy
header-img: img/post-android-bulletin-2018.jpg
catalog: true
tags:
    - debug
    - gdb
---

需要在多个linux内核版本上调试代码，因此需要快速搭建基于多种内核版本的调试环境

## Kernel 

用主线代码, git 下载

```c
git clone https://github.com/torvalds/linux.git $KERNEL
```

用 ubuntu 维护的内核代码, 下载对应版本的源码包并解压,　比如 ubuntu20.04 对应的源码包为　linux-source-5.4.0
[focal linux-source-5.4.0](http://archive.ubuntu.com/ubuntu/pool/main/l/linux/linux_5.4.0.orig.tar.gz)

如果是调试内核漏洞利用，则推荐使用主线的源码，因为对应用层依赖不多，如果是调试内核模块或者工具，则推荐使用ubuntu维护的源码包，因为很多依赖的库需要从ubuntu的源下载，内核跟ubuntu保持一致比较省事

编译时要保证　CONFIG_DEBUG_INFO 打开，最好是同时让　CONFIG_RANDOMIZE_BASE 关闭，这个选项会影响调试

```bash
cd $KERNEL
make mrproper
export CC=/your/gcc
make defconfig
make kvmconfig
# modify some config
# CONFIG_DEBUG_INFO=y
# CONFIG_RANDOMIZE_BASE=n
make olddefconfig
make
```


## Image

使用　debootstrap 建立根系统镜像

```bash
sudo apt-get install debootstrap
```

拷贝下面的脚本到文件　create-image.sh, 然后运行，可以得到镜像文件　eoan.img (ubuntu19.10) 或者　focal.img (ubuntu20.04)

```bash

#!/bin/bash
# create-image.sh creates a minimal ubuntu image suitable for kernel debugging.
# Copyright 2020 chengjia4574@gmail.com

set -eux

RELEASE=eoan  # ubuntu19.10
# RELEASE=focal # ubuntu20.04

DIR=chroot-$RELEASE
sudo rm -rf $DIR
sudo mkdir -p $DIR
sudo chmod 0755 $DIR

INSTALL_PKGS=openssh-server,curl,tar,gcc,libc6-dev,time,strace,sudo,less,psmisc,net-tools,build-essential,vim,git,make

sudo debootstrap --include=$INSTALL_PKGS --components=main,contrib,non-free $RELEASE $DIR

# Set some defaults and enable promtless ssh to the machine for root.
sudo sed -i '/^root/ { s/:x:/::/ }' $DIR/etc/passwd
echo 'T0:23:respawn:/sbin/getty -L ttyS0 115200 vt100' | sudo tee -a $DIR/etc/inittab
# 必须关闭 kaslr 才可以正常调试内核
echo 'GRUB_CMDLINE_LINUX_DEFAULT="nokaslr"' | sudo tee -a $DIR/etc/default/grub
# /etc/network/interfaces 已经被 ubuntu 放弃，改为了　netplan 方式
printf 'network:\n    version: 2\n    renderer: networkd\n    ethernets:\n        enp0s3:\n            dhcp4: true\n' | sudo tee -a $DIR/etc/netplan/01-network-manager-all.yaml
echo '/dev/root / ext4 defaults 0 0' | sudo tee -a $DIR/etc/fstab
echo 'debugfs /sys/kernel/debug debugfs defaults 0 0' | sudo tee -a $DIR/etc/fstab
echo 'securityfs /sys/kernel/security securityfs defaults 0 0' | sudo tee -a $DIR/etc/fstab
echo 'configfs /sys/kernel/config/ configfs defaults 0 0' | sudo tee -a $DIR/etc/fstab
echo 'binfmt_misc /proc/sys/fs/binfmt_misc binfmt_misc defaults 0 0' | sudo tee -a $DIR/etc/fstab
echo "kernel.printk = 7 4 1 3" | sudo tee -a $DIR/etc/sysctl.conf
echo 'debug.exception-trace = 0' | sudo tee -a $DIR/etc/sysctl.conf
echo "net.core.bpf_jit_enable = 1" | sudo tee -a $DIR/etc/sysctl.conf
echo "net.core.bpf_jit_kallsyms = 1" | sudo tee -a $DIR/etc/sysctl.conf
echo "net.core.bpf_jit_harden = 0" | sudo tee -a $DIR/etc/sysctl.conf
echo "kernel.softlockup_all_cpu_backtrace = 1" | sudo tee -a $DIR/etc/sysctl.conf
echo "kernel.kptr_restrict = 0" | sudo tee -a $DIR/etc/sysctl.conf
echo "kernel.watchdog_thresh = 60" | sudo tee -a $DIR/etc/sysctl.conf
echo "net.ipv4.ping_group_range = 0 65535" | sudo tee -a $DIR/etc/sysctl.conf
echo -en "127.0.0.1\tlocalhost\n" | sudo tee $DIR/etc/hosts
echo "nameserver 8.8.8.8" | sudo tee -a $DIR/etc/resolve.conf
echo "jiayy" | sudo tee $DIR/etc/hostname
ssh-keygen -f $RELEASE.id_rsa -t rsa -N ''
sudo mkdir -p $DIR/root/.ssh/
cat $RELEASE.id_rsa.pub | sudo tee $DIR/root/.ssh/authorized_keys

# Build a disk image
dd if=/dev/zero of=$RELEASE.img bs=1G seek=20 count=1
sudo mkfs.ext4 -F $RELEASE.img
sudo mkdir -p /mnt/$DIR
sudo mount -o loop $RELEASE.img /mnt/$DIR
sudo cp -a $DIR/. /mnt/$DIR/.
sudo umount /mnt/$DIR

```

## Qemu

先安装　qemu

```bash
sudo apt-get install qemu-system-x86
```

使用下面的脚本启动虚拟机

```bash
qemu-system-x86_64   \
-smp 4 \
-enable-kvm -m 2G -nographic \
-kernel /path-to/linux/arch/x86/boot/bzImage  \
-append "console=ttyS0 root=/dev/sda earlyprintk=serial rw nokaslr"  \
-drive file=/path-to/focal.img,format=raw   \
-device e1000,netdev=t0  \
-pidfile vm.pid \
-netdev user,id=t0,hostfwd=tcp::10022-:22 -s -S  
```

注意内核启动选项必须增加　nokaslr 关闭地址随机化，否则就需要kernel编译时关闭　CONFIG_RANDOMIZE_BASE, kaslr 会导致断点出问题无法调试

如果需要杀死虚拟机，用下面的命令:

```bash
kill $(cat vm.pid)
```

## Ssh

登录时用秘钥文件登录

```bash
ssh -i focal.id_rsa -p 10022 -o "StrictHostKeyChecking no" root@localhost
#ssh -i eoan.id_rsa -p 10022 -o "StrictHostKeyChecking no" root@localhost
```

登录guest后首先替换源　/etc/apt/source.list , 然后 apt update

如果需要在　guest 编译驱动或者内核源码，需要安装依赖

```bash
apt-get build-dep linux
apt-get install libncurses-dev flex bison openssl libssl-dev dkms libelf-dev libudev-dev libpci-dev libiberty-dev autoconf
```

## Gdb 

```bash
(gdb) target remote :1234
(gdb) b xxx_function
(gdb) c
```

# Ref

[setup_ubuntu-host_qemu-vm_x86-64-kernel.md](https://github.com/google/syzkaller/blob/master/docs/linux/setup_ubuntu-host_qemu-vm_x86-64-kernel.md)
[booting-a-custom-linux-kernel-in-qemu-and-debugging-it-with-gdb](http://nickdesaulniers.github.io/blog/2018/10/24/booting-a-custom-linux-kernel-in-qemu-and-debugging-it-with-gdb/)

