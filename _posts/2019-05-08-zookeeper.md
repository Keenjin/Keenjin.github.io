---
layout: post
title: 【架构设计】ZooKeeper原理
date: 2019-05-08
tags: 架构设计
---

# ZooKeeper原理

<!-- TOC -->

- [ZooKeeper原理](#zookeeper原理)
    - [1. 分布式要解决的常规问题](#1-分布式要解决的常规问题)
    - [2. ZooKeeper的用途及特性](#2-zookeeper的用途及特性)
    - [3. 安装使用](#3-安装使用)
        - [3.1. Windows下的安装运行](#31-windows下的安装运行)
        - [3.2. Docker下的安装运行](#32-docker下的安装运行)
        - [3.3. 命令行的使用](#33-命令行的使用)
        - [3.4. Watcher机制](#34-watcher机制)
        - [3.5. 基本调用代码](#35-基本调用代码)
    - [4. 应用案例](#4-应用案例)
        - [4.1. 服务发现](#41-服务发现)
            - [4.1.1. 使用临时节点，查询服务的ip和端口，以及其他一些配置信息](#411-使用临时节点查询服务的ip和端口以及其他一些配置信息)
                - [4.1.1.1. zookeeper的通用接口](#4111-zookeeper的通用接口)
                - [4.1.1.2. 服务注册端](#4112-服务注册端)
                - [4.1.1.3. 服务发现端](#4113-服务发现端)
            - [4.1.2. 设置节点和子节点监控，让动态注册服务和配置改变，能被客户端接收到](#412-设置节点和子节点监控让动态注册服务和配置改变能被客户端接收到)
            - [4.1.3. 故障转移](#413-故障转移)
        - [4.2. 故障发现和转移](#42-故障发现和转移)
        - [4.3. 统一配置管理](#43-统一配置管理)
        - [4.4. 领导选举](#44-领导选举)
        - [4.5. 消息同步](#45-消息同步)

<!-- /TOC -->

## 1. 分布式要解决的常规问题

- 业务发现（service discovery）：找到分布式系统中存在那些可用的服务和节点
- 名字服务 （name service）：通过给定的名字知道到对应的资源
- 配置管理 （configuration management）：如何在分布式的节点中共享配置文件，保证一致性。
- 故障发现和故障转移 （failure detection and failover）：当某一个节点出故障的时候，如何检测到并通知其它节点， 或者把想用的服务转移到其它的可用节点
- 领导选举（leader election）：如何在众多的节点中选举一个领导者，来协调所有的节点
- 分布式的锁 （distributed exclusive lock）：如何通过锁在分布式的服务中进行同步
- 消息和通知服务 （message queue and notification）：如何在分布式的服务中传递消息，以通知的形式对事件作出主动的响应

## 2. ZooKeeper的用途及特性

用途：

- 针对分布式服务的配置信息的统一管理维护：由于分布式服务下配置过于分散，不方便运维人员维护，而采用ZooKeeper可以统一管理
- 统一命名服务：统一的命名服务方便在分布式机器上查找信息
- 分布式服务同步：分布式服务针对资源的竞争，需要统一的同步管理机制
- 群组管理：特别适用于master/slave模型，可以方便管理群组，方便选举master

特性：

- 通过一堆的znodes分级存储
- 存储在内存中而非磁盘中，因此具有高吞吐和低延迟特性
- 更新操作顺序一致性、原子性、可靠性（确保所有端的更新操作都覆盖到）、及时性（确保更新信息在短时间内迅速覆盖）

## 3. 安装使用

### 3.1. Windows下的安装运行

```bash
# 下载zookeeper安装包：https://www.apache.org/dyn/closer.cgi/zookeeper/
# 解压
# 确保安装了jdk，并且设置了JAVA_HOME环境变量
# 拷贝一份conf/zoo_sample.cfg为zoo.cfg
# 运行bin/zkServer.Cmd
```

### 3.2. Docker下的安装运行

```bash
# 安装
[root@client-centos7 centosfull]# docker pull zookeeper

# 普通运行
[root@client-centos7 centosfull]# docker run -d --name zookeeper-01 --restart always zookeeper:latest

# 查看ip地址
[root@client-centos7 centosfull]# docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                                    NAMES
d7c022fa6375        zookeeper:latest    "/docker-entrypoint.…"   26 minutes ago      Up 26 minutes       2181/tcp, 2888/tcp, 3888/tcp, 8080/tcp   zookeeper-01
[root@client-centos7 centosfull]# docker inspect --format '{{ .NetworkSettings.Networks.bridge.IPAddress }}' d7c022fa6375
172.17.0.2

# 想要给zookeeper进行配置，进入系统，修改conf目录下的zoo.cfg（如果没有此文件，从安装包里面去找）
docker exec -it zookeeper-01 /bin/bash
```

### 3.3. 命令行的使用

```bash
# 使用zkCli.cmd连接zkServer
zkCli.cmd -server 127.0.0.1:2181

[zk: 127.0.0.1:2181(CONNECTED) 0]

# ls命令，查看当前根有哪些配置
ls /

[zookeeper]     # 表示当前的根只有zookeeper这一个配置项

# create、set命令，尝试创建一个结点并设置值。get命令，尝试获取值
create /baidu_url
set /baidu_url www.baidu.com
get /baidu_url

www.baidu.com   # 存储的值

```

### 3.4. Watcher机制

- 可以设置节点数据watcher和子节点watcher
- 节点数据watcher：GetW、ExistsW。针对设置的监控节点的变化，包括create、delete、set操作，分别对应的事件
- 子节点watcher：ChildrenW。针对节点的一级子节点的变化（包括当前节点的变化），包括create、delete操作
- 监控是一次性机制，监控完的事件，只使用一次，如果想再次监控，则需要再次设置监控机制

### 3.5. 基本调用代码

```go
package main

import (
	"fmt"
	"time"

	"github.com/samuel/go-zookeeper/zk"
)

func main() {
	c, _, err := zk.Connect([]string{"127.0.0.1"}, time.Second) //*10)
	if err != nil {
		panic(err)
	}
	children, stat, ch, err := c.ChildrenW("/")
	if err != nil {
		panic(err)
	}
	fmt.Printf("%+v %+v\n", children, stat)
	e := <-ch
	fmt.Printf("%+v\n", e)
}
```

```go

```

## 4. 应用案例

### 4.1. 服务发现

![jpg](/images/post/后台框架/zoo_service_discover.jpg)

#### 4.1.1. 使用临时节点，查询服务的ip和端口，以及其他一些配置信息

##### 4.1.1.1. zookeeper的通用接口

```go
package zookeeper

import (
	"encoding/json"
	"fmt"
	"time"

	"github.com/samuel/go-zookeeper/zk"
)

// ServiceNode 服务节点
type ServiceNode struct {
	Name string
	Host string
	Port string
}

// Discovery 服务发现
type Discovery struct {
	zkServers []string
	zkRoot    string
	conn      *zk.Conn
}

// Create def
func (d *Discovery) Create(zkServers []string, zkRoot string, timeout int) error {
	d.zkServers = zkServers
	d.zkRoot = zkRoot

	// 连接服务器
	conn, _, err := zk.Connect(zkServers, time.Duration(timeout)*time.Second)
	if err != nil {
		return err
	}
	d.conn = conn

	// 创建根节点
	err = d.ensureRoot()
	if err != nil {
		d.conn.Close()
		return err
	}

	return nil
}

// Close def
func (d *Discovery) Close() {
	d.conn.Close()
}

// CreateNode def
func (d *Discovery) CreateNode(name string, data []byte, flag int32) error {
	path := d.zkRoot + "/" + name
	exists, s, err := d.conn.Exists(path)
	if err != nil {
		return err
	}

	if !exists {
		// 一个临时节点
		_, err := d.conn.Create(path, data, flag, zk.WorldACL(zk.PermAll))
		if err != nil && err != zk.ErrNodeExists {
			return err
		}
	} else {
		_, err = d.conn.Set(path, data, s.Version)
		if err != nil {
			return err
		}
	}
	return nil
}

// Register 服务注册
func (d *Discovery) Register(srvGroup string, node *ServiceNode) error {
	err := d.CreateNode(srvGroup, nil, 0)
	if err != nil {
		return err
	}

	data, err := json.Marshal(node)
	if err != nil {
		return err
	}

	err = d.CreateNode(srvGroup+"/"+node.Name, data, zk.FlagEphemeral)
	if err != nil {
		return err
	}

	return nil
}

func (d *Discovery) ensureRoot() error {
	exists, _, err := d.conn.Exists(d.zkRoot)
	if err != nil {
		return err
	}

	if !exists {
		_, err := d.conn.Create(d.zkRoot, []byte(""), 0, zk.WorldACL(zk.PermAll))
		if err != nil && err != zk.ErrNodeExists {
			return err
		}
	}

	return nil
}

// GetNodeInfo 获取节点信息
func (d *Discovery) GetNodeInfo(name string) (*ServiceNode, error) {
	path := d.zkRoot + "/" + name
	data, _, err := d.conn.Get(path)
	if err != nil {
		return nil, err
	}

	node := new(ServiceNode)
	err = json.Unmarshal(data, node)
	if err != nil {
		return nil, err
	}

	return node, nil
}

// GetServiceNodes 获取所有子节点信息
func (d *Discovery) GetServiceNodes(name string) ([]*ServiceNode, error) {
	path := d.zkRoot + "/" + name

	// 获取子节点名称
	childs, _, err := d.conn.Children(path)
	if err != nil {
		if err == zk.ErrNoNode {
			return []*ServiceNode{}, nil
		}
		return nil, err
	}
	nodes := []*ServiceNode{}
	for _, child := range childs {
		fullpath := path + "/" + child
		data, _, err := d.conn.Get(fullpath)
		if err != nil {
			if err == zk.ErrNoNode {
				continue
			}
			return nil, err
		}
		node := new(ServiceNode)
		err = json.Unmarshal(data, node)
		if err != nil {
			return nil, err
		}
		nodes = append(nodes, node)
	}

	return nodes, nil
}

// Run def
func (d *Discovery) Run() {
	select {}
}
```

##### 4.1.1.2. 服务注册端

```go
package main

import (
	"fmt"
	"zookeeper"
)

func server() {
	fmt.Println("Server!!!")
	servers := []string{"127.0.0.1:2181"}
	ds := new(zookeeper.Discovery)
	err := ds.Create(servers, "/Services", 10)
	if err != nil {
		panic(err)
	}
	defer ds.Close()

    // 注册一个DbMaster服务。每一个服务起来的时候，自己向zookeeper注册一下
	dbnode := &zookeeper.ServiceNode{Name: "DbMaster", Host: "127.0.0.1", Port: "11200"}
	if err := ds.Register("Db", dbnode); err != nil {
		panic(err)
	}

	ds.Run()
}

func main() {
	go server()
	select {}
}
```

##### 4.1.1.3. 服务发现端

```go
package main

import (
	"fmt"
	"zookeeper"
)

func client() {
	fmt.Println("Client!!!")
	servers := []string{"127.0.0.1:2181"}
	ds := new(zookeeper.Discovery)
	err := ds.Create(servers, "/Services", 10)
	if err != nil {
		panic(err)
	}
	defer ds.Close()

	// 获取节点信息：/Services/Db/DbMaster
	node, err := ds.GetNodeInfo("Db/DbMaster")
	if err != nil {
		panic(err)
	}
	fmt.Println(node.Name, node.Host, node.Port)

	ds.Run()
}

func main() {
	go client()
	select {}
}
```

#### 4.1.2. 设置节点和子节点监控，让动态注册服务和配置改变，能被客户端接收到

#### 4.1.3. 故障转移

### 4.2. 故障发现和转移

### 4.3. 统一配置管理

### 4.4. 领导选举

### 4.5. 消息同步
