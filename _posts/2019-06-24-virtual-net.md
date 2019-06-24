---
layout: post
title: 网络虚拟化
date: 2019-06-24
tags: 网络 虚拟化
---

# 虚拟网络原理

## 1 概述

网络虚拟化，是随着云计算时代，虚拟机盛行而流行起来的技术。由于云主机实体机一般而言比较贵，云时代虚拟机技术，能极大降低成本。对于主机而言，网络设备（物理网卡、交换机、路由器等）也是高成本的，不能为每一台虚拟机分配一个物理网卡设备，交换机或者路由器等，虚拟网络设备技术，正是为此而来。  

网络虚拟化，本质是把物理网卡，虚拟成多个网卡设备，可能是虚拟出一个交换机，可能是虚拟出一台路由器，或者虚拟出一个物理网卡等。VMWare和VBox，都自带这种虚拟网络设备功能。  

对于linux上，使用的网络虚拟化解决方案，有很多种：tap/tun、veth-pair、bridge、macvlan等

## 2 虚拟网络模型

有四种常见的网络模型：

- 桥接（Bridge Adapter）
- NAT（网络地址转换）
- 主机（Host-only Adapter）
- 内部网络（Internal）

### 2.1 桥接（Bridge Adapter）

#### 2.1.1 桥接模型

桥接网络模型就是使用虚拟交换机（Linux Bridge），将虚拟机和物理机连接起来。所有虚拟机连接到这个交换机bridge上，同时，物理主机也同样连接到这个交换机bridge上。虚拟机ip地址需要与主机在同一个网段，如果需要联网，则网关与DNS需要与主机网卡一致。  

由于所有虚拟设备和主机网卡设备，都连接在同一个交换机，相当于与主机所在的局域网，共同组成了局域网络，互联互通，因此这里虚拟机的ip地址，就一定得分配成不同于主机所在局域网的其他ip地址，防止冲突。这个也是这种模型的局限之处。

网络结构图如下：

![jpg](/images/post/virtual_network/bridge.jpg)

桥接模型的好处是简单方便，但是有一个明显问题：如果虚拟机太多，相当于增加局域网内的机器数，arp广播包就会很严重。适合小规模网络，不适合公司级网络，一般公司也会禁止此方法。

#### 2.1.2 桥接在linux上的实现

```bash
# 查看当前网络设备。
ifconfig

ens33: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.2.132  netmask 255.255.255.0  broadcast 192.168.2.255
        inet6 fe80::3668:cf64:7950:5d47  prefixlen 64  scopeid 0x20<link>
        inet6 fe80::3911:abf0:abbe:4579  prefixlen 64  scopeid 0x20<link>
        inet6 fe80::9422:95b:a1fc:ad87  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:49:2a:69  txqueuelen 1000  (Ethernet)
        RX packets 3008721  bytes 4200389677 (3.9 GiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 1703371  bytes 165210021 (157.5 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

# 网卡设备为ens33

# ----------------------------------
# 方法1：brctl添加网桥和接口（通过brctl添加的网桥和接口，重启系统后会被删除。）
# 添加网桥
brctl addbr br0
# 添加ip
ifconfig br0 192.168.2.132 netmask 255.255.255.0
# 清除ens33的ip地址
ifconfig ens33 0.0.0.0
# 桥接到物理网卡设备
brctl addif br0 ens33
# 到这一步，虚拟交换机br0就已经建立起来了，此刻只需要将kvm链接到br0上即可
virt-install --virt-type=kvm --name=centos7 --vcpus=2 --memory=4096 --location=/tmp/CentOS-7-x86_64-Minimal-1810.iso --disk path=/home/vms/centos7.qcow2,size=40,format=qcow2 --network bridge=br0 --graphics none --extra-args='console=ttyS0' --force
# ----------------------------------

# ----------------------------------
# 方法2：静态方法添加网桥和接口（重启系统依旧生效）
# 增加网桥配置文件
vim /etc/sysconfig/network-scripts/ifcfg-br0

TYPE=Bridge
BOOTPROTO=static      # <---------- 看这里。给网桥ip固定，对后续端口转发之类的比较有好处
DEFROUTE=yes
PEERDNS=yes
PEERROUTES=yes
IPV4_FAILURE_FATAL=no
DEVICE=br0
ONBOOT=yes
IPADDR=192.168.2.132
NETMASK=255.255.255.0
GATEWAY=192.168.2.2

# 修改ens33配置文件
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=dhcp      # <---------- 看这里。设置动态ip，将以下ip无效化。实际这里的ip地址也是无效的
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=ens33
UUID=d19fab40-0b57-42db-a884-1f9acbe5a5b5
DEVICE=ens33
ONBOOT=yes
IPADDR=192.168.2.132
NETMASK=255.255.255.0
GATEWAY=192.168.2.2
PEERDNS=yes
PEERROUTES=yes
IPV6_PEERDNS=yes
IPV6_PEERROUTES=yes
IPV6_PRIVACY=yes
BRIDGE=br0      # <---------- 看这里。挂在到br0虚拟交换机网桥上

# 激活br0，重启ens33
ifup br0
ifdown ens33
ifup ens33
# ----------------------------------

```

### 2.2 NAT（网络地址转换）

### 2.3 主机（Host-only Adapter）

### 2.4 内部网络（Internal）

## 3 Linux虚拟网络设备：tap/tun、veth-pair

## 4 Linux网卡虚拟化：macvlan

