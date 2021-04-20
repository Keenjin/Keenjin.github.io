---
layout: post
title: 【架构设计】ELK + FileBeat实现实时日志收集和分析系统
date: 2019-03-12
tags: 架构设计 
---

<!-- TOC -->

- [客户机安装FileBeat](#客户机安装filebeat)
- [服务器安装logstash](#服务器安装logstash)
- [打通FileBeat和logstash](#打通filebeat和logstash)
- [准备Elasticsearch](#准备elasticsearch)
- [安装kibana（web服务器）](#安装kibanaweb服务器)

<!-- /TOC -->

# 客户机安装FileBeat

```txt
简单的安装，以及配置需要监控的日志目录，以及输出采用logstash还是其他，如果是logstash，需要配置输出的IP
```

# 服务器安装logstash

```txt
主要是需要配置beats的监听端口，以及输出的IP
```

# 打通FileBeat和logstash

```txt
注意：关闭防火墙、ip配置、端口启动
```

# 准备Elasticsearch

```txt
安装需要在非root权限下进行，root权限无法运行
```

# 安装kibana（web服务器）

```txt
这里启动了一个web服务器，设置了一个特定的端口，然后可以在我们自己的客户机的浏览器中，访问这个IP:端口，即可
```

