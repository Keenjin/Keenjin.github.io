---
layout: post
title: Squid代理服务器搭建
date: 2019-07-10
tags: Linux
---

# Squid代理服务器搭建

## 代理服务器

基本作用，是代理网络用户，去请求获取网络信息  

优势：

- 缓存数据，以便加速网络请求
- 隔离内网和外网，方便管理上网请求

分类：

- 正向代理：包括普通代理（最常用代理，通过/etc/profile指定代理服务器地址，实现网络请求的转发）、透明代理（一般用于网关主机，不需要用户设置代理服务器地址）
- 反向代理：通常是企业的服务部署在内网机器，反向代理的作用就是支持外网对企业服务的请求，转发到内网，并把回包从内网回传给互联网上的用户

## Squid安装

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

## Squid配置详解

参考<http://www.squid-cache.org/Versions/v3/3.5/cfgman/>  

### Squid多进程使用CPU多核

Squid3.2之后，开始支持多核模式，有两种多核支持方式可选：workers或者cpu_affinity_map。csdn上说的，经过测试，cpu_affinity_map的方式比workers方式更佳

```bash
# 设置机器的CPU核心数（SMP多核模式），默认是不设置的话，是单核的
workers 4   # 设置cpu核心数为4个

# 把不同进程，绑定到不同核心上去，做1对1映射map
cpu_affinity_map process_numbers 1,2,3,4 cores 1,2,3,4  # 把四个squid进程，分别绑定到4个核心上，注意核心数0留给操作系统使用，从1开始
```

### 代理服务器登录认证相关

代理如果需要身份认证才能访问，则需要添加这里的选项。这里参考<https://maoxian.de/2016/06/1415.html>

```bash
auth_param basic program /usr/lib/squid/ncsa_auth /etc/squid/squid_passwd   # 指定密码文件和用来验证密码的程序
auth_param basic children 5                             # 鉴权进程的数量
auth_param basic realm Squid proxy-caching web server   # 用户在web输入用户名和密码时，看到的提示信息
auth_param basic credentialsttl 2 hours                 # 用户名密码缓存时间
auth_param basic casesensitive off                      # 用户名是否需要匹配大小写
```

### 访问控制

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

### Squid的网络配置选项

```bash
# http_port用来配置Squid监听http请求的服务程序端口号
http_port 3128

# https_port用来配置Squid监听https请求（TLS或者SSL）的服务程序端口号
https_port 3129

# ftp_port用来配置Squid监听ftp请求的服务程序端口号
ftp_port 3130
```

### 缓存配置

```bash
# refresh_pattern配置缓存策略有效期，usage: refresh_pattern [-i] regex min percent max [options]
# min：表示ftp请求，缓存最小时间是1440分钟（24小时）。
# percent：假设这条请求对应的文件上一次修改时间是2019-07-11 02:00:00，进入cache的时间是2019-07-11 04:00:00，如果当前cache已缓存时间是5分钟，此时剩余失效时间是：60分钟*20% - 5分钟 = 7分钟。也就是说，剩余有效时间最多剩余7分钟。第二个参数是一个动态参数。当请求频率小于后台更新频率，比如后台10分钟更新一次，请求30分钟一次，如果我想保证每次都能请求到最新的，那么我每次请求，cache都必须失效，也就是第二次请求，后台最多刚更新10分钟，前一次cache时间是30分钟，也就是说，即使我的percent设置到100%，也一定会更新到最新的。如果后台50分钟更新一次，请求10分钟一次，则要想每次都能拉去最新，percent可以设置为20%，也就是说，当次请求距离上一次10分钟，已cache了10分钟，假设后台文件50分钟前，则最多失效时间为50*20%=10分钟。也就是说，percent=请求频率/后台更新频率，表示每次请求都能获取到最新数据
# max：表示ftp请求，缓存最大时间是10080（7天）
# 失效计算方式：如果当前cache已缓存时间比min小，无论如何，都继续缓存，因为还不到缓存最小值；如果当前cache已缓存时间比max大，则直接失效过期；如果当前cache已缓存时间在min和max之间，则根据percent来决策，如果剩余失效时间 <= 0，则过期
refresh_pattern ^ftp:		1440	20%	10080   
```

### 附件A：squid_external_acl_helper.py

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