---
layout: post
title: 网络是怎样连接的
date: 2019-03-13
tags: 网络
---

# 0 探索线路

![png](/images/post/network_connect/1.png)
![png](/images/post/network_connect/2.png)

# 1 Web浏览器

## 1.1 生成HTTP请求

### 1.1.1 网址结构

![png](/images/post/network_connect/3.png)

### 1.1.2 网址解析

![png](/images/post/network_connect/4.png)

如果访问类似 <http://www.lab.glasscom.com/dir/> 这种以"/"结尾的，相当有访问index.html或default.html

### 1.1.3 HTTP的基本思路

![jpg](/images/post/network_connect/1.jpg)

请求消息表达的意思：对什么（URI，是指/dir1/file1.html或/dir1/program1.cgi文件名，也可以是整个url）进行怎么样的操作（方法）  

方法：  
![png](/images/post/network_connect/5.png)
PUT和DELETE方法，出于安全考虑，一般是不支持的。  

### 1.1.4 浏览器生成HTTP请求消息和收到的消息响应

请求消息及收到响应格式：
![png](/images/post/network_connect/6.png)
消息头：
![png](/images/post/network_connect/7.png)
![png](/images/post/network_connect/8.png)

消息响应的HTTP状态码概要：
![png](/images/post/network_connect/9.png)

HTTP消息示例：
![png](/images/post/network_connect/10.png)
![png](/images/post/network_connect/11.png)
![png](/images/post/network_connect/12.png)

## 1.2 向DNS服务器查询Web服务器的IP地址

### 1.2.1 IP地址基本知识

![png](/images/post/network_connect/13.png)
接入同一个集线器的机器，组成了子网；  
通过路由器，把多个子网连接起来。  

IP地址表示方法：
![png](/images/post/network_connect/14.png)
每一台设备的IP地址，都包括网络号+主机号。网络号是掩码部分 & IP地址；主机号是~掩码 & IP地址。  
主机号全0，表示的是整个子网，不能表示某个设备；  
主机号全1，表示向子网上所有设备发送包，即广播。
![jpg](/images/post/network_connect/2.jpg)

### 1.2.2 DNS服务器

通过域名来查询IP，提供此服务的是DNS服务器。计算机上，则安装有DNS客户端，通过调用Socket库，来进行解析，常见的api：IP地址 = gethostbyname("www.baidu.com")，就是用来查询IP地址的。  
![png](/images/post/network_connect/15.png)

这里的前提是，我们需要先知道DNS服务器的IP地址，这个是作为TCP/IP的一个设置项目，事先设置好的，不需要再去查询。我机器上，是我的小米路由器的IP地址，小米路由器作为我的DNS服务器。
![jpg](/images/post/network_connect/3.jpg)

## 1.3 全世界DNS服务器的大接力

### 1.3.1 DNS服务器的基本工作

![png](/images/post/network_connect/16.png)

### 1.3.2 DNS服务器寻找IP的过程

![png](/images/post/network_connect/17.png)
然而，实际的查询过程，并非每一个域都有一个DNS服务器，可能是共享服务器设备，部分查询可以简化，少查询一层两层等。另外，也会缓存未查询到的结果，下一次请求，直接返回，或者查询到了，直接存储到本地，当数据到达有效期，就删除。我上面的小米路由器作为DNS服务器，应该也是有这些功能，目的就是响应时间优化。

## 1.4 委托协议栈发送消息

### 1.4.1 数据收发操作概览

消息要能传达出去，首先要建立一个通信管道，建立的基础，是两个管道出口的套接字。
![png](/images/post/network_connect/18.png)

### 1.4.2 套接字使用过程

创建套接字 ——> 连接服务器 ——> 发送数据 ——> 接收数据 ——> 断开连接
![png](/images/post/network_connect/19.png)

描述符socket句柄：应用程序用来识别套接字的机制  
IP地址和端口号：客户端和服务器之间，用来识别对方套接字的机制  

这里套接字是在协议栈的更上层，是协议栈的高度封装。

# 2 协议栈、网卡

围绕着“创建套接字 ——> 连接服务器 ——> 发送数据 ——> 接收数据 ——> 断开连接”来讲解

## 2.1 创建套接字

### 2.1.1 协议栈内部结构

![png](/images/post/network_connect/20.png)

浏览器、邮件等一般应用程序收发数据用TCP；  
DNS查询等收发较短的控制数据用UDP。  

IP层中包含ICMP协议（用于告知网络包传送过程中产生的错误以及各种控制消息）和ARP协议（用于根据IP地质查询相应的以太网MAC地址）  

IP层以下是网卡驱动程序负责控制网卡硬件，更下面是网卡，对网线中的电或光信号执行发送和接收操作

### 2.1.2 套接字对于协议栈的作用

套接字实体，包含IP地址和端口号（源和目标），以及控制信息（包括是否收到响应、多长时间了、是否重试了等）。协议栈根据套接字存储的信息，判断下一步的行动。  

windows中netstat命令，展示了所有套接字：
![png](/images/post/network_connect/21.png)

### 2.1.3 调用socket时的操作

![png](/images/post/network_connect/22.png)

## 2.2 连接服务器

### 2.2.1 连接的含义

连接，就是客户端和服务器相互交换套接字控制信息（比如IP地址、端口号等）

### 2.2.2 控制信息头部

![png](/images/post/network_connect/24.png)
TCP头部格式：
![png](/images/post/network_connect/23.png)

### 2.2.3 连接操作的实际过程

（1）首先，客户机跟服务器需要交换控制信息（包括双方的IP地址端口号等），结束后，客户机的头部SYN比特置1。等待连接状态切换为正在连接状态  
（2）服务器的SYN比特置1，ACK比特置1，回包给客户机  
（3）当客户机收到服务器的套接字信息后，检查SYN是否为1，如果为1表示连接成功，正在连接状态切换为连接成功状态  
（4）客户机将ACK置1，返回给服务器，告知对方已收到回包

## 2.3 收发数据



# 3 集线器、交换机、路由器

# 4 接入网、网络运营商

# 5 防火墙、缓存服务器

# 6 Web服务器

# 