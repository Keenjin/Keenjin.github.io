---
layout: post
title: Docker笔记
date: 2019-06-11
tags: docker
---

# CentOS下安装docker教程

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

# docker使用的代理问题

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

# 通过docker安装一个centos系统

```bash
# 安装7.3的centos版本
$ docker pull centos:7.3.1611

# 启动7.3系统
$ docker run -t -i centos:7.3.1611 /bin/bash
```

# 通过Dockerfile管理系统部署

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
```
