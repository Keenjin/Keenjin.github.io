---
layout: post
title: 网络是怎样连接的（一）
date: 2019-03-13
tags: 网络
---

<!-- TOC -->

- [0 探索线路](#0-探索线路)
- [1 Web浏览器](#1-web浏览器)
    - [1.1 生成HTTP请求](#11-生成http请求)
        - [1.1.1 网址结构](#111-网址结构)
        - [1.1.2 网址解析](#112-网址解析)
        - [1.1.3 HTTP的基本思路](#113-http的基本思路)
        - [1.1.4 浏览器生成HTTP请求消息和收到的消息响应](#114-浏览器生成http请求消息和收到的消息响应)
    - [1.2 向DNS服务器查询Web服务器的IP地址](#12-向dns服务器查询web服务器的ip地址)
        - [1.2.1 IP地址基本知识](#121-ip地址基本知识)
        - [1.2.2 DNS服务器](#122-dns服务器)
    - [1.3 全世界DNS服务器的大接力](#13-全世界dns服务器的大接力)
        - [1.3.1 DNS服务器的基本工作](#131-dns服务器的基本工作)
        - [1.3.2 DNS服务器寻找IP的过程](#132-dns服务器寻找ip的过程)
    - [1.4 委托协议栈发送消息](#14-委托协议栈发送消息)
        - [1.4.1 数据收发操作概览](#141-数据收发操作概览)
        - [1.4.2 套接字使用过程](#142-套接字使用过程)

<!-- /TOC -->

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
![jpg](/images/post/network_connect/1/2.jpg)

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

