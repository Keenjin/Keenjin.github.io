---
layout: post
title: ELK + FileBeat实现实时日志收集和分析系统
date: 2019-03-12
tags: Linux  
---

# 客户机安装FileBeat

```txt

```

# 服务器安装logstash

# 打通FileBeat和logstash

```txt
注意：关闭防火墙、ip配置、端口启动
```

# 准备Elasticsearch

## 下载安装Elasticsearch

```txt
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.6.1.rpm
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.6.1.rpm.sha512
shasum -a 512 -c elasticsearch-6.6.1.rpm.sha512
sudo rpm --install elasticsearch-6.6.1.rpm
```

## 启动Elasticsearch服务

```txt
sudo /bin/systemctl daemon-reload
sudo /bin/systemctl enable elasticsearch.service
sudo systemctl start elasticsearch.service
```

## 检查Elasticsearch服务运行状态

```txt
curl http://localhost:9200
```

![jpg](/images/post/elk/1.jpg)

