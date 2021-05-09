---
layout: post
title: 【系统原理】Centos系统安装启动原理
date: 2019-10-16
tags: 系统原理
---

# 一些资料

kickstart原理：<https://fedoraproject.org/wiki/Anaconda/Kickstart/zh-cn>  

# 可启动盘原理

Step1：BIOS开机检查启动盘的第17个扇区，run初始指令  
Step2：17扇区的汇编代码，会查找到启动记录卷描述表（Boot Record Volume Descripter）数据结构指针地址，并根据这个索引表，继续查找启动目录（Booting Catalog）  
Step3：汇编代码继续执行，根据启动目录结构，拿到启动入口（Boot Entry），进而找到启动盘映像（Bootable Disk Image，例如isolinux）或启动引导文件  
Step4：继续执行启动引导的二进制中的汇编代码  

![png](/images/post/linux/booting.png)

# 安装包目录文件

使用的安装包：CentOS-7-x86_64-Minimal-1908.iso  

```bash
C:.
│  .discinfo
│  .treeinfo
│
├─EFI       # EFI引导使用的文件
│  │
│  └─BOOT
│      │  BOOTIA32.EFI
│      │  BOOTX64.EFI
│      │  grub.cfg
│      │  grubia32.efi
│      │  grubx64.efi
│      │  mmia32.efi
│      │  mmx64.efi
│      │
│      └─fonts
│              unicode.pf2
│
├─images    # 
│  │  efiboot.img       # 解压后，就是EFI/BOOT里面的东西
│  │
│  └─pxeboot            # 提供网络接口启动计算机的机制，不依赖本地数据存储设备或已安装的操作系统
│          initrd.img
│          vmlinuz
│
├─isolinux  # 一个轻量级的linux系统，硬件无关性较强，用来做引导安装使用，相当于PE系统
│      boot.cat
│      boot.msg
│      grub.conf
│      initrd.img       # 内存映像文件（initial ramdisk）。它本质就是一个微内核linux系统，内部打包了各种驱动程序，它是硬件兼容性好的最关键点
│      isolinux.bin
│      isolinux.cfg
│      memtest
│      splash.png
│      vesamenu.c32
│      vmlinuz
│
├─LiveOS
│      squashfs.img
│
├─Packages
│      acl-2.2.51-14.el7.x86_64.rpm
│      aic94xx-firmware-30-6.el7.noarch.rpm
│      aide-0.15.1-13.el7.x86_64.rpm
│      alsa-firmware-1.0.28-2.el7.noarch.rpm
|      ......
│
└─repodata
       16890efb08ba2667b3cfd83c4d234d5fabea890e6ed2ade4d4d7adec9670a9a5
       3654075e05ea799f20d35bc250759378ae24bbba929973efc9ab809b182a4c7c
       4af1fba0c1d6175b7e3c862b4bddfef93fffb84c37f7d5f18cfbff08abc47f8a
       521f322f05f9802f2438d8bb7d97558c64ff3ff74c03322d77787ade9152d8bb
       5a783c36d40a53713ff101426228dc9d0459e7246e1a1bdb530c97a478784d53
       7b2c05dfb46ff4a36822aa3c9d3484a7bdd3f356f4638ddf239f89d5f8f9bca1
       83b61f9495b5f728989499479e928e09851199a8846ea37ce008a3eb79ad84a0
       848a2b870c34dbf5186e3a09d6e0105959117b6864a5715df3cf3a40d8b04e60
       d4de4d1e2d2597c177bb095da8f1ad794d69f76e8ac7ab1ba6340fdd0969e936
       fa095a8e52d5d2f81f1947b3a25f10608250d40b14b579469a93c28fced5d60e
       repomd.xml
       repomd.xml.asc

```

# linux文件系统作用

```bash
# 分区相关
/etc/mtab：记录了当前挂在的分区信息
/etc/fstab：开机启动后，可以根据此配置来自动挂在分区


```