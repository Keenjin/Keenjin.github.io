---
layout: post
title: 网络虚拟化
date: 2019-06-24
tags: 网络 虚拟化
---

# 虚拟网络原理

<!-- TOC -->

- [虚拟网络原理](#虚拟网络原理)
    - [1 概述](#1-概述)
    - [2 虚拟网络模型](#2-虚拟网络模型)
        - [2.1 桥接（Bridge Adapter）](#21-桥接bridge-adapter)
            - [2.1.1 桥接模型](#211-桥接模型)
            - [2.1.2 桥接在linux上的实现](#212-桥接在linux上的实现)
        - [2.2 NAT（网络地址转换）](#22-nat网络地址转换)
            - [2.2.1 NAT模型](#221-nat模型)
            - [2.2.2 NAT的配置实现](#222-nat的配置实现)
                - [2.2.2.1 使用libvirt命令配置永久生效](#2221-使用libvirt命令配置永久生效)
                - [2.2.2.2 使用iptables配置临时转发策略（重启后失效）](#2222-使用iptables配置临时转发策略重启后失效)
            - [2.2.3 利用tun/tap设备手动创建NAT网络（libvirt内部实现原理）](#223-利用tuntap设备手动创建nat网络libvirt内部实现原理)
        - [2.3 主机网络（Host-only Adapter）](#23-主机网络host-only-adapter)
        - [2.4 内部网络（Internal）](#24-内部网络internal)
    - [3 Linux虚拟网络设备：tap/tun、veth-pair](#3-linux虚拟网络设备taptunveth-pair)
        - [3.1 tap/tun](#31-taptun)
            - [3.1.1 传统网络与tap/tun网络对比](#311-传统网络与taptun网络对比)
            - [3.1.2 tap与tun的对比](#312-tap与tun的对比)
            - [3.1.3 linux下的应用](#313-linux下的应用)
        - [3.2 veth-pair](#32-veth-pair)
            - [3.2.1 网络模型](#321-网络模型)
            - [3.2.2 linux下的应用实例：同主机网段不同的两台虚拟机如何通信](#322-linux下的应用实例同主机网段不同的两台虚拟机如何通信)
        - [3.3 Linux网卡虚拟化：macvlan](#33-linux网卡虚拟化macvlan)

<!-- /TOC -->

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

#### 2.2.1 NAT模型

NAT（Network Address Translatatioin），网络地址转换。  

- IP地址不够用，是NAT技术出现的根本原因。
- 实现的基本原理，就是划分了一部分网段用于内网（公司局域网、机构局域网、个人局域网能使用的IPv4地址包括：10.0.0.0/8、172.16.0.0/12、192.168.0.0/16），内网内的ip不会对外暴露，因此不会可以重复利用。
- 每个内网连接到互联网，都通过一个公网IP连接出去。而NAT，就是实现这样1对多的网络架构，做网络数据包的地址转换。路由器中，就是利用这种原理来实现的。
- 从实现原理上看，NAT本质上，就是iptable映射表，把源ip端口，映射到目标ip端口。  

NAT的实现方式有三种：

- 静态NAT（Static NAT）：1对1的方式
- 动态NAT（Pooled NAT）
- 网络地址端口多路复用NAPT（Port-level NAT）：最常用的方式，真正实现1对多。包括SNAT和DNAT

![jpg](/images/post/virtual_network/NAPT.jpg)

源NAT（Source NAT，SNAT）：修改数据包的源地址。源NAT改变第一个数据包的来源地址，它永远会在数据包发送到网络之前完成，数据包伪装就是一具SNAT的例子  
目的NAT（Destination NAT，DNAT）：修改数据包的目的地址。Destination NAT刚好与SNAT相反，它是改变第一个数据懈的目的地地址，如平衡负载、端口转发和透明代理就是属于DNAT  

NAT的主要应用场景：

- 数据包伪装：可以将内网数据包中的地址信息更改成统一的对外地址信息，不让内网主机直接暴露在因特网上，保证内网主机的安全。同时，该功能也常用来实现共享上网
- 平衡负载：目的地址转换NAT可以重定向一些服务器的连接到其他随机选定的服务器。如果一个系统有一台通过路由器访问的关键服务器，一旦路由器检测到该服务器当机，它可以使用目的地址转换NAT透明的把连接转移到一个备份服务器上
- 端口转发：当内网主机对外提供服务时，由于使用的是内部私有IP地址，外网无法直接访问。因此，需要在网关上进行端口转发，将特定服务的数据包转发给内网主机。
- 透明代理：NAT可以把连接到因特网的HTTP连接重定向到一个指定的HTTP代理服务器以缓存数据和过滤请求。一些因特网服务提供商就使用这种技术来减少带宽的使用而不用让他们的客户配置他们的浏览器支持代理连接

#### 2.2.2 NAT的配置实现

##### 2.2.2.1 使用libvirt命令配置永久生效

（1）查看本机NAT映射表（需要安装libvirt，参考《CentOS7下KVM的安装使用》）

```bash
# 先查看当前有哪些网桥
ifconfig

ens33: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        ether 00:0c:29:49:2a:69  txqueuelen 1000  (Ethernet)
        RX packets 10801  bytes 1249426 (1.1 MiB)
        RX errors 0  dropped 91  overruns 0  frame 0
        TX packets 77326  bytes 5092019 (4.8 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 38751  bytes 3413716 (3.2 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 38751  bytes 3413716 (3.2 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

virbr0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500        # 安装virt的时候自动给装上的一个NAT网络
        inet 192.168.122.1  netmask 255.255.255.0  broadcast 192.168.122.255
        ether 52:54:00:7b:4d:e7  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

# ----------------------------------------------------------

# 查看当前的NAT表，只关注POSTROUTING即可，POSTROUTING表示在路由之后处理，也即在路由之后做了一个hook动作
iptables -t nat -L -nv

Chain POSTROUTING (policy ACCEPT 17 packets, 1020 bytes)        # 这里配置了NAT的映射表
 pkts bytes target     prot opt in     out     source               destination           
    0     0 RETURN     all  --  *      *       192.168.122.0/24     224.0.0.0/24        
    0     0 RETURN     all  --  *      *       192.168.122.0/24     255.255.255.255     
    0     0 MASQUERADE  tcp  --  *      *       192.168.122.0/24    !192.168.122.0/24     masq ports: 1024-65535
    0     0 MASQUERADE  udp  --  *      *       192.168.122.0/24    !192.168.122.0/24     masq ports: 1024-65535
    0     0 MASQUERADE  all  --  *      *       192.168.122.0/24    !192.168.122.0/24              

```

（2）参照默认的/usr/share/libvirt/networks/default.xml，来看当前的NAT，重新创建一个新的

```bash
# 查看libvirt默认创建的NAT
cd /usr/share/libvirt/networks/
vim default.xml

<network>                                                                                                                                                                        <name>default</name>
  <bridge name="virbr0"/>
  <forward/>
  <ip address="192.168.122.1" netmask="255.255.255.0">
    <dhcp>
      <range start="192.168.122.2" end="192.168.122.254"/>
    </dhcp>
  </ip>
</network>

# 复制出来，新建一个自己的
cp default.xml keennet.xml
vim keennet.xml

<network>
  <name>keennet</name>
  <bridge name="keenbr0"/>
  <forward/>
  <ip address="192.168.132.1" netmask="255.255.255.0">
    <dhcp>
      <range start="192.168.132.2" end="192.168.132.254"/>
    </dhcp>
  </ip>
</network>

# 定义成虚拟NAT网络
virsh net-define keennet.xml
# 标记自动启动
virsh net-autostart keennet
# 启动keennet虚拟网络
virsh net-start keennet
# brctl show或者ifconfig校验一下
ifconfig

ens33: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        ether 00:0c:29:49:2a:69  txqueuelen 1000  (Ethernet)
        RX packets 11125  bytes 1280700 (1.2 MiB)
        RX errors 0  dropped 92  overruns 0  frame 0
        TX packets 78340  bytes 5201581 (4.9 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

keenbr0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500       # 新增的keenbr0网桥
        inet 192.168.132.1  netmask 255.255.255.0  broadcast 192.168.132.255
        ether 52:54:00:e1:4e:00  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 39191  bytes 3452436 (3.2 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 39191  bytes 3452436 (3.2 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

virbr0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 192.168.122.1  netmask 255.255.255.0  broadcast 192.168.122.255
        ether 52:54:00:7b:4d:e7  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

# iptables查看最新的NAT表。可以看到，多了192.168.132.0/24的虚拟网络
iptables -t nat -L -nv

Chain POSTROUTING (policy ACCEPT 333 packets, 19980 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 RETURN     all  --  *      *       192.168.132.0/24     224.0.0.0/24        
    0     0 RETURN     all  --  *      *       192.168.132.0/24     255.255.255.255     
    0     0 MASQUERADE  tcp  --  *      *       192.168.132.0/24    !192.168.132.0/24     masq ports: 1024-65535
    0     0 MASQUERADE  udp  --  *      *       192.168.132.0/24    !192.168.132.0/24     masq ports: 1024-65535
    0     0 MASQUERADE  all  --  *      *       192.168.132.0/24    !192.168.132.0/24    
    0     0 RETURN     all  --  *      *       192.168.122.0/24     224.0.0.0/24        
    0     0 RETURN     all  --  *      *       192.168.122.0/24     255.255.255.255     
    0     0 MASQUERADE  tcp  --  *      *       192.168.122.0/24    !192.168.122.0/24     masq ports: 1024-65535
    0     0 MASQUERADE  udp  --  *      *       192.168.122.0/24    !192.168.122.0/24     masq ports: 1024-65535
    0     0 MASQUERADE  all  --  *      *       192.168.122.0/24    !192.168.122.0/24    

# 还需要确认主机是否允许ipv4转发
vim /etc/sysctl.conf

net.ipv4.ip_forward=1

# 至此，创建NAT虚拟网络设备结束，后续KVM创建虚拟机，传递keenbr0作为网桥即可。这里也进一步证明，网桥不止可以用于桥接模式，也可以用于NAT模式，它只是一个工具。模式的不同，是网络配置成桥接还是映射的不同
```

##### 2.2.2.2 使用iptables配置临时转发策略（重启后失效）

一般而言，SNAT是必要配置的，确保能正确上网，或者配置通用的转发规则，如[2.2.2.1 使用libvirt命令配置永久生效](#2221-使用libvirt命令配置永久生效)，让一个网段的ip都能正常转发，或者配置临时转发规则。  

（1）SNAT配置（使用POSTROUTING，在路由选择之后进行处理），用来共享上网

```bash
# 添加SNAT策略的防火墙规则（192.168.2.2是我机器vmware为内部虚拟机创建的网关）
# 这里的含义是：当192.168.3.0/24的请求直接向外输出到eth0，也就是从当前虚拟网关出去，就把ip地址修改为192.168.2.2网关的ip地址，这样对外就不会暴露内网地址
# 如果target网关是动态ip地址，使用命令替换：iptables -t nat -A POSTROUTING -s 192.168.3.0/24 -o eth0 -j MASQUERADE
iptables -t nat -A POSTROUTING -s 192.168.3.0/24 -o eth0 -j SNAT --to-source 192.168.2.2
# 校验一下是否成功
iptables -t nat -L -nv

Chain POSTROUTING (policy ACCEPT 11 packets, 660 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 RETURN     all  --  *      *       192.168.132.0/24     224.0.0.0/24        
    0     0 RETURN     all  --  *      *       192.168.132.0/24     255.255.255.255     
    0     0 MASQUERADE  tcp  --  *      *       192.168.132.0/24    !192.168.132.0/24     masq ports: 1024-65535
    0     0 MASQUERADE  udp  --  *      *       192.168.132.0/24    !192.168.132.0/24     masq ports: 1024-65535
    0     0 MASQUERADE  all  --  *      *       192.168.132.0/24    !192.168.132.0/24    
    0     0 RETURN     all  --  *      *       192.168.122.0/24     224.0.0.0/24        
    0     0 RETURN     all  --  *      *       192.168.122.0/24     255.255.255.255     
    0     0 MASQUERADE  tcp  --  *      *       192.168.122.0/24    !192.168.122.0/24     masq ports: 1024-65535
    0     0 MASQUERADE  udp  --  *      *       192.168.122.0/24    !192.168.122.0/24     masq ports: 1024-65535
    0     0 MASQUERADE  all  --  *      *       192.168.122.0/24    !192.168.122.0/24    
    0     0 MASQUERADE  all  --  *      !docker0  172.17.0.0/16        0.0.0.0/0           
    0     0 MASQUERADE  all  --  *      !br-459e55e82c16  192.168.0.0/24       0.0.0.0/0           
    0     0 SNAT       all  --  *      eth0    192.168.3.0/24       0.0.0.0/0            to:192.168.2.2
```

（2）DNAT配置（使用PREROUTING，在路由选择之前进行处理），用来做端口转发策略

```bash
# 添加DNAT策略的防火墙规则（10.16.104.67是vmware所在的主机的ip地址）
# 这里的含义是：当外网（对vmware而言，外网就是vmware所在的主机）10.16.104.67的数据包（tcp包端口号是80），通过网关eth0进入虚拟机，会被映射到192.168.3.10上去（没有填目标端口号，表示不更改端口映射关系）
# 如果想将ip和端口都修改掉，则输入这条命令即可：iptables -t nat -A PREROUTING -i eth0 -d 10.16.104.67 -p tcp --dport 80 -j DNAT --to-destination 192.168.3.10:10080
iptables -t nat -A PREROUTING -i eth0 -d 10.16.104.67 -p tcp --dport 80 -j DNAT --to-destination 192.168.3.10

# 校验
iptables -t nat -L -nv

Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    5   260 DOCKER     all  --  *      *       0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL
    0     0 DNAT       tcp  --  eth0   *       0.0.0.0/0            10.16.104.67         tcp dpt:80 to:192.168.3.10

# 应用案例（公司路由策略）：
# 从Internet中访问网关主机（120.196.122.145）的8888端口时，实际由运行在局域网主机（192.168.3.10）的8080端口应用程序提供服务。
# iptables -t nat -A PREROUTING -i eth0 -d 120.196.122.145 -p tcp --dport 8888 -j DNAT --to-destination 192.168.3.10:8080 
# 后续可以写脚本，一键配置n条策略，参考：https://blog.51cto.com/liweizhong/976183
```

#### 2.2.3 利用tun/tap设备手动创建NAT网络（libvirt内部实现原理）

tun/tap设备原理，参见[3.1 tap/tun](#31-taptun)

```bash
# 在主机上创建一个桥
brctl addbr br1
brctl show

bridge name	bridge id		STP enabled	interfaces
br1		8000.000000000000	no		# 新创建的，还没有任何网卡接口关联上来
keenbr0		8000.525400e14e00	yes		keenbr0-nic
virbr0		8000.5254007b4de7	yes		virbr0-nic

# 创建tap设备（以太网层设备），命名为tap1。一个tap设备，就是一个虚拟网卡接口
ip tuntap add dev tap1 mode tap
# 将tap设备tap1加到桥br1，实现与虚拟机互联
brctl addif br1 tap1
ip link set tap1 up

# 查看创建完成的interface
# 可以看到，tap1被加到了br1桥上
ip tuntap
ifconfig
brctl show

tap1: tap UNKNOWN_FLAGS:800

ens33: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        ether 00:0c:29:49:2a:69  txqueuelen 1000  (Ethernet)
        RX packets 12510  bytes 1429004 (1.3 MiB)
        RX errors 0  dropped 100  overruns 0  frame 0
        TX packets 86406  bytes 5757427 (5.4 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 43087  bytes 3795284 (3.6 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 43087  bytes 3795284 (3.6 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

tap1: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        ether 72:14:68:98:fd:d1  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

bridge name	bridge id		STP enabled	interfaces
br1		8000.72146898fdd1	no		tap1

# 给br1桥添加ip地址
ifconfig br1 192.168.142.1
ifconfig

br1: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 192.168.142.1  netmask 255.255.255.0  broadcast 192.168.142.255
        ether 72:14:68:98:fd:d1  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

ens33: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        ether 00:0c:29:49:2a:69  txqueuelen 1000  (Ethernet)
        RX packets 12638  bytes 1439858 (1.3 MiB)
        RX errors 0  dropped 100  overruns 0  frame 0
        TX packets 86538  bytes 5773535 (5.5 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 43117  bytes 3797924 (3.6 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 43117  bytes 3797924 (3.6 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

tap1: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        ether 72:14:68:98:fd:d1  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

# 后续启动虚拟机，直接设置-net使用tap1的tap设备即可
qemu -m 1024 -hda /home/royce/isorepo/ubuntu-vm1.img -netdev tap,ifname=tap1,id=hostnet0,script=no,downscript=no -device virtio-net-pci,netdev=hostnet0,id=net0, ac=00:16:3e:a5:fb:48 --cdrom /home/royce/isorepo/Fedora-19-x86_64-DVD.iso

# 配置主机的iptables规则，保证从虚拟机发出的包做源地址伪装，允许虚拟机上网
iptables -t nat -A POSTROUTING -o eth0 -s 192.168.142.0/24 -j MASQUERADE
```

### 2.3 主机网络（Host-only Adapter）

仅限主机内部访问的网络：主机和虚拟机可以互通，虚拟机之间可以互通（如果需要虚拟机能访问外网，需要单独设置一下，将物理网卡和虚拟网卡桥接上）  

![png](/images/post/virtual_network/hostonly.png)
![png](/images/post/virtual_network/hostnotonly.png)

### 2.4 内部网络（Internal）

最后一种网络模型是内部网络，这种模型是相对最简单的一种，虚拟机与外部环境完全断开，只允许虚拟机之间互相访问，这种模型一般不怎么用

## 3 Linux虚拟网络设备：tap/tun、veth-pair

### 3.1 tap/tun

linux内核2.4.x版本之后实现的虚拟网络设备。tap/tun对应的网络驱动：/dev/net/tun  

#### 3.1.1 传统网络与tap/tun网络对比

![jpg](/images/post/virtual_network/tun.jpg)
![jpg](/images/post/virtual_network/tap.jpg)

#### 3.1.2 tap与tun的对比

模拟网络层：

- tun设备是三层设备，只模拟到IP层。我们可以通过/dev/net/tun收发IP层数据包。可以使用ifconfig设置ip地址
- tap设备是二层设备，比tun更深入，可以通过/dev/net/tun文件收发mac层数据包。可以使用ifconfig设置ip地址，也可以设置mac地址

网桥功能：

- tun设备无法与物理网卡做bridge，但是可以通过ip_forward与物理网卡连通
- tap设备拥有mac层功能，可以与物理网卡做bridge，支持mac层广播

应用场景：

- tun设备常用于vpn、tunnel、ipsec之类的隧道（openvpn、vtun都是利用此机制实现的隧道封装）
- tap设备常用于虚拟交换机、网桥等

#### 3.1.3 linux下的应用

参考[2.2.3 利用tun/tap设备手动创建NAT网络（libvirt内部实现原理）](#223-利用tuntap设备手动创建nat网络libvirt内部实现原理)  

OpenVPN的实现，后续总结

### 3.2 veth-pair

linux内核2.6版本支持的特性。  

veth-pair主要应用场景：连接两个不同网段的容器，可以是docker，也可以是kvm。  

关于namespace和veth-pair的关联：

```txt
veth-pair技术出现的前提，是linux namespace技术的出现，而namespace技术，也正是docker技术出现的底层基础。  
参考：<https://www.cnblogs.com/bakari/p/8560437.html>  

安全技术上，我们有沙箱概念，就是将程序运行的环境隔离。一个虚拟机就是一个完全隔离的环境。但是虚拟机本身隔离的实现，是非常重的，需要实现指令集的支持，完整实现操作系统底层技术实现。而隔离技术，通过将一些全局资源很轻量的隔离起来，就实现前述所想实现的原理。docker就是利用此技术而产生的。  

所谓隔离，就是将一些全局资源（一个操作系统的基本运行要素，但非所有，而是几个必要的）划分namespace，每一个namespace都包含了一个小型操作系统的基本运行要素，包括：主机名、用户权限、文件系统、网络、进程号、进程间通信。（我们在docker中，可以运行一个操作系统，也证明了要素对于linux系统的完备性。而与此同时，我们可以看到，docker内部的资源，不太适合再次嵌套docker或者虚拟机）  

此课题比较高级，需要研究一下linux内核实现，后续再慢慢研究。  
```

#### 3.2.1 网络模型

![jpg](/images/post/virtual_network/veth-pair.jpg)

#### 3.2.2 linux下的应用实例：同主机网段不同的两台虚拟机如何通信

用两个namespace来模拟虚拟机。后续可以用真实虚拟机测试

![jpg](/images/post/virtual_network/veth-pair-exp.jpg)

```bash
# 创建两个namespace
ip netns add ns1
ip netns add ns2

# 创建两对veth-pair，一端分别挂在两个namespace
ip link add v1 type veth peer name v1_r
ip link add v2 type veth peer name v2_r
ip link set v1 netns ns1
ip link set v2 netns ns2

# 分别给两对veth-pair断点配上IP并启用
ip a a 10.10.10.1/24 dev v1_r
ip l s v1_r up
ip a a 10.10.20.1/24 dev v2_r
ip l s v2_r up

# ifconfig查看当前网络接口。v1_r和v2_r都分别生效了
ifconfig

ens33: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        ether 00:0c:29:49:2a:69  txqueuelen 1000  (Ethernet)
        RX packets 15117  bytes 1721868 (1.6 MiB)
        RX errors 0  dropped 117  overruns 0  frame 0
        TX packets 103248  bytes 6846513 (6.5 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 51392  bytes 4526124 (4.3 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 51392  bytes 4526124 (4.3 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

v1_r: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 10.10.10.1  netmask 255.255.255.0  broadcast 0.0.0.0
        ether 12:43:96:3f:68:94  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

v2_r: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 10.10.20.1  netmask 255.255.255.0  broadcast 0.0.0.0
        ether 4a:73:7c:00:5e:f7  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

# 分别给两个namespace配置ip
ip netns exec ns1 ip a a 10.10.10.2/24 dev v1
ip netns exec ns1 ip l s v1 up
ip netns exec ns2 ip a a 10.10.20.2/24 dev v2
ip netns exec ns2 ip l s v2 up

# 分别查看两个namespace的网络接口配置
ip netns exec ns1 ifconfig

v1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.10.10.2  netmask 255.255.255.0  broadcast 0.0.0.0
        inet6 fe80::b443:e5ff:fe4c:e832  prefixlen 64  scopeid 0x20<link>
        ether b6:43:e5:4c:e8:32  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 8  bytes 656 (656.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

ip netns exec ns2 ifconfig

v2: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.10.20.2  netmask 255.255.255.0  broadcast 0.0.0.0
        inet6 fe80::e02c:aaff:fe19:e4a6  prefixlen 64  scopeid 0x20<link>
        ether e2:2c:aa:19:e4:a6  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 8  bytes 656 (656.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

# 验证一下ns1 ping ns2，看是否ping得通（先需要确保开启了ipv4路由，cat /proc/sys/net/ipv4/ip_forward为1）
ip netns exec ns1 ping 10.10.20.2
# 无法ping通
connect: Network is unreachable

# 查看ns1的路由表
ip netns exec ns1 route -n
# 只有一条路由，没有通往10.10.20.0/24网段的路由
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
10.10.10.0      0.0.0.0         255.255.255.0   U     0      0        0 v1

# 给ns1配置一条路由，通往ns2
ip netns exec ns1 route add -net 10.10.20.0 netmask 255.255.255.0 gw 10.10.10.1
ip netns exec ns1 route -n
# 查看到，已经新增了
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
10.10.10.0      0.0.0.0         255.255.255.0   U     0      0        0 v1
10.10.20.0      10.10.10.1      255.255.255.0   UG    0      0        0 v1
# 尝试ping，发现网络可达，但是对方的网络却不能到达本机，ping要求双通
ip netns exec ns1 ping 10.10.20.2

# 给ns2配置一条路由，通往ns1
ip netns exec ns2 route add -net 10.10.10.0 netmask 255.255.255.0 gw 10.10.20.1

# 再次尝试，可以ping通
ip netns exec ns1 ping 10.10.20.2

PING 10.10.20.2 (10.10.20.2) 56(84) bytes of data.
64 bytes from 10.10.20.2: icmp_seq=1 ttl=63 time=0.043 ms
64 bytes from 10.10.20.2: icmp_seq=2 ttl=63 time=0.049 ms
64 bytes from 10.10.20.2: icmp_seq=3 ttl=63 time=0.042 ms
64 bytes from 10.10.20.2: icmp_seq=4 ttl=63 time=0.123 ms
```

### 3.3 Linux网卡虚拟化：macvlan

linux内核4.0支持的新特性。（这是一个相当新的内核版本才有的特性，我们现在流行的CentOS7.6的内核版本，也才3.10，CentOS6的内核版本，是2.6）  

暂时不研究
