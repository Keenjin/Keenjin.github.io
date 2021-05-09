---
layout: post
title: 【架构设计】Elasticsearch总结
date: 2019-08-01
tags: 架构设计
---

<!-- TOC -->

- [1. 安装部署](#1-安装部署)
    - [1.1. 准备环境](#11-准备环境)
    - [1.2. 安装ES](#12-安装es)
    - [1.2. 配置](#12-配置)
    - [1.3. 开机启动](#13-开机启动)
        - [Step1：创建es服务系统环境变量配置文件（/etc/sysconfig/elasticsearch）](#step1创建es服务系统环境变量配置文件etcsysconfigelasticsearch)
        - [Step2：创建es服务（/etc/systemd/system/elasticsearch.service）](#step2创建es服务etcsystemdsystemelasticsearchservice)
        - [Step3：权限更改](#step3权限更改)
        - [Step4：设置开机启动](#step4设置开机启动)
        - [Step4：启动es](#step4启动es)
- [2. 增删改查操作](#2-增删改查操作)
    - [2.1. 普通增删改查](#21-普通增删改查)
- [3. 原理](#3-原理)

<!-- /TOC -->

# 1. 安装部署

## 1.1. 准备环境

- 宿主机：tlinux
- JDK：ES捆绑自带了一个jdk版本，安装的时候附带一起安装

## 1.2. 安装ES

```bash
# 安装
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.3.0-linux-x86_64.tar.gz
tar -xzf elasticsearch-7.3.0-linux-x86_64.tar.gz -C /usr/local

# 增加用户keen（es不能用root用户启动）
useradd keen

# 给/usr/local/elasticsearch-7.3.0目录改为keen的权限
chown -R keen /usr/local/elasticsearch-7.3.0

# 后台运行es（如果需要前台运行，不要加-d）
su keen
cd /usr/local/elasticsearch-7.3.0
bin/elasticsearch -d

# 测试是否启动正常
curl -X GET "localhost:9200/"

# >>>>>>>output
{
  "name" : "VM_15_61_centos",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "kYzCQkb-QO2s7o-0Te_dAQ",
  "version" : {
    "number" : "7.3.0",
    "build_flavor" : "default",
    "build_type" : "tar",
    "build_hash" : "de777fa",
    "build_date" : "2019-07-24T18:30:11.767338Z",
    "build_snapshot" : false,
    "lucene_version" : "8.1.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
# <<<<<<<
```

## 1.2. 配置

配置选项，参考：<https://www.elastic.co/guide/en/elasticsearch/reference/7.3/important-settings.html>  

## 1.3. 开机启动

### Step1：创建es服务系统环境变量配置文件（/etc/sysconfig/elasticsearch）

```ini
#######################
#    Elasticsearch    #
#######################

# Elasticsearch home directory
ES_HOME=/usr/local/elasticsearch-7.3.0

# Elasticsearch Java path
JAVA_HOME=/usr/local/elasticsearch-7.3.0/jdk
CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JAVA_HOMR/jre/lib

# Elasticsearch configuration directory。注意，这里必须写绝对路径，不能使用$ES_HOME
ES_PATH_CONF=/usr/local/elasticsearch-7.3.0/config

# Elasticsearch PID directory。设置了这个，才能正常Stop掉elasticsearch进程
PID_DIR=/var/run/elasticsearch

#############################
#   Elasticsearch Service   #
#############################

# SysV init.d
# The number of seconds to wait before checking if elasticsearch started successfully as a daemon process
ES_STARTUP_SLEEP_TIME=5

################################
#   Elasticsearch Properties   #
################################
# Specifies the maximum file descriptor number that can be opened by this process
# When using Systemd,this setting is ignored and the LimitNOFILE defined in
# /usr/lib/systemd/system/elasticsearch.service takes precedence
#MAX_OPEN_FILES=65536

# The maximum number of bytes of memory that may be locked into RAM
# Set to "unlimited" if you use the 'bootstrap.memory_lock: true' option
# in elasticsearch.yml.
# When using Systemd,LimitMEMLOCK must be set in a unit file such as
# /etc/systemd/system/elasticsearch.service.d/override.conf.
#MAX_LOCKED_MEMORY=unlimited

# Maximum number of VMA(Virtual Memory Areas) a process can own
# When using Systemd,this setting is ignored and the 'vm.max_map_count'
# property is set at boot time in /usr/lib/sysctl.d/elasticsearch.conf
#MAX_MAP_COUNT=262144
```

### Step2：创建es服务（/etc/systemd/system/elasticsearch.service）

```ini
[Unit]
Description=elasticsearch
Documentation=http://www.elastic.co
Wants=network-online.target
After=network-online.target

[Service]
EnvironmentFile=/etc/sysconfig/elasticsearch
User=keen
LimitNOFILE=100000
LimitNPROC=100000
# 这里的exe路径，必须为绝对路径。因为是service，所以无需增加-d参数
ExecStart=/usr/local/elasticsearch-7.3.0/bin/elasticsearch -p /var/run/elasticsearch/elasticsearch.pid

# StandardOutput is configured to redirect to journalctl since
# some error messages may be logged in standard output before
# elasticsearch logging system is initialized. Elasticsearch
# stores its logs in /var/log/elasticsearch and does not use
# journalctl by default. If you also want to enable journalctl
# logging, you can simply remove the "quiet" option from ExecStart.
# 这里使用journal，因此，可以使用journalctl -u elasticsearch.service查看输出日志
StandardOutput=journal
StandardError=inherit

# Specifies the maximum file descriptor number that can be opened by this process
LimitNOFILE=65536

# Specifies the maximum number of process
LimitNPROC=4096

# Specifies the maximum size of virtual memory
LimitAS=infinity

# Specifies the maximum file size
LimitFSIZE=infinity

# Disable timeout logic and wait until process is stopped
TimeoutStopSec=0

# SIGTERM signal is used to stop the Java process
KillSignal=SIGTERM

# Send the signal only to the JVM rather than its control group
KillMode=process

# Java process is never killed
SendSIGKILL=no

# When a JVM receives a SIGTERM signal it exits with code 143
# SuccessExitStatus=143

[Install]
WantedBy=multi-user.target
```

### Step3：权限更改

```bash
chmod +x /etc/systemd/system/elasticsearch.service
chown -R keen /var/run/elasticsearch
```

### Step4：设置开机启动

```bash
systemctl daemon-reload
systemctl enable elasticsearch.service
```

### Step4：启动es

```bash
systemctl start elasticsearch
systemctl status elasticsearch

# 查看错误日志
journalctl -u elaticsearch.service
```

# 2. 增删改查操作

## 2.1. 普通增删改查



# 3. 原理

