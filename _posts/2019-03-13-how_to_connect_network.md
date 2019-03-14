---
layout: post
title: 网络是怎样连接的
date: 2019-03-13
tags: 网络
---

# 0 探索线路

![jpg](/images/post/network_connect/1/01.jpg)
![jpg](/images/post/network_connect/1/02.jpg)

# 1 Web浏览器

## 1.1 生成HTTP请求

### 1.1.1 网址结构

![jpg](/images/post/network_connect/1/03.jpg)

### 1.1.2 网址解析

![jpg](/images/post/network_connect/1/4.jpg)

如果访问类似 <http://www.lab.glasscom.com/dir/> 这种以"/"结尾的，相当有访问index.html或default.html

### 1.1.3 HTTP的基本思路

![jpg](/images/post/network_connect/1/1.jpg)

请求消息表达的意思：对什么（URI，是指/dir1/file1.html或/dir1/program1.cgi文件名，也可以是整个url）进行怎么样的操作（方法）  

方法：  
![jpg](/images/post/network_connect/1/5.jpg)
PUT和DELETE方法，出于安全考虑，一般是不支持的。  

### 1.1.4 浏览器生成HTTP请求消息和收到的消息响应

请求消息及收到响应格式：
![jpg](/images/post/network_connect/1/6.jpg)
消息头：
![jpg](/images/post/network_connect/1/7.jpg)
![jpg](/images/post/network_connect/1/8.jpg)

消息响应的HTTP状态码概要：
![jpg](/images/post/network_connect/1/9.jpg)

HTTP消息示例：
![jpg](/images/post/network_connect/1/10.jpg)
![jpg](/images/post/network_connect/1/11.jpg)
![jpg](/images/post/network_connect/1/12.jpg)

## 1.2 向DNS服务器查询Web服务器的IP地址

### 1.2.1 IP地址基本知识

![jpg](/images/post/network_connect/1/13.jpg)
接入同一个集线器的机器，组成了子网；  
通过路由器，把多个子网连接起来。  

IP地址表示方法：
![jpg](/images/post/network_connect/1/14.jpg)
每一台设备的IP地址，都包括网络号+主机号。网络号是掩码部分 & IP地址；主机号是~掩码 & IP地址。  
主机号全0，表示的是整个子网，不能表示某个设备；  
主机号全1，表示向子网上所有设备发送包，即广播。
![jpg](/images/post/network_connect/2.jpg)

### 1.2.2 DNS服务器

通过域名来查询IP，提供此服务的是DNS服务器。计算机上，则安装有DNS客户端，通过调用Socket库，来进行解析，常见的api：IP地址 = gethostbyname("www.baidu.com")，就是用来查询IP地址的。  
![jpg](/images/post/network_connect/1/15.jpg)

这里的前提是，我们需要先知道DNS服务器的IP地址，这个是作为TCP/IP的一个设置项目，事先设置好的，不需要再去查询。我机器上，是我的小米路由器的IP地址，小米路由器作为我的DNS服务器。
![jpg](/images/post/network_connect/1/3.jpg)

## 1.3 全世界DNS服务器的大接力

### 1.3.1 DNS服务器的基本工作

![jpg](/images/post/network_connect/1/16.jpg)

### 1.3.2 DNS服务器寻找IP的过程

![jpg](/images/post/network_connect/1/17.jpg)
然而，实际的查询过程，并非每一个域都有一个DNS服务器，可能是共享服务器设备，部分查询可以简化，少查询一层两层等。另外，也会缓存未查询到的结果，下一次请求，直接返回，或者查询到了，直接存储到本地，当数据到达有效期，就删除。我上面的小米路由器作为DNS服务器，应该也是有这些功能，目的就是响应时间优化。

## 1.4 委托协议栈发送消息

### 1.4.1 数据收发操作概览

消息要能传达出去，首先要建立一个通信管道，建立的基础，是两个管道出口的套接字。
![jpg](/images/post/network_connect/1/18.jpg)

### 1.4.2 套接字使用过程

创建套接字 ——> 连接服务器 ——> 发送数据 ——> 接收数据 ——> 断开连接
![jpg](/images/post/network_connect/1/19.jpg)

描述符socket句柄：应用程序用来识别套接字的机制  
IP地址和端口号：客户端和服务器之间，用来识别对方套接字的机制  

这里套接字是在协议栈的更上层，是协议栈的高度封装。

# 2 协议栈、网卡

围绕着“创建套接字 ——> 连接服务器 ——> 发送数据 ——> 接收数据 ——> 断开连接”来讲解

## 2.1 创建套接字

### 2.1.1 协议栈内部结构

![jpg](/images/post/network_connect/2/20.jpg)

浏览器、邮件等一般应用程序收发数据用TCP；  
DNS查询等收发较短的控制数据用UDP。  

