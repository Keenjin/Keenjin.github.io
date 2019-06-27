---
layout: post
title: ZooKeeper原理
date: 2019-05-08
tags: 后台框架
---

# ZooKeeper原理

<!-- TOC -->

- [ZooKeeper原理](#zookeeper原理)
    - [1. 分布式要解决的常规问题](#1-分布式要解决的常规问题)
    - [2. ZooKeeper的用途](#2-zookeeper的用途)

<!-- /TOC -->

## 1. 分布式要解决的常规问题

- 业务发现（service discovery）：找到分布式系统中存在那些可用的服务和节点
- 名字服务 （name service）：通过给定的名字知道到对应的资源
- 配置管理 （configuration management）：如何在分布式的节点中共享配置文件，保证一致性。
- 故障发现和故障转移 （failure detection and failover）：当某一个节点出故障的时候，如何检测到并通知其它节点， 或者把想用的服务转移到其它的可用节点
- 领导选举（leader election）：如何在众多的节点中选举一个领导者，来协调所有的节点
- 分布式的锁 （distributed exclusive lock）：如何通过锁在分布式的服务中进行同步
- 消息和通知服务 （message queue and notification）：如何在分布式的服务中传递消息，以通知的形式对事件作出主动的响应

## 2. ZooKeeper的用途

- 针对分布式服务的配置信息的统一管理维护：由于分布式服务下配置过于分散，不方便运维人员维护，而采用ZooKeeper可以统一管理
- 统一命名服务：统一的命名服务方便在分布式机器上查找信息
- 分布式服务同步：分布式服务针对资源的竞争，需要统一的同步管理机制
- 群组管理：特别适用于master/slave模型，可以方便管理群组，方便选举master

