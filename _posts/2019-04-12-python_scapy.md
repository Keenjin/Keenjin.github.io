---
layout: post
title: 【源码片段】Python Scapy模块使用
date: 2019-04-12
tags: Python 源码片段
---

# 可以用来干嘛？

构造包，解析包，发送数据包，捕获数据包，模拟请求和回复；  
扫描网络、路由trace、探针、单元测试、网络攻击、网络探测；  
可以替代hping、arpspoof、arpsk、arping、pOf，甚至替代Nmap、tcpdump、tshark模块一部分；  

发送无效数据帧、注入802.11数据帧、组合各种技术等  
总而言之，可以任意发送原始数据包，实现任何数据包发送与解析，而且代码量极少  

