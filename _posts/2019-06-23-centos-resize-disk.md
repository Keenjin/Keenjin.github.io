---
layout: post
title: CentOS文件系统扩容
date: 2019-06-23
tags: linux
---

<!-- TOC -->

- [1. 磁盘空间不足，查看剩余空间](#1-磁盘空间不足查看剩余空间)
- [2. 先删除/root目录下到一点东西，以便后续到缓存](#2-先删除root目录下到一点东西以便后续到缓存)
- [3. 扩展vbox到虚拟机空间大小](#3-扩展vbox到虚拟机空间大小)
- [4. 磁盘分区](#4-磁盘分区)
- [5. 添加新LVM到已有LVM扩容](#5-添加新lvm到已有lvm扩容)
- [6. 文件系统扩容](#6-文件系统扩容)

<!-- /TOC -->

# 1. 磁盘空间不足，查看剩余空间

```bash
[root@localhost /]# df -h
文件系统                 容量  已用  可用 已用% 挂载点
/dev/mapper/centos-root  6.2G  6.2G   20K  100% /
devtmpfs                 484M     0  484M    0% /dev
tmpfs                    496M     0  496M    0% /dev/shm
tmpfs                    496M  6.7M  489M    2% /run
tmpfs                    496M     0  496M    0% /sys/fs/cgroup
/dev/sda1               1014M  133M  882M   14% /boot
tmpfs                    100M     0  100M    0% /run/user/0
```

# 2. 先删除/root目录下到一点东西，以便后续到缓存

```bash
[root@localhost /]# cd root
[root@localhost ~]# ls
anaconda-ks.cfg  linux-5.1.14  linux-5.1.14.tar.xz  qemu-4.0.0  qemu-4.0.0.tar.xz
[root@localhost ~]# rm -rf linux-5.1.14
[root@localhost ~]# df -h
文件系统                 容量  已用  可用 已用% 挂载点
/dev/mapper/centos-root  6.2G  4.7G  1.6G   76% /
devtmpfs                 484M     0  484M    0% /dev
tmpfs                    496M     0  496M    0% /dev/shm
tmpfs                    496M  6.7M  489M    2% /run
tmpfs                    496M     0  496M    0% /sys/fs/cgroup
/dev/sda1               1014M  133M  882M   14% /boot
tmpfs                    100M     0  100M    0% /run/user/0
```

# 3. 扩展vbox到虚拟机空间大小

```bash
# 如果发现vbox虚拟机分配到磁盘空间太小，可以尝试使用以下命令修改。注意，单位上MByte
VBoxManage list hdds
VBoxManage modifyhd 74fda9b9-81c3-4b15-a51a-dab1b3b12001 --resize 35960

# 查看调整后到磁盘状态
[root@localhost ~]# fdisk -l

磁盘 /dev/sda：37.7 GB, 37706792960 字节，73646080 个扇区
```

此时，centos到/root目录不会自动调整大小，需要手动调整root大小

# 4. 磁盘分区

```bash
# 这里的每一步，要看清选项，不同系统或者不同fdisk版本，选项可能不一样。
[root@localhost ~]# fdisk /dev/sda

p       # 查看已分区数量（我看到有两个 /dev/sda1 和/dev/sda2） 
n       # 新增加一个分区 
p       # 分区类型，选择主分区 
        # 分区号选3（1和2已占用，见上） 
回车     # 默认选择（起始扇区） 
回车     # 默认选择（结束扇区） 
t       # 修改分区类型 
        # 选分区3（要修改的分区号，上面新增的） 
8e      # 修改为LVM（这里最好使用L去list所有type，找到LVM。不一定8e就是LVM） 
w       # 写分区表，完成后退出fdisk命令

# 重启机器（注意，这一步要慎重，对于线上系统而言，可能重启系统会导致一些问题。这里可以推荐使用partprobe -s来动态扫描分区表），并格式化分区
[root@localhost ~]# reboot
[root@localhost ~]# mkfs.ext3 /dev/sda3
```

# 5. 添加新LVM到已有LVM扩容

```bash
[root@localhost ~]# lvm
lvm> pvcreate /dev/sda3         # 初始化刚才到分区
lvm> vgdisplay                  # 查看VG Name
  --- Volume group ---
  VG Name               centos
lvm> vgextend centos /dev/sda3  # VG Name上centos，将初始化过的分区加入到虚拟卷组
lvm> vgdisplay                  # 查看剩余空间
Free  PE / Size       6941 / 27.11 GiB
lvm> lvextend -L +26G /dev/mapper/centos-root   # 扩展已有卷的容量
lvm> pvdisplay                  # 查看卷容量，看到成功了
  --- Physical volume ---
  PV Name               /dev/sda2
  VG Name               centos
  PV Size               <7.00 GiB / not usable 3.00 MiB
  Allocatable           yes (but full)
  PE Size               4.00 MiB
  Total PE              1791
  Free PE               0
  Allocated PE          1791
  PV UUID               NXjrej-QnyY-6FeH-rMTd-U1pC-6LMT-deK2C0

  --- Physical volume ---
  PV Name               /dev/sda3
  VG Name               centos
  PV Size               <27.12 GiB / not usable 4.00 MiB
  Allocatable           yes
  PE Size               4.00 MiB
  Total PE              6941
  Free PE               285
  Allocated PE          6656
  PV UUID               dcy5yt-Ylil-Rn3J-ktEM-1TVO-buv3-5db9RF
lvm> quit                       # 退出
```

# 6. 文件系统扩容

```bash
# 优先使用xfs_growfs命令；如果出现“is not a mounted XFS filesystem”，则尝试使用resize2fs命令：resize2fs /dev/mapper/centos-root
[root@localhost ~]# xfs_growfs /dev/mapper/centos-root
[root@localhost ~]# df -h
文件系统                 容量  已用  可用 已用% 挂载点
/dev/mapper/centos-root   33G  4.7G   28G   15% /
devtmpfs                 484M     0  484M    0% /dev
tmpfs                    496M     0  496M    0% /dev/shm
tmpfs                    496M  6.6M  490M    2% /run
tmpfs                    496M     0  496M    0% /sys/fs/cgroup
/dev/sda1               1014M  133M  882M   14% /boot
tmpfs                    100M     0  100M    0% /run/user/0
```