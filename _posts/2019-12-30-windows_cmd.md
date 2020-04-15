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

# 管理员模式
修改注册表：HKEY_LOCAL_MACHINE/SOFTWARE/MICROSOFT/WINDOWS NT/CURRENTVERSION/WINLOGON
（1）添加SpecialAccounts/UserList
（2）然后添加键值DWORD(32位)，名称Administrator，键值1
（3）开机进入带命令提示符安全模式，输入net user administrator  /active:yes;
```
