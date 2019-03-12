---
layout: post
title: 常用开源分布式文件系统架构
date: 2019-03-12
tags: 经验  
---

# 单机文件系统的问题

（1）共享：无法同时为分布在多个机器中的应用提供访问，于是有了 NFS 协议，可以将单机文件系统通过网络的方式同时提供给多个机器访问。  

（2）容量：无法提供足够空间来存储数据，数据只好分散在多个隔离的单机文件系统里。  

（3）性能：无法满足某些应用需要非常高的读写性能要求，应用只好做逻辑拆分同时读写多个文件系统。  

（4）可靠性：受限于单个机器的可靠性，机器故障可能导致数据丢失。  

（5）可用性：受限于单个操作系统的可用性，故障或者重启等运维操作会导致不可用。  

参考链接：  
[https://blog.csdn.net/liuaigui/article/details/6284551](https://blog.csdn.net/liuaigui/article/details/6284551)  
[https://juejin.im/entry/5b39e494e51d4558b80b4db7](https://juejin.im/entry/5b39e494e51d4558b80b4db7)  

# GlusterFS（Cluster公司开发的POSIX分布式文件系统）

[png](/images/post/filesystem/1.png)  
整体组成：存储服务器（Brick Server）、客户端、NFS/Samba存储网关（有装GlusterFS客户端的，通过TCP/IP和InfiniBand RDMA告诉网络互连；没有安装的，通过NFS网关连接）  

特点：无元数据管理，所有存储节点都会保存完整的数据目录结构  

# GFS（Google，适合大文件存储）

[png](/images/post/filesystem/2.png)  
文件被划分为64MB的Chunk存储到几个ChunkServer上  

# HDFS（参照GFS设计的）

[png](/images/post/filesystem/3.png)
[png](/images/post/filesystem/4.png)

# MooseFS（参照GFS设计的）

参考链接：[https://blog.51cto.com/nolinux/1600890](https://blog.51cto.com/nolinux/1600890)

# JuiceFS

GFS、HDFS、MooseFS是针对硬件设备，需要考虑多副本存储。但是在公有云和私有云中，已经不需要考虑这些了，可以使用JuiceFS，针对HDFS和MooseFS的架构优化。  

# 几大架构出现的时间线

[png](/images/post/filesystem/5.png)

以GFS为代表的元数据和数据分离的系统搞设计，和以JuiceFS为代表的对象存储方式，避免分布式系统时的双层冗余。  

# 其他文件系统

TFS：小文件存储（不超过1M），适合图片等  
GridFS：MongoDB内置文件系统，提供文件操作的API利用NoSql数据库MongoDB存储文件，也适合小文件存储  
