---
layout: post
title: 【调试技术】Linux桌面客户端C++开发
date: 2023-03-11
tags: Linux
---

# 1. 兼容性

## 1.1. glibc版本号

|系统家族	|系统大版本	|系统子版本	|linux内核版本	|内嵌gcc版本	|glibc最高支持版本|
|:---------:|:--------:|:--------:|:------------:|:-------------:|:---------------:|
|Ubuntu	|14.04	|14.04	|4.14	|gcc 4.8|	2.19|
|Ubuntu	|16.04 LTS	|16.04 LTS	|4.14	|gcc 5.4.0|	2.23|
|Ubuntu|18.04 LTS	|18.04 LTS|	4.14	|gcc 7.5.0|	2.27|
|Ubuntu|20.04 LTS	|20.04 LTS	|5.4|	gcc 9.3.0|	2.31|
|国产系统	|KylinV10	|KylinV10|	5.4|	gcc 9.3.0	|2.31|
|国产系统|UOS|	UOS	|4.19|	gcc 8.2.0|	2.27|
|CentOS	|CentOS7|	CentOS7|	3.10.0	|gcc 4.8.5	|2.17|
|CentOS|CentOS8|	CentOS8|	4.18.0	|gcc 8.2.0	|2.27|
|RHEL	|RHEL7|	RHEL7	|3.10.0	|gcc 4.8.5	|2.17|
|RHEL|RHEL8	|RHEL8|	4.18.0|	gcc 8.2	|2.27|
|RHEL|RHEL9	|RHEL9|	5.14|	gcc 11	|2.31|

# 2. 多架构编译

