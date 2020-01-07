---
layout: post
title: Windows命令行
date: 2019-12-30
tags: windows
---

# 系统相关

```bash
# 关闭Win10 休眠管理
powercfg -h off

# 单网卡同时支持dhcp和静态ip
# Windows 10要求：1703以上版本
netsh int ipv4 set interface "以太网" dhcpstaticipcoexistence=enabled
netsh int ipv4 add address "以太网" 192.168.100.101 255.255.255.0
```