IP层中包含ICMP协议（用于告知网络包传送过程中产生的错误以及各种控制消息）和ARP协议（用于根据IP地质查询相应的以太网MAC地址）  

IP层以下是网卡驱动程序负责控制网卡硬件，更下面是网卡，对网线中的电或光信号执行发送和接收操作

### 2.1.2 套接字对于协议栈的作用

套接字实体，包含IP地址和端口号（源和目标），以及控制信息（包括是否收到响应、多长时间了、是否重试了等）。协议栈根据套接字存储的信息，判断下一步的行动。  

windows中netstat命令，展示了所有套接字：
![jpg](/images/post/network_connect/2/21.jpg)

### 2.1.3 调用socket时的操作

![jpg](/images/post/network_connect/2/22.jpg)

## 2.2 连接服务器

### 2.2.1 连接的含义

连接，就是客户端和服务器相互交换套接字控制信息（比如IP地址、端口号等）

### 2.2.2 控制信息头部

![jpg](/images/post/network_connect/2/24.jpg)
TCP头部格式：
![jpg](/images/post/network_connect/2/23.jpg)

### 2.2.3 连接操作的实际过程

（1）首先，客户机跟服务器需要交换控制信息（包括双方的IP地址端口号等），结束后，客户机的头部SYN比特置1。等待连接状态切换为正在连接状态  
（2）服务器的SYN比特置1，ACK比特置1，回包给客户机  
（3）当客户机收到服务器的套接字信息后，检查SYN是否为1，如果为1表示连接成功，正在连接状态切换为连接成功状态  
（4）客户机将ACK置1，返回给服务器，告知对方已收到回包

## 2.3 收发数据

### 2.3.1 HTTP请求消息交给协议栈

上层应用通过write传递一段二进制序列给协议栈，协议栈不会立刻发送出去，而是将数据存放在内部的发送缓冲区中，并等待应用程序的下一段数据。当数据累积到一定量，才会进行发送。  

累积数据的长度决定因素：  
（1）MTU（最大传输单元）。在以太网中一般是1500 Bytes，包含包头长度。MSS（最大分段大小）是MTU-len（包头）  
（2）时间。协议栈内部有一个计时器，达到这个时间后，即使包大小不到MTU，依旧发送  
（3）应用程序可以指定选项：不等待填满缓冲区直接发送（浏览器一般是如此）  
![jpg](/images/post/network_connect/2/25.jpg)

### 2.3.2 对较大的数据进行拆分

包太大，TCP会将包拆分成小包发送
![jpg](/images/post/network_connect/2/26.jpg)

### 2.3.3 使用ACK号确认网络包已收到

发送方说：“现在发送的是从第1字节开始的部分，一共有1460字节”  
接收方回复说：“到第1461字节之前的数据，我已经都收到了”。ACK号是1461
![jpg](/images/post/network_connect/27.jpg)

数据双向传输：  
![jpg](/images/post/network_connect/2/28.jpg)

连接操作 + 收发操作：  
![jpg](/images/post/network_connect/2/29.jpg)

### 2.3.4 动态调整ACK号等待时间（超时时间）

TCP采用动态调整等待ACK号返回的方法，在数据发送过程中，持续测量ACK号返回时间。返回变慢，则增长等待时间；否则，缩短等待时间

### 2.3.5 使用窗口有效管理ACK号

一个简单的异步调用过程，在等待ACK号这段时间内，继续发送后续的一系列包。  
一个问题：如果一直收不到ACK号，却依旧一直发送后续的包，会导致发送包频率超过接收方能处理能力的情况。
![jpg](/images/post/network_connect/2/30.jpg)

因此，需要采用滑动窗口的方式，来动态平衡发送和接收频率。具体做法：接收方需要告诉发送方自己最多能接收多少数据（比如此刻队列已快满了，最多能接收的数据就比较少），发送方根据这个大小来发送数据。
![jpg](/images/post/network_connect/2/31.jpg)

更新窗口大小的时机：接收方从缓冲区中取出数据给应用程序之后，需要告知发送方。通常会将ACK包和更新窗口大小的包合并在一起发送，会有一个等待时间，不是立刻发送，这样也可以省略中间过程。

### 2.3.7 接收HTTP响应请求

协议栈会检查收到的数据块和TCP头部的内容，判断是否有数据丢失，如果没有问题，则返回ACK号。然后将数据copy到应用程序内存

## 2.4 从服务器断开并删除套接字

### 2.4.1 数据发送完毕后断开连接

![jpg](/images/post/network_connect/2/32.jpg)

### 2.4.2 删除套接字

等待重传完毕后，才删除套接字，一般等待几分钟

### 2.4.3 TCP整体流程

![jpg](/images/post/network_connect/2/33.jpg)

## 2.5 IP与以太网的包收发操作

## 2.6 UDP协议的收发操作

# 3 集线器、交换机、路由器

# 4 接入网、网络运营商

# 5 防火墙、缓存服务器

# 6 Web服务器