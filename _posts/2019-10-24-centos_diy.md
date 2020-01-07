---
layout: post
title: CentOS系统定制
date: 2019-10-24
tags: linux
---

<!-- TOC -->

- [1. 环境准备](#1-环境准备)
- [2. ISO生成](#2-iso生成)
- [3. 无人值守ISO制作](#3-无人值守iso制作)
    - [3.1 获取ks.cfg模板](#31-获取kscfg模板)
    - [3.2 以anaconda-ks.cfg为模板，进行修改](#32-以anaconda-kscfg为模板进行修改)
    - [3.3 将ks.cfg加入到ISO中](#33-将kscfg加入到iso中)

<!-- /TOC -->

# 1. 环境准备

```bash
# 1.宿主系统：CentOS7（随意）  
# 2.目标镜像：CentOS-7-x86_64-Minimal-1908.iso  
wget https://mirrors.tuna.tsinghua.edu.cn/centos/7/isos/x86_64/CentOS-7-x86_64-Minimal-1908.iso
# 3.制作工具：createrepo、mkisofs、isomd5sum、syslinux（在宿主系统上安装）
yum install createrepo mkisofs isomd5sum syslinux -y
```

# 2. ISO生成

```bash
# 1.准备一个文件夹，将CentOS-7-x86_64-Minimal-1908.iso镜像内容全部拷贝出来
mount CentOS-7-x86_64-Minimal-1908.iso /mnt/centos  # 先挂在，才能拷贝
mkdir ~/KeenCentOS
cp -r /mnt/centos/* ~/KeenCentOS/
cd ~/KeenCentOS

# 2.修改isolinux/isolinux.cfg，增加自己的启动引导
------------
label linux
  menu label ^Install CentOS 7
  kernel vmlinuz
  append initrd=initrd.img inst.stage2=hd:LABEL=CentOS\x207\x20x86_64 quiet

label KeenLinux
  menu label Install ^Keen Custom CentOS7
  kernel vmlinuz
  append initrd=initrd.img inst.stage2=hd:LABEL=KeenCentOS7 quiet
------------

# 3.制作ISO
# -D：ISO9660要求的，禁止深度重定向，不晓得啥意思
# -r：允许常规的UNIX的文件名和Rock Ridge extension的一些属性，并开放全部文件的读权限
# -V：设置Volume ID，作为将来mount的名字（也是isolinux.cfg中LABEL后的名字，append initrd=initrd.img inst.stage2=hd:LABEL=KeenCentOS7 quiet）
# -v：调试日志
# -cache-inodes：需要检查硬链接，我们这儿应该不需要
# -J：通过Joliet扩展，允许使用微软的UCS-2的文件名
# -l：使用ISO 9660 32字符长度的文件名。
# -b：指定由光盘引导的开机镜像文件（El Torito这种类型）
# -c：配合-b使用（El Torito这种类型的东西，会有个分类索引文件，这种规则必须这么使用的）
# -no-emul-boot：ISOLINUX和GRUB2必选项，系统将会直接加载和执行镜像，而不做任何磁盘模拟，类似建虚拟存储空间
# -boot-load-size：-no-emul-boot一般这里就填4，表示启动镜像有多少扇区会被BIOS加载（4个扇区相当于2K的数据）
# -boot-info-table：ISOLINUX和GRUB2必选项，会使用启动信息表来填补启动镜像，大概是说有一些校验、长度、描述符等
# -o：输出文件路径
mkisofs -D -r -V "KeenCentOS7" -v -cache-inodes -J -l -b isolinux/isolinux.bin -c isolinux/boot.cat -no-emul-boot -boot-load-size 4 -boot-info-table -o ~/KeenCentOS7_V1.iso ~/KeenCentOS

# 4.给ISO添加MD5校验
implantisomd5 ~/KeenCentOS7_V1.iso
```

# 3. 无人值守ISO制作

我们下载的官方CentOS镜像ISO，无论是否是Minimal版本，默认都是带界面的，需要用户交互来设置各个选项。我们这里采用CentOS支持的Anaconda-KickStart，来实现无人值守的安装包制作。  

## 3.1 获取ks.cfg模板

我们可以先用我们的母盘CentOS-7-x86_64-Minimal-1908.iso在虚拟机中，安装一个系统，然后在~/目录中，会存在一个anaconda-ks.cfg，它就是我们接下来使用的模板

## 3.2 以anaconda-ks.cfg为模板，进行修改

参考<https://fedoraproject.org/wiki/Anaconda/Kickstart/zh-cn>
```bash
#version=DEVEL
# --------------------------------------------------
# System authorization information
# 设置系统的授权选项，默认情况，密码会被加密，但是不会放在/etc/shadow（或者/etc/shadow-）文件中，而是放在/etc/passwd（或者/etc/passwd-）文件中。由于shadow只能root权限看到，passwd是任何用户都能看见，为了安全起见，通常需要开启shadow。
auth --enableshadow --passalgo=sha512
# --------------------------------------------------
# Use CDROM installation media
# 这里表示的是安装类型，可以是cdrom、harddrive、liveimg、nfs、url
# cdrom一般表示从系统上的第一个CD-ROM/DVD驱动安装，一般是虚拟光驱加载ISO安装，或者真实光盘安装。虚拟机安装经常采用的方式。
# harddrive一般表示本地驱动器中包含ISO镜像的目录安装，通常是ISO直接解压到本地文件系统中，直接从这里安装。格式：harddrive --partition=hdb2 --dir=/tmp/myiso
# liveimg这个一般只作为辅助的动作，相当于装一个PE镜像系统，然后通过它来引导安装实际操作系统。它是安装一个磁盘镜像而不是软件包。镜像可以是Live iso上的squashfs.img，也可以是任何可以被安装介质挂载的文件系统（例如ext4）。如果该镜像包含/LiveOS/*.img（这是squashfs.img的构成），LiveOS中的第一个*.img将会被挂载，并用来安装目标系统。格式：liveimg --url=<url>
# nfs表示从指定的NFS（网络文件系统）服务器安装。可以是分解的安装树或者ISO镜像的目录。如果是后者，和上面所描述的硬盘驱动器安装方式所遵循的规则一样，必须提供install.img。格式：nfs --server=nfsserver.example.com --dir=/tmp/install-tree
# url表示从FTP或HTTP服务器安装。对于docker镜像，一般采用此方式，例如：url --url="http://mirrors.kernel.org/centos/7/os/x86_64/"
cdrom
# --------------------------------------------------
# Use graphical install
# 采用文本还是图形界面，完成安装。默认都是有图形界面的，如果不想要这些界面，就使用text模式
#graphical
text
# --------------------------------------------------
# Run the Setup Agent on first boot
# 决定是否在第一次启动系统后，运行设置代理程序，去设置语言、鼠标、键盘、根密码、安全级别、时区以及网络配置选项。一般我们都需要开启这个
firstboot --enable
# --------------------------------------------------
# 控制安装过程中，仅能使用的系统磁盘。要注意的是，磁盘类型。linux系统中，每一个设备，对应一个文件，不同设备，都有标准命名。
# 常见磁盘类型：
#   IDE硬盘机：/dev/hd[a-d]（根据顺序，依次hda、hdb、hdc...）
#   SCSI/SATA/USB硬盘机：/dev/sd[a-p]
#   软盘驱动器：/dev/fd[0-1]
#   CDROM/DVDROM：/dev/cdrom
# 由于这里边需要预先填写，因此等于是需要预先知道设备类型。一般而言，设备类型都是sda，有时候可以看到vda，虚拟的磁盘类型，也要注意
ignoredisk --only-use=sda
# --------------------------------------------------
# Keyboard layouts
# 设置键盘类型，必选项
keyboard --vckeymap=cn --xlayouts='cn'
# System language
# 设置系统语言，必选项
lang zh_CN.UTF-8
# --------------------------------------------------
# Network information
# 设置网络。
# 这里边使用了--no-activate，如果在安装过程中需要网络，例如从网络安装，这里边需要设置成--activate激活网络设备
# --bootproto一般选择dhcp或者static，设置静态ip还是动态ip。设置静态ip示例：network --bootproto=static --ip=10.0.2.15 --netmask=255.255.255.0 --gateway=10.0.2.254 --nameserver=10.0.2.1
# --device指定要配置的设备。ens33是centos7的命名方式
#network  --bootproto=dhcp --device=ens33 --onboot=off --ipv6=auto --no-activate
network  --bootproto=dhcp --device=ens33 --onboot=on --ipv6=auto --no-activate
network  --hostname=localhost.localdomain
# --------------------------------------------------
# Root password
rootpw --iscrypted $6$kXSuQUjnucKrLggj$YNiaXuOOYj0yBR4q2bxaM.8yBje.NmSRcQzi/6bxQpm2WBONQuJUMuCI2ltqKj1OUuuhCilF3vQ6n0RESEIa0/
# --------------------------------------------------
# System services
# 允许时间同步服务启动，并设置时区
services --enabled="chronyd"
# System timezone
timezone Asia/Shanghai --isUtc
# --------------------------------------------------
# System bootloader configuration
# 设置引导程序如何被安装
# --append指定内核vmlinuz参数，默认参数rhgb quiet（rhgb表示redhat graphics boot，图片代理启动过程中的文本，不过要取决于内核的支持，minimal版本内核不支持）
# --location指定引导记录写入位置
bootloader --append=" crashkernel=auto" --location=mbr --boot-drive=sda
# --------------------------------------------------
# Partition clearing information
#clearpart --none --initlabel
clearpart --all --initlabel --drives=sda
# 自动创建分区--一个根分区(/)、一个swap分区，以及一个适合体系架构(architecture)的boot分区。如果磁盘驱动器足够大，也会创建/home分区
autopart --type=lvm
#这里改成自己创建分区，主动将分区挂在给/data，不过要非常清楚磁盘大小
#part biosboot --fstype=biosboot --size=1
#part /boot --fstype ext4 --size=500
#part swap --size=4096
#part / --fstype ext4 --size=102400
#part /data --fstype ext4 --size=51200 --grow
reboot --eject
# --------------------------------------------------
# 这里边，包含了需要安装的包
%packages
# 以组的形式添加包
@^minimal
@core
# 以单独的rpm包形式，添加
chrony
kexec-tools
#额外添加
net-tools
openssl
%end

%addon com_redhat_kdump --enable --reserve-mb='auto'
%end

%anaconda
pwpolicy root --minlen=6 --minquality=1 --notstrict --nochanges --notempty
pwpolicy user --minlen=6 --minquality=1 --notstrict --nochanges --emptyok
pwpolicy luks --minlen=6 --minquality=1 --notstrict --nochanges --notempty
%end
# --------------------------------------------------
# 安装系统初期做的事情（ks.cfg加载后，已完成了语言、键盘、基本网络IP等配置，但是还未完成磁盘分区）
%pre
%end
# --------------------------------------------------
# 安装后执行脚本
# --nochroot：表示还未切换到新系统时执行的脚本，不带这个选项，表示已经切换到新系统后执行的脚本
# --log=/tmp/ks-post.log：日志存储到文件
%post --nochroot
%end

%post
%end
```

%pre的示例：

```bash
%pre
#!/bin/bash
hds=""
mymedia=""

for file in /sys/block/sd*; do
hds="$hds $(basename $file)"
done

set $hds
numhd=$(echo $#)

drive1=$(echo $hds | cut -d' ' -f1)
drive2=$(echo $hds | cut -d' ' -f2)


if [ $numhd == "2" ]  ; then
echo "#partitioning scheme generated in %pre for 2 drives" > /tmp/part-include
echo "clearpart --all" >> /tmp/part-include
echo "part /boot --fstype ext4 --size 512 --ondisk sda" >> /tmp/part-include
echo "part / --fstype ext4 --size 10000 --grow --ondisk sda" >> /tmp/part-include
echo "part swap --recommended --ondisk $drive1" >> /tmp/part-include
echo "part /home --fstype ext4 --size 10000 --grow --ondisk sdb" >> /tmp/part-include
else
echo "#partitioning scheme generated in %pre for 1 drive" > /tmp/part-include
echo "clearpart --all" >> /tmp/part-include
echo "part /boot --fstype ext4 --size 521" >> /tmp/part-include
echo "part swap --recommended" >> /tmp/part-include
echo "part / --fstype ext4 --size 2048" >> /tmp/part-include
echo "part /home --fstype ext4 --size 2048 --grow" >> /tmp/part-include
fi
%end
```

%post示例

```bash
%post --nochroot
# chroot可以将当前执行环境，切换成对应的目录，以对应目录，作为根目录
chroot /mnt/sysimage sh /root/dosomething.sh
```

## 3.3 将ks.cfg加入到ISO中

```bash
# 1.将文件放入isolinux目录中（isolinux/ks.cfg）
# 2.修改isolinux.cfg，将ks.cfg作为参数传递给vmlinuz
label KeenLinux   （用来指定系统启动时的菜单选项）
  menu label Install ^Keen Custom CentOS7
  kernel vmlinuz
  append initrd=initrd.img inst.ks=hd:LABEL=KeenCentOS7:/isolinux/ks.cfg inst.stage2=hd:LABEL=KeenCentOS7 quiet
```
