---
layout: post
title: Squid代理服务器
date: 2019-07-10
tags: Linux
---

<!-- TOC -->

- [1. 代理服务器](#1-代理服务器)
- [2. Squid安装](#2-squid安装)
- [3. Squid配置详解](#3-squid配置详解)
    - [3.1. Squid多进程使用CPU多核](#31-squid多进程使用cpu多核)
    - [3.2. 代理服务器登录认证相关](#32-代理服务器登录认证相关)
    - [3.3. 访问控制](#33-访问控制)
    - [3.4. Squid的网络配置选项](#34-squid的网络配置选项)
    - [3.5. 缓存配置](#35-缓存配置)
    - [3.6. 附件A：squid_external_acl_helper.py](#36-附件asquid_external_acl_helperpy)
- [4. squid源码分析](#4-squid源码分析)
    - [4.1. 准备环境](#41-准备环境)
    - [4.2. 安装必备工具](#42-安装必备工具)
    - [4.3. squid源码编译](#43-squid源码编译)
    - [4.4. CLion远程调试squid](#44-clion远程调试squid)
    - [4.5. squid源码解析](#45-squid源码解析)

<!-- /TOC -->

# 1. 代理服务器

基本作用，是代理网络用户，去请求获取网络信息  

优势：

- 缓存数据，以便加速网络请求
- 隔离内网和外网，方便管理上网请求

分类：

- 正向代理：包括普通代理（最常用代理，通过/etc/profile指定代理服务器地址，实现网络请求的转发）、透明代理（一般用于网关主机，不需要用户设置代理服务器地址）
- 反向代理：通常是企业的服务部署在内网机器，反向代理的作用就是支持外网对企业服务的请求，转发到内网，并把回包从内网回传给互联网上的用户

# 2. Squid安装

```bash
# 安装
yum -y install squid
# 设置开机启动
systemctl enable squid.service
# 启动服务
systemctl start squid.service
# 查看版本号（后面配置文档，需要根据不同版本进行配置），这里是3.5
squid -v
```

# 3. Squid配置详解

参考<http://www.squid-cache.org/Versions/v3/3.5/cfgman/>  

## 3.1. Squid多进程使用CPU多核

Squid3.2之后，开始支持多核模式，有两种多核支持方式可选：workers或者cpu_affinity_map。csdn上说的，经过测试，cpu_affinity_map的方式比workers方式更佳

```bash
# 设置机器的CPU核心数（SMP多核模式），默认是不设置的话，是单核的
workers 4   # 设置cpu核心数为4个

# 把不同进程，绑定到不同核心上去，做1对1映射map
cpu_affinity_map process_numbers 1,2,3,4 cores 1,2,3,4  # 把四个squid进程，分别绑定到4个核心上，注意核心数0留给操作系统使用，从1开始
```

## 3.2. 代理服务器登录认证相关

代理如果需要身份认证才能访问，则需要添加这里的选项。这里参考<https://maoxian.de/2016/06/1415.html>

```bash
auth_param basic program /usr/lib/squid/ncsa_auth /etc/squid/squid_passwd   # 指定密码文件和用来验证密码的程序
auth_param basic children 5                             # 鉴权进程的数量
auth_param basic realm Squid proxy-caching web server   # 用户在web输入用户名和密码时，看到的提示信息
auth_param basic credentialsttl 2 hours                 # 用户名密码缓存时间
auth_param basic casesensitive off                      # 用户名是否需要匹配大小写
```

## 3.3. 访问控制

访问控制相关的，允许某些流量的通过，类似防火墙，可以选择过滤掉不想要的东西  
这里的squid_external_acl_helper.py参考[附件A：squid_external_acl_helper.py](附件A：squid_external_acl_helper.py)

```bash
# external_acl_type用来做自定义认证，可以通过自己指定一个后台程序，传递一些通用参数进去，进行认证放行
external_acl_type ip_user_check children-startup=5 ipv4 %LOGIN %SRC %MYADDR /usr/bin/python /etc/squid/squid_external_acl_helper.py     # 指定认证程序squid_external_acl_helper.py，传递登录用户名：%LOGIN%、客户端IP：%SRC%、服务端IP：%MYADDR，并使用一些特定参数，例如使用ipv4作为与helper程序通信的方式

# acl（access control list）用来定义一些访问控制列表，格式：acl 列表名称group 列表类型 列表参数
# acl一般与xxx_access一起使用，用来控制访问，xxx_access只需要deny或者allow对应的ACL即可
acl SSL_ports port 443  # 定义一个列表名称group：SSL_ports，表示的是端口号为443

# http_access用于控制允许或者拒绝http、https、ftp的相关访问请求。默认是都拒绝。
http_access deny CONNECT !SSL_ports     # 拒绝连接到非SSL的端口请求

# http_reply_access用于控制http请求的回包允许或者拒绝。默认都允许。
http_reply_access SSL_ports
```

## 3.4. Squid的网络配置选项

```bash
# http_port用来配置Squid监听http请求的服务程序端口号
http_port 3128

# https_port用来配置Squid监听https请求（TLS或者SSL）的服务程序端口号
https_port 3129

# ftp_port用来配置Squid监听ftp请求的服务程序端口号
ftp_port 3130
```

## 3.5. 缓存配置

```bash
# refresh_pattern配置缓存策略有效期，usage: refresh_pattern [-i] regex min percent max [options]
# min：表示ftp请求，缓存最小时间是1440分钟（24小时）。
# percent：假设这条请求对应的文件上一次修改时间是2019-07-11 02:00:00，进入cache的时间是2019-07-11 04:00:00，如果当前cache已缓存时间是5分钟，此时剩余失效时间是：60分钟*20% - 5分钟 = 7分钟。也就是说，剩余有效时间最多剩余7分钟。第二个参数是一个动态参数。当请求频率小于后台更新频率，比如后台10分钟更新一次，请求30分钟一次，如果我想保证每次都能请求到最新的，那么我每次请求，cache都必须失效，也就是第二次请求，后台最多刚更新10分钟，前一次cache时间是30分钟，也就是说，即使我的percent设置到100%，也一定会更新到最新的。如果后台50分钟更新一次，请求10分钟一次，则要想每次都能拉去最新，percent可以设置为20%，也就是说，当次请求距离上一次10分钟，已cache了10分钟，假设后台文件50分钟前，则最多失效时间为50*20%=10分钟。也就是说，percent=请求频率/后台更新频率，表示每次请求都能获取到最新数据
# max：表示ftp请求，缓存最大时间是10080（7天）
# 失效计算方式：如果当前cache已缓存时间比min小，无论如何，都继续缓存，因为还不到缓存最小值；如果当前cache已缓存时间比max大，则直接失效过期；如果当前cache已缓存时间在min和max之间，则根据percent来决策，如果剩余失效时间 <= 0，则过期
refresh_pattern ^ftp:		1440	20%	10080   
```

## 3.6. 附件A：squid_external_acl_helper.py

```python
# coding: utf-8
# squid_external_acl_helper.py
# Author: redice(qi@site-digger.com)
# Created at: 2016-10-14

# Python版本的Squid external_acl_type扩展ACL后台脚本
# 在squid.conf中的"htcp_access deny all"之前加入如下配置：
#external_acl_type ip_user_check children-startup=5 ipv4 %LOGIN %SRC %MYADDR /usr/bin/python /etc/squid/squid_external_acl_helper.py
#acl ipuseracl external ip_user_check
#http_access allow ipuseracl


import sys
import logging

# 记录日志
# sudo chmod 755 /var/log/squid/squid_auth_helper.log
# sudo chown proxy:proxy /var/log/squid/squid_auth_helper.log
logging.basicConfig(level=logging.DEBUG,
                    format='%(asctime)s %(levelname)s %(message)s',
                    filename='/var/log/squid/squid_auth_helper.log', filemode='a')


if __name__ == '__main__':
    while True:
        # 从stdin读取一行
        line = sys.stdin.readline()
        username, client_ip, local_ip = line.split()
        logging.info('New auth request: username = {}, client_ip = {}, local_ip = {}'.format(username, client_ip, local_ip))
        # 这里直接输出'OK'（通过认证，反之输出'ERR\n'）。你可以根据上述参数实现复杂的认证逻辑。
        sys.stdout.write('OK\n')
        sys.stdout.flush()

```

# 4. squid源码分析

## 4.1. 准备环境

- centos7.6
- CLion

## 4.2. 安装必备工具

```bash
yum update

# 准备工具
## make工具
yum install libtool autoconf automake libtool-ltdl-devel
## 解压工具
yum install zip unzip bzip2
## 基础gcc g++工具
yum install gcc gcc-g++

# 安装gcc新版本
## 下载gcc新版本
curl -OL "http://mirrors.nju.edu.cn/gnu/gcc/gcc-9.3.0/gcc-9.3.0.tar.gz"
tar -xzf gcc-9.3.0.tar.gz
cd gcc-9.3.0
## 预下载前置安装包（如果网络不好，可能出现文件校验失败，实际是文件没有完整下载，需要删除缓存文件，重新下载）；另外，如果这里下载一只超时，可能是下载路径不行，搜索download_prerequisites中的base_url，将链接替换为：http://mirror.linux-ia64.org/gnu/gcc/infrastructure/
contrib/download_prerequisites
## 新建临时目录，以便不污染源代码
mkdir build
cd build
## 收集系统信息，自动生成配置头。这里只编译64位库。
../configure --disable-multilib
## 开始编译，这个时间相当长，花费了一上午编译完成。（设置并行编译应该会更快）
make
## 开始安装
make install
## 安装完成，需要新建一个bash才能生效，查看gcc版本
gcc -v

## gcc编译完成，还需要链接新的动态链接库
cp /usr/local/lib64/libstdc++.so.6.0.28 /usr/lib64
rm -rf /usr/lib64/libstdc++.so.6
ln -s /usr/lib64/libstdc++.so.6.0.28 /usr/lib64/libstdc++.so.6
```

## 4.3. squid源码编译

```bash

# 源码同步
git clone -b v5 https://github.com/squid-cache/squid.git
cd squid

# 一堆的工具检测、配置检测、配置生成，依赖autoconf、automake等，最后会根据configure.ac和makefile.in生成configure和makefile文件
./bootstrap.sh

# 运行configure
mkdir build
cd build
../configure

# 编译
make
```

## 4.4. CLion远程调试squid

> Step1：准备环境

```bash
# 远程机器是mac（CLion在这里），被远程主机是centos
# 需要在被远程主机上，执行以下动作
yum install openssl openssl-devel

# 需要在被远程主机上，安装cmake。由于yum中自带等cmake版本过低，所以使用这里等源码安装，可以先试试yum install cmake
curl -OL https://github.com/Kitware/CMake/releases/download/v3.17.0/cmake-3.17.0.tar.gz
tar -xzf cmake*
cd cmake-3.17.0
mkdir build
cd build
../bootstrap
make && make install

# 需要在被远程主机上，安装gdb和gdbserver（源码编译时，这俩在一块儿）。注意的是，CLion只支持7.6到8.3版本，不要用太高的版本
curl -OL http://ftp.gnu.org/gnu/gdb/gdb-8.3.tar.gz
tar -xzf gdb-8.3*
cd gdb-8.3
mkdir build
cd build
../configure
make && make install

# 需要将远程主机的防火墙关闭
systemctl stop firewalld
systemctl disable firewalld
```

> Step2：配置CLion代码同步  

这里，首先需要利用CLion的deployment去同步代码  
![png](/images/post/squid/clion_deployment1.png)
![png](/images/post/squid/clion_deployment2.png)
![png](/images/post/squid/clion_deployment3.png)

> Step3：配置CLion进行代码调试

- 首先，需要配置调试目标
![png](/images/post/squid/clion_debug_remote.png)
- 然后，需要配置工具链
![png](/images/post/squid/clion_debug_toolchain.png)

> Step4：在远程主机上，写一个脚本，方便在启动时候，自动运行gdbserver调试

```bash
##startdebug.sh
#! /bin/bash

# 杀死历史所有的gdbserver
ps -ef | grep gdbserver | grep -v grep | awk '{print($2)}' | xargs kill -9
# 有可能一时半会儿杀不死，需要确保杀死，这里简单延时1s
sleep 1
# 后台运行gdbserver
nohup gdbserver :1234 build/src/squid -f build/src/squid.conf.default &
# 确保后台启动成功
sleep 1
```
![png](/images/post/squid/clion_debug_tools.png)

## 4.5. squid源码解析

