---
layout: post
title: 【操作指令】VMware + CentOS7 + XShell6环境配置
date: 2019-03-11 
tags: 操作指令 
---

<!-- TOC -->

- [1 虚拟机安装教程](#1-虚拟机安装教程)
- [2 网络配置](#2-网络配置)
- [3 SSH支持](#3-ssh支持)

<!-- /TOC -->

# 1 虚拟机安装教程

参考链接：[https://www.cnblogs.com/wcwen1990/p/7630545.html](https://www.cnblogs.com/wcwen1990/p/7630545.html)

# 2 网络配置

（1）配置网关  
cd /etc/sysconfig/network-scripts/  
ls  
![jpg](/images/post/vmware_centeros/1.jpg)
修改ip和网关：vi ifcfg-ens33  
![jpg](/images/post/vmware_centeros/2.jpg)
重启network服务：service network start  
确认ip地址配置ok：ip addr

（2）代理设置  
全局代理：vi /etc/profile  
![jpg](/images/post/vmware_centeros/3.jpg)
yum代理：vi /etc/yum.conf  
![jpg](/images/post/vmware_centeros/4.jpg)
wget代理：vi /etc/wgetrc  
![jpg](/images/post/vmware_centeros/5.jpg)

让profile生效：source /etc/profile

（3）网络命令支持（比如ifconfig、netstat）  
yum -y install net-tools

# 3 SSH支持

参考链接：[https://blog.csdn.net/trackle400/article/details/52755571](https://blog.csdn.net/trackle400/article/details/52755571)

（1）CenterOS虚拟机内安装openssh-server  
CenterOS虚拟机内，确保安装了openssh：yum list installed | grep openssh-server  
![jpg](/images/post/vmware_centeros/6.jpg)
如果CenterOS虚拟机内没有安装，使用命令：yum install openssh_server进行安装  

（2）CenterOS虚拟机内ssh配置修改  
cd /etc/ssh  
vi sshd_config  
![jpg](/images/post/vmware_centeros/7.jpg)
![jpg](/images/post/vmware_centeros/8.jpg)
启动sshd：service sshd start  
确认sshd已启动：ps -e | grep sshd，或者netstat -an | grep 22  
![jpg](/images/post/vmware_centeros/9.jpg)

（3）确保主机和虚拟机双向能ping通（NAT模式下，前面网关修改正确，应该是能ping通的）  
主机ip：cmd，然后ipconfig  
虚拟机ip：ifconfig  

（4）XShell配置连接虚拟机CenterOS  
![jpg](/images/post/vmware_centeros/10.jpg)
![jpg](/images/post/vmware_centeros/11.jpg)

（5）将sshd添加到自启动列表，避免每次都手动启动  
systemctrl enable sshd  
![jpg](/images/post/vmware_centeros/12.jpg)