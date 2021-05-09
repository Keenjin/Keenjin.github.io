---
layout: post
title: 【网络技术】网络是怎样连接的（四）- 接入网、网络运营商
date: 2017-03-13
tags: 网络技术
---

<!-- TOC -->

- [0 探索线路](#0-探索线路)
- [4 接入网、网络运营商](#4-接入网网络运营商)
    - [4.1 ADSL接入网的结构和工作方式](#41-adsl接入网的结构和工作方式)
        - [4.1.1 互联网基本结构和家庭、公司网络是相同的](#411-互联网基本结构和家庭公司网络是相同的)
        - [4.1.2 连接用户与互联网的接入网](#412-连接用户与互联网的接入网)
        - [4.1.3 ADSL Modem将包拆分成信元](#413-adsl-modem将包拆分成信元)
        - [4.1.4 ADSL将信元“调制”成信号、通过使用多个波来提高速率](#414-adsl将信元调制成信号通过使用多个波来提高速率)
        - [4.1.5 分离器的作用](#415-分离器的作用)
        - [4.1.6 从用户到电话局，通过DSLAM（电话局用的多路ADSL Modem）到达BAS（Broadband Access Server，宽带接入服务）](#416-从用户到电话局通过dslam电话局用的多路adsl-modem到达basbroadband-access-server宽带接入服务)
    - [4.2 光纤接入网（FTTH）](#42-光纤接入网ftth)
        - [4.2.1 FTTH接入网结构](#421-ftth接入网结构)
    - [4.3 接入网中使用的PPP和隧道](#43-接入网中使用的ppp和隧道)
        - [4.3.1 用户认证和配置下发](#431-用户认证和配置下发)
        - [4.3.2 在以太网上传输PPP消息](#432-在以太网上传输ppp消息)
        - [4.3.3 通过隧道将网络包发送给运营商](#433-通过隧道将网络包发送给运营商)
        - [4.3.4 接入网的整体工作过程](#434-接入网的整体工作过程)
        - [4.3.5 不适用PPP协议，而使用DHCP协议](#435-不适用ppp协议而使用dhcp协议)
    - [4.4 网络运营商的内部](#44-网络运营商的内部)
        - [4.4.1 POP（Point of Presense，接入点）和NOC（Network Operation Center，网络运营中心）](#441-poppoint-of-presense接入点和nocnetwork-operation-center网络运营中心)
    - [4.5 跨越运营商的网络包](#45-跨越运营商的网络包)
        - [4.5.1 运营商之间的连接及路由信息交换](#451-运营商之间的连接及路由信息交换)
        - [4.5.2 与公司网络中自动更新路由表机制的区别](#452-与公司网络中自动更新路由表机制的区别)
        - [4.5.3 IX（Internet eXchange，互联网交换中心）的必要性](#453-ixinternet-exchange互联网交换中心的必要性)

<!-- /TOC -->

# 0 探索线路

![jpg](/images/post/network_connect/1/01.jpg)
![jpg](/images/post/network_connect/1/02.jpg)

# 4 接入网、网络运营商

## 4.1 ADSL接入网的结构和工作方式

### 4.1.1 互联网基本结构和家庭、公司网络是相同的

区别：  
（1）路由距离不同  
（2）路由表记录数量庞大，路由表的维护方式不同
![jpg](/images/post/network_connect/4/64.jpg)

### 4.1.2 连接用户与互联网的接入网

互联网接入路由器是按照接入网规则来发送包的（接入网方式：ADSL、FTTH、CATV、电话线、ISDN等）

### 4.1.3 ADSL Modem将包拆分成信元

互联网接入路由器会再网络包前面加上MAC头部、PPPoE头部、PPP头部，然后发送给ADSL Modem（PPPoE方式下）
![jpg](/images/post/network_connect/4/65.jpg)
![jpg](/images/post/network_connect/4/66.jpg)

信元：用于一种叫做ATM（异步传输模式）的通信技术，开头5字节，后面48字节的一个小包

### 4.1.4 ADSL将信元“调制”成信号、通过使用多个波来提高速率

以太网传输，是采用方波模拟信号传输；互联网宽带上传输，采用的是正弦波合成的模拟信号表示0和1来传输。称为调制。  
信号调制方式：  
![jpg](/images/post/network_connect/4/67.jpg)

不同频率的信号，可以使用滤波器分离出来

### 4.1.5 分离器的作用

将ADSL信号和电话的语音信号进行分离。ADSL Modem内部已经具备将ADSL频率外的信号过滤掉的功能。
![jpg](/images/post/network_connect/4/68.jpg)

### 4.1.6 从用户到电话局，通过DSLAM（电话局用的多路ADSL Modem）到达BAS（Broadband Access Server，宽带接入服务）

从分离器出来，就是插电话线的接口，通过室内电话线，到达大楼的IDF和MDF（中控配线盘之类的东西，这里会加入一些防雷电之类的保护措施），连接外部电话线。然后，信号进入电线杆上架设的电话电缆（或埋在地底下的），连接到电话局的MDF。  

电话局采用多路ADSL Modem，最后到达BAS，会将受到的MAC头部和PPPoE头部丢弃，取出PPP头部以及后面的数据。接下来，BAS会再报的前面加上隧道专用头部，并发送到隧道出口

## 4.2 光纤接入网（FTTH）

### 4.2.1 FTTH接入网结构

单模光纤传输距离更远，所以FTTH一般用单模光纤（内部只传输一种角度的光信号，不会衰减）；多模光纤传输多种角度的光信号，要求更高，但是传输距离更短（易失真），因此多用于建筑内。  

ONU和OLT是用来识别信号是发送给自己的。
![jpg](/images/post/network_connect/4/69.jpg)

## 4.3 接入网中使用的PPP和隧道

### 4.3.1 用户认证和配置下发

ADSL和FTTH接入网中，都需要先输入用户名和密码，登录之后才能访问互联网，BAS使用PPPoE方式来实现这个功能。PPPoE由传统的PPP协议发展而来的。
![jpg](/images/post/network_connect/4/70.jpg)

### 4.3.2 在以太网上传输PPP消息

拨号上网使用PPP协议，PPP协议没有定义报头，所以需要容器（HDLC协议）。  
FTTH和ADSL不能使用HDLC，使用新的PPPoE容器。  

PPP认证流程：  
![jpg](/images/post/network_connect/4/71.jpg)
![jpg](/images/post/network_connect/4/72.jpg)

### 4.3.3 通过隧道将网络包发送给运营商

BAS作为认证和隧道的作用。隧道将接入网一直延伸到运营商路由器。  
两种隧道方式：  
![jpg](/images/post/network_connect/4/73.jpg)

### 4.3.4 接入网的整体工作过程

（1）从用户端的接入路由器开始  
（2）接入路由器中需要配置运营商分配的用户名和密码  
（3）接入路由器根据PPPoE的发现机制来寻找BAS（类似ARP一样的广播包），发包获取MAC地址  
（4）拿到MAC地址，准备开始通信  
（5）进入用户认证阶段（用户名和密码通过加密CHAP方式，或者不加密PAP方式，发送给BAS）  
（6）校验完密码，BAS向用户下发TCP/IP配置信息（包括分配给上网设备的IP地址-公有地址、DNS服务器的IP地址、默认网关的IP地址）  
（7）开始发送网络数据包：BAS收到用户路由器发送的网络包之后，去掉MAC头部PPPoE头部，然后用隧道机制将包发送给网络运营商的路由器
![jpg](/images/post/network_connect/4/74.jpg)

### 4.3.5 不适用PPP协议，而使用DHCP协议

![jpg](/images/post/network_connect/4/75.jpg)

## 4.4 网络运营商的内部

### 4.4.1 POP（Point of Presense，接入点）和NOC（Network Operation Center，网络运营中心）

网络包已经通过接入网，到达网络运营商的路由器，这里是互联网的入口，网络包从这里进入互联网内部。
![jpg](/images/post/network_connect/4/76.jpg)
![jpg](/images/post/network_connect/4/77.jpg)

互联网内部的汇聚路由器（比如RAS路由器）是超高性能的，数据吞吐量达到1Tbit/s，而我们常规的是100Mbit/s，相差1万倍。。。。

## 4.5 跨越运营商的网络包

### 4.5.1 运营商之间的连接及路由信息交换

如果网络包目的Wrb服务器和客户端是连接在同一个运营商中的，那么POP路由器的路由表中应该有相应的转发目标。路由器之间会自动进行路由表交换。  
如果不在同一个运营商中，借助于运营商之间的路由信息交换技术，来实现网络包转发。  
运营商之间的路由交换技术核心BGP（Border Gateway Protocol，边界网关协议）：相邻的路由器告知对方自身路由信息。
![jpg](/images/post/network_connect/4/78.jpg)
![jpg](/images/post/network_connect/4/79.jpg)

### 4.5.2 与公司网络中自动更新路由表机制的区别

公司网络：寻找与目的地之间的最短路由，并按照最短路由来转发包。路由是平等的。  
运营商之间：特定路由器间的一对一进行路由交换。

### 4.5.3 IX（Internet eXchange，互联网交换中心）的必要性

国外有非常多个运营商，这样，可以使用IX中心设备，来减少连接线路数量
![jpg](/images/post/network_connect/4/80.jpg)
