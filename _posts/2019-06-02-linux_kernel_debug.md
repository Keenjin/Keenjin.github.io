---
layout: post
title: Linux内核调试方法 - 使用qemu + gdb准备调试环境
date: 2019-06-02
tags: linux
---

# Linux 内核调试方法

## 1 准备环境

注意：由于linux内核编译依赖linux系统特性，所以编译工作需要放在linux系统上进行  

宿主机：mac下Parallel Desktop下的Centos 7

## 2 安装qenu虚拟机

```bash
$ yum install qemu-kvm qemu-img virt-manager libvirt libvirt-python libvirt-client virt-install virt-viewer
$ yum install qemu
```

## 3 Centos中编译linux内核

```bash
# 下载linux内核，这里下载最新稳定版本5.1.6
$ mkdir ～/LinuxKernel
$ cd LinuxKernel
$ wget https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.1.6.tar.xz
$ tar -xvf linux-5.1.6.tar
$ cd linux-5.1.6
$ make i386_defconfig               
# 可能出现时间戳不一致的警告，使用touch 对应文件，即可更新时间戳。如果太多的情况下，使用：find ./ * -exec touch {} \;   来递归修改

$ make
# 这期间，缺少某个命令，则安装某个命令即可，通常直接yum install xxx（提示的缺失命令名）
# 如果出现ssh相关的，还需要安装openssl相关：yum install openssl-devel（ubuntu下上libssh-dev）
# 最后编译生成的二进制镜像：arch/x86/boot/bzImage
```

## 4 尝试使用qenu运行linux内核

```bash
$ qemu-system-x86_64 arch/x86/boot/bzImage
```

## 5 准备调试，需要重新用调试选项编译linux内核

```bash
make menuconfig（选择kernel hacking-->compile with debug info）
make -j20

# 生成的镜像，还上在arch/x86_64/boot/bzImage

# 从centos的iso里面，获取到isolinux/initrd.img，拷贝到linux内核文件夹

# qemu带调试启动，在启动时暂停
qemu-system-x86_64 -initrd initrd.img -kernel arch/x86/boot/bzImage -s -S

# 启动调试器gdb
gdb vmlinux
target remote:1234

# 至此，调试准备ok了

```