---
layout: post
title: 【网络技术】网络是怎样连接的（六）- Web服务器、网络包旅程
date: 2017-03-13
tags: 网络技术
---

<!-- TOC -->

- [0 探索线路](#0-探索线路)
- [6 Web服务器](#6-web服务器)
    - [6.1 服务器概览](#61-服务器概览)
        - [6.1.1 服务器程序的结构](#611-服务器程序的结构)
        - [6.1.2 服务器端的套接字和端口号](#612-服务器端的套接字和端口号)
    - [6.2 浏览器接收响应消息并显示内容](#62-浏览器接收响应消息并显示内容)
        - [6.2.1 通过响应的数据类型判断其中的内容](#621-通过响应的数据类型判断其中的内容)
- [7 网络包旅程](#7-网络包旅程)

<!-- /TOC -->

# 0 探索线路

![jpg](/images/post/network_connect/1/01.jpg)
![jpg](/images/post/network_connect/1/02.jpg)

# 6 Web服务器

## 6.1 服务器概览

### 6.1.1 服务器程序的结构

![jpg](/images/post/network_connect/6/88.jpg)

### 6.1.2 服务器端的套接字和端口号

![jpg](/images/post/network_connect/6/89.jpg)

## 6.2 浏览器接收响应消息并显示内容

### 6.2.1 通过响应的数据类型判断其中的内容

中间的过程由于跟客户端发送端基本对称，全部省略，直接到最后这一步。  
HTTP的Content-Type定义的数据类型：
![jpg](/images/post/network_connect/6/90.jpg)

# 7 网络包旅程

![jpg](/images/post/network_connect/6/91.jpg)
![jpg](/images/post/network_connect/6/92.jpg)