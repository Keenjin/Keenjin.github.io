---
layout: post
title: 【网络技术】网络是怎样连接的（五）- 防火墙、缓存服务器
date: 2017-03-13
tags: 网络技术
---

<!-- TOC -->

- [0 探索线路](#0-探索线路)
- [5 防火墙、缓存服务器](#5-防火墙缓存服务器)
    - [5.1 Web服务器的部署地点](#51-web服务器的部署地点)
        - [5.1.1 在公司里部署Web服务器](#511-在公司里部署web服务器)
    - [5.2 防火墙的结构和原理](#52-防火墙的结构和原理)
        - [5.2.1 主流的包过滤方式](#521-主流的包过滤方式)
    - [5.3 通过将请求平均分配给多态服务器来平衡负载](#53-通过将请求平均分配给多态服务器来平衡负载)
        - [5.3.1 性能不足时需要负载均衡](#531-性能不足时需要负载均衡)
        - [5.3.2 负载均衡器分配访问](#532-负载均衡器分配访问)
    - [5.4 使用缓存服务器分担负载](#54-使用缓存服务器分担负载)
        - [5.4.1 如何使用缓存服务器](#541-如何使用缓存服务器)
        - [5.4.2 代理](#542-代理)
    - [5.5 内容分发服务](#55-内容分发服务)
        - [5.5.1 利用内容分发服务分担负载](#551-利用内容分发服务分担负载)
        - [5.5.2 通过重定向服务器分配访问目标](#552-通过重定向服务器分配访问目标)
        - [5.5.3 缓存的更新方法会影响性能](#553-缓存的更新方法会影响性能)
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

# 5 防火墙、缓存服务器

## 5.1 Web服务器的部署地点

### 5.1.1 在公司里部署Web服务器

![jpg](/images/post/network_connect/5/81.jpg)

也可以直接部署到运营商的数据中心，或者租用运营商的服务器

## 5.2 防火墙的结构和原理

### 5.2.1 主流的包过滤方式

防火墙可分为：包过滤、应用层网关、电路层网关等方式。主流的是包过滤。  

常见包过滤规则：  
![jpg](/images/post/network_connect/5/82.jpg)

典型示例 - 只允许外面访问Web服务器，不允许Web服务器方位互联网（防止一些木马程序已经入侵了服务器，进而扩散）：
![jpg](/images/post/network_connect/5/83.jpg)

（1）也可以通过端口号限定应用程序：例如Web服务器的端口号是80，那么可以只允许80通过。当接收方IP是Web服务器并且端口号为80时，允许通过，其他包不允许通过  

（2）通过控制位判断连接方向（TCP）：只阻止某个控制包，就可以终止整个连接  

（3）是否允许地址转换：否的话，就能让外部无法访问公司内网  

包过滤方式的防火墙可根据：接收方IP地址、发送方IP地址、接收方端口号、发送方端口号、控制位等信息来判断某个包是否允许通过  

若Web服务器有bug，那是内容层级的，防火墙通常也无能为力。也可以加上一些能识别内容的防火墙软件。

## 5.3 通过将请求平均分配给多态服务器来平衡负载

### 5.3.1 性能不足时需要负载均衡

流量包太多了，服务器处理不过来，这个时候需要分布式架构。

### 5.3.2 负载均衡器分配访问

![jpg](/images/post/network_connect/5/84.jpg)
定期采集Web服务器的CPU、内存使用率，并根据这些数据判断服务器负载情况，也可以发送测试包，根据响应时间判断。  
需要将相关性的请求，分发到同一台Web服务器，就需要扩展相关协议，将类似HTTP包的各个独立包的相关性，在协议中表达出来。  

## 5.4 使用缓存服务器分担负载

### 5.4.1 如何使用缓存服务器

按功能来分担负载的方法。  
介于Web服务器和客户端之间。可以缓存相关数据，直接回复给客户端（缓存数据同步问题）；处理掉无需经过Web服务器的内容；让缓存服务器处理掉一部分内容  
![jpg](/images/post/network_connect/5/85.jpg)

### 5.4.2 代理

![jpg](/images/post/network_connect/5/86.jpg)
![jpg](/images/post/network_connect/5/87.jpg)

正向代理：在客户端的一个缓存服务器，一般是在浏览器的设置窗口中的“代理服务器”协商正向代理IP地址。  
反向代理：靠Web服务器端的缓存服务器，通过DNS服务器解析引导的方法  
透明代理：将请求消息放到网络包路径中，当消息经过时进行拦截

## 5.5 内容分发服务

### 5.5.1 利用内容分发服务分担负载

CDSP（Content Delivery Service，内容分发服务运营商），会与运营商签约，部署很多CDSP，可以借用他们减少成本。

### 5.5.2 通过重定向服务器分配访问目标

Location字段，将客户端访问引导到另一台Web服务器的操作，称为重定向

### 5.5.3 缓存的更新方法会影响性能

Web服务器数据有更新时，通知缓存服务器。

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