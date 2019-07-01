---
layout: post
title: Docker笔记
date: 2019-06-11
tags: docker
---

<!-- TOC -->

- [1. CentOS下安装docker教程](#1-centos下安装docker教程)
- [2. docker使用的代理问题](#2-docker使用的代理问题)
- [3. 通过docker安装一个centos系统](#3-通过docker安装一个centos系统)
- [4. 通过Dockerfile管理系统部署](#4-通过dockerfile管理系统部署)
- [5. Docker不要跑多个进程](#5-docker不要跑多个进程)
- [6. Docker的常用命令](#6-docker的常用命令)
- [7. Docker下rabbitmq实战](#7-docker下rabbitmq实战)
- [8. Docker镜像迁移](#8-docker镜像迁移)

<!-- /TOC -->

# 1. CentOS下安装docker教程

参考：<https://docs.docker.com/install/linux/docker-ce/centos/>

```bash
# 安装必要工具集
$ yum -y install -y yum-utils device-mapper-persistent-data lvm2

# 下载docker源（注意，yum-config-manager是一个python程序，仅支持python2，如果是python3的话，需要改一下这个文件的头部）
$ yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

# 安装docker
$ yum -y install docker-ce docker-ce-cli containerd.io

# 启动docker
$ systemctl start docker

```

# 2. docker使用的代理问题

```bash
# 在centos中，/etc/profile中可以设置全局的网络代理，但是docker却不会使用此代理上网。需要设置单独的代理
# 新建一个服务目录
$ mkdir /etc/systemd/system/docker.service.d

# 新建一个配置文件
$ vim /etc/systemd/system/docker.service.d/http-proxy.conf

# 填写如下内容
Environment="HTTP_PROXY=http://proxy.example.com:80/"
Environment="HTTPS_PROXY=http://proxy.example.com:80/"
Environment="NO_PROXY=localhost,127.0.0.0/8,docker-registry.somecorporation.com"
```

# 3. 通过docker安装一个centos系统

```bash
# 安装7.3的centos版本
$ docker pull centos:7.3.1611

# 启动7.3系统
$ docker run -t -i centos:7.3.1611 /bin/bash
```

# 4. 通过Dockerfile管理系统部署

```bash
# 在host机器任意地方，建一个目录DockerApp1
$ mkdir /opt/DockerHome/DockerApp1

# 创建一个Dockerfile文件
$ vim Dockerfile

# 写下以下内容（主要使用FROM命令和RUN命令）
FROM centos:7.3.1611
MAINTAINER Docker Keenjin <xxx@qq.com>
COPY epel.repo /etc/yum.repos.d/
RUN yum -y install wget
RUN wget -O /etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7 https://archive.fedoraproject.org/pub/epel/RPM-GPG-KEY-EPEL-7
RUN yum -y upgrade
RUN yum -y install epel-release
RUN yum -y install net-tools zip unzip lrzsz vim
RUN yum -y install go

# 编译docker
docker build -t workos:v1 .

# 运行docker
docker run -it workos:v1

# 注意：
# 使用RUN的时候，每一条RUN运行，都会启动一个bash，执行完命令，会自动commit，这样镜像的层级就新增一层。
# 由于层级的增加会导致无用的浪费，最好RUN尽可能少，把命令都放在一个bash里面执行完成，如下：
RUN yum -y install wget \
    && yum -y upgrade \
    && wget xxxxxxx
```

```bash
# Dockerfile案例
FROM centos:latest
MAINTAINER Docker Keenjin <Keenjin@tencent.com>

# 先修改环境变量，确保可以正常联网
ENV http_proxy http://web-proxy.tencent.com:8080
ENV https_proxy http://web-proxy.tencent.com:8080

# 安装基础设备
RUN yum -y install wget

# 安装镜像源和必要工具集
COPY epel.repo /etc/yum.repos.d/
RUN wget -O /etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7 https://archive.fedoraproject.org/pub/epel/RPM-GPG-KEY-EPEL-7 \
    && yum -y upgrade \
    && yum -y install epel-release \
    && yum -y install net-tools zip unzip lrzsz vim iproute

# 安装go开发套件
RUN wget https://dl.google.com/go/go1.12.6.linux-amd64.tar.gz \
    && tar -C /usr/local -xzf go1.12.6.linux-amd64.tar.gz
ENV PATH $PATH:/usr/local/go/bin
ENV GOROOT /usr/local/go
RUN mkdir /opt/go
ENV GOPATH /opt/go

# 安装rabbitmq
RUN curl -s https://packagecloud.io/install/repositories/rabbitmq/rabbitmq-server/script.rpm.sh | bash \
    && curl -s https://packagecloud.io/install/repositories/rabbitmq/erlang/script.rpm.sh | bash \
    && yum -y install erlang \
    && yum -y install rabbitmq-server
```

# 5. Docker不要跑多个进程

```bash
# 今天本来想把docker当作虚拟机使用，发现这里踩坑比较严重。
# docker本质是当作一个微服务使用，本质而言，一个docker运行一个服务，只是我们把所有的相关依赖，全部可以通过Dockerfile打包起来，这样一个容器一个服务，就可以跑在所有其他任何地方。
```

# 6. Docker的常用命令

```bash
# 查看当前有哪些镜像（通过docker pull拉取下来的，以及通过docker build根据Dockerfile构造的）
docker image ls
# 针对镜像的删除操作
docker image rm -f xxx（这里xxx可以是repository:tag，也可以是imageid，其中，imageid可以只填写部分）
# 查看当前运行的容器进程
docker ps -a
# 查看容器的所有配置信息（比如网络、ip等）
docker inspect 容器id
```

# 7. Docker下rabbitmq实战

```bash
# 对于一般场景，使用rabbitmq:latest即可，但是如果想要management，能通过web访问rabbitmq，就需要使用rabbitmq:3-management
docker pull rabbitmq:3-management

# 运行rabbitmq容器（-d表示后台运行，-rm表示容器退出后自动清除，--hostname是由于rabbitmq会根据主机名创建节点，如果用默认，可能多个节点名一致不好区分。）
docker run -d --rm --hostname keen-master --name rabbit-01 rabbitmq:3-management

# 按照上面的方式，如果想要访问docker内的rabbitmqmgr，在虚拟机外部（windows机器），建立路由表，能将172.17.0.0网段的所有请求，都转到192.168.2.132（docker宿主机）上去。
# 这种方式，直接访问http://172.17.0.2:15672即可
route add 172.17.0.0 mask 255.255.255.0 192.168.2.132

# 如果不使用路由表，也可以直接建立端口转发，使用-p参数，如下，直接使用http://192.168.2.132:8080，就可以访问docker内的rabbitmq web服务
docker run -d --rm --hostname keen-master --name rabbit-01 -p 8080:15672 rabbitmq:3-management

# 如果不设置ip，docker会自动分配172号段的ip地址给每一个容器。如果想要ip自己可控，则可以自己传递一个固定ip地址进去
# 参考这里的：https://www.jianshu.com/p/19cd3654e4a4
# linux下的网络模型，简单来说，有网桥和主机模式两种。docker默认使用的网桥模型，docker启动的时候，就建立了默认网桥，如下：
[root@client-centos7 dockerhome]# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
4a7b5cd62362        bridge              bridge              local
64187f1770f6        host                host                local
324eb80ac409        none                null                local
# 其中，bridge是docker默认建立的网桥，通过docker network inspect bridge，可以查看详情，如下：
[root@client-centos7 dockerhome]# docker network inspect bridge
[
    {
        "Name": "bridge",
        "Id": "4a7b5cd623620a809a8b3fc1d766591b8eaf1dc18b6a0d45e20b15fb3ecd607f",
        "Created": "2019-06-12T08:02:17.546758645-08:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.0.0/16",
                    "Gateway": "172.17.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {},
        "Options": {
            "com.docker.network.bridge.default_bridge": "true",
            "com.docker.network.bridge.enable_icc": "true",
            "com.docker.network.bridge.enable_ip_masquerade": "true",
            "com.docker.network.bridge.host_binding_ipv4": "0.0.0.0",
            "com.docker.network.bridge.name": "docker0",
            "com.docker.network.driver.mtu": "1500"
        },
        "Labels": {}
    }
]
# 可以看到，子网是172.17号段，如果使用这个默认的网桥，来设置固定ip地址，如下：
[root@client-centos7 dockerhome]# docker run -d --rm --hostname keen-master --name rabbit-01 -p 8080:15672 --network bridge --ip 172.17.0.100 rabbitmq:3-management
07da4b8138ae70d8760f30643d1dcb88b9145c458a5e429db87774d5819458ad
docker: Error response from daemon: user specified IP address is supported on user defined networks only.
# 很明显，可以看到，我们就不能使用默认的网桥，如果要指定固定ip，就必须使用自定义网桥。这里大概意思是，默认网桥留给自动分配ip使用。
# 创建自定义网桥，如下：
[root@client-centos7 dockerhome]# docker network create --subnet 192.168.0.0/24 --gateway 192.168.0.1 keennet
459e55e82c169ada4d877c1809f1e730fa498cd6c0b4de0c8ffa09cd15d6681f
[root@client-centos7 dockerhome]# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
4a7b5cd62362        bridge              bridge              local
64187f1770f6        host                host                local
459e55e82c16        keennet             bridge              local
324eb80ac409        none                null                local
# 这样，就建立的自己的网桥keennet，查看详情如下：
[root@client-centos7 dockerhome]# docker network inspect keennet
[
    {
        "Name": "keennet",
        "Id": "459e55e82c169ada4d877c1809f1e730fa498cd6c0b4de0c8ffa09cd15d6681f",
        "Created": "2019-06-21T03:40:32.064733752-08:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "192.168.0.0/24",
                    "Gateway": "192.168.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {},
        "Options": {},
        "Labels": {}
    }
]
# 也可以使用ifconfig查看
[root@client-centos7 dockerhome]# ifconfig
br-459e55e82c16: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 192.168.0.1  netmask 255.255.255.0  broadcast 192.168.0.255
        ether 02:42:fa:82:1b:7a  txqueuelen 0  (Ethernet)
        RX packets 2203683  bytes 3100652975 (2.8 GiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 1337767  bytes 113125240 (107.8 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

docker0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 172.17.255.255
        ether 02:42:f7:dd:7e:69  txqueuelen 0  (Ethernet)
        RX packets 821346  bytes 37828065 (36.0 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 1587502  bytes 2531736409 (2.3 GiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

ens33: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.2.132  netmask 255.255.255.0  broadcast 192.168.2.255
        inet6 fe80::9422:95b:a1fc:ad87  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:49:2a:69  txqueuelen 1000  (Ethernet)
        RX packets 2203683  bytes 3100652975 (2.8 GiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 1337767  bytes 113125240 (107.8 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 121217  bytes 10667096 (10.1 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 121217  bytes 10667096 (10.1 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

virbr0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 192.168.122.1  netmask 255.255.255.0  broadcast 192.168.122.255
        ether 52:54:00:7b:4d:e7  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
# 第一个就是我们的网桥信息。这样，就可以用我们自定义的网桥，来设置固定ip
docker run -d --rm --hostname keen-master --name rabbit-01 -p 8080:15672 --network keennet --ip 192.168.0.100 rabbitmq:3-management

# 因为默认的账户guest，只能localhost访问（宿主机内的虚拟机访问除外），所以我们也需要设置用户名和密码
docker run -d --rm --hostname keen-master --name rabbit-01 -p 8080:15672 --network keennet --ip 192.168.0.100 --env RABBITMQ_DEFAULT_USER=keen-rabbit --env RABBITMQ_DEFAULT_PASS=keen123 rabbitmq:3-management
```

# 8. Docker镜像迁移

```bash
# 列出已有镜像
[root@client-centos7 centos7.6]# docker image ls
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
rabbitmq            3-management        6ffc11daa8d0        5 days ago          186MB
rabbitmq            latest              35d90e3cc645        5 days ago          156MB
centos              latest              9f38484d220f        3 months ago        202MB

# 镜像备份
[root@client-centos7 centos7.6]# docker save 6ffc11daa8d0 | gzip > rabbitmq-3-management.tar.gz

# 镜像还原
[root@localhost ~]# docker load -i rabbitmq-3-management.tar.gz
```
