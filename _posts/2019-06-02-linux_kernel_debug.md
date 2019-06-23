---
layout: post
title: Linux内核调试方法
date: 2019-06-02
tags: linux
---

# Linux 内核调试方法

## 1 准备环境

注意：由于linux内核编译依赖linux系统特性，所以编译工作需要放在linux系统上进行  

宿主机：mac  
virtual box  
CentOS-7-x86_64-Minimal-1810.iso（配置网络，参考linux_cmd章节）

## 2 安装编译qenu虚拟机

```bash
# 我们要使用到是qemu-system，centos上只有yum -y install qemu-kvm。所以采用源码编译
wget https://download.qemu.org/qemu-4.0.0.tar.xz
tar -xvf qemu-4.0.0.tar.xz
yum -y install yum install gzlib-devel glib2-devel pixman-devel gcc
yum -y install zlib*
cd /qemu-4.0.0
./configure
make -j20
make install
```

```bash
# 如果发现vbox虚拟机分配到磁盘空间太小，可以尝试使用以下命令修改。注意，单位上MByte
VBoxManage list hdds
VBoxManage modifyhd 74fda9b9-81c3-4b15-a51a-dab1b3b12001 --resize 35960
```

## 3 Centos中编译linux内核

```bash
# 安装必要到环境集合
# 如果出现ssh相关的，还需要安装openssl相关：yum install openssl-devel（ubuntu下上libssh-dev）
yum -y install ncurses-devel flex bison openssl-devel elfutils-libelf-devel bc 

# 下载linux内核，这里下载最新稳定版本5.1.14
wget https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.1.14.tar.xz
tar -xvf linux-5.1.14.tar
cd linux-5.1.14
make menuconfig（选择kernel hacking-->compile with debug info）              
# 可能出现时间戳不一致的警告，使用touch 对应文件，即可更新时间戳。如果太多的情况下，使用：find ./ * -exec touch {} \;   来递归修改

# -j参数，上使用并行编译。编译耗时很长，所以有多cpu核心，就多搞几个编译
make -j20

# 最后编译生成的二进制镜像：arch/x86/boot/bzImage
```

## 4 busybox生成必要东西

```bash
wget https://busybox.net/downloads/busybox-1.31.0.tar.bz2

make menuconfig
# Busybox Settings -> Build Options -> Build Busybox as a static binary 编译成 静态文件
# Linux System Utilities -> [] Support mounting NFS file system 网络文件系统 Networking Utilities -> [] inetd (Internet超级服务器)

# 安装静态库，防止出现ld 未找到的错误
yum install glibc-static

make install

# 构建文件系统
cd _install
mkdir proc sys dev etc etc/init.d
vim etc/init.d/rcS

#!/bin/sh
mount -t proc none /proc
mount -t sysfs none /sys
/sbin/mdev -s

chmod +x etc/init.d/rcS
find . | cpio -o --format=newc > ../rootfs.img
```

## 5 准备调试

```bash
# 生成的镜像，还上在arch/x86/boot/bzImage
# 将rootfs.img，拷贝到linux内核文件夹
# -S表示：qemu带调试启动，在启动时暂停
# append里面，noapic可规避kernel因为apic导致的panic
# append里面，nokaslr可规避start_kernel无法中断问题
# append里面，console=ttyS0，可以让日志都显示在当前控制台
qemu-system-x86_64 -initrd rootfs.img -kernel arch/x86/boot/bzImage -append "console=ttyS0 root=/dev/ram rdinit=/sbin/init noapic nokaslr" --nographic -s -S

# 启动调试器gdb
gdb vmlinux
target remote:1234

# 注意，如果此时出现一堆Remote 'g' packet reply is too long错误，表示vmlinux跟当前gdb都architecture不匹配，需要设置
set architecture i386:x86-64

# 至此，调试准备ok了。设置启动断点。c表示continue，会在start_kernel下中断
b start_kernel
c
```
