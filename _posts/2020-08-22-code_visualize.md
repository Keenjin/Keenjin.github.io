---
layout: post
title: 代码阅读可视化
date: 2020-08-22
tags: C++
---

## 背景

最近看一些开源代码，代码量之大需要较多的阅读时间，在梳理开源代码调用关系的时候，通常要花费大量时间，一直想能有一套可视化工具，将阅读的代码关系连接起来，将极大提升阅读效率。  

开源社区中普遍使用graphviz来进行图形绘制，使用其他工具，生成调用链。

## codeviz+graphviz（不好用，没有成功过）

网址：<https://github.com/petersenna/codeviz>  

通过介绍可知，它本来是作者为了分析linux内核源码所设计的，有着与大家一样的困惑的同时，不甘于把时间耗费在巨大的代码量中，而设计了这么个代码调用关系可视化工具。这个工具的实现原理，是采用定制gcc编译器（给gcc源码打patch），在生成c代码编译关系之时，生成调用信息，交给graphviz来绘制。也能对c++调用关系做处理。正因如此实现机制，在分析任何源码前，都需要明确对应的gcc使用版本，然后针对处理。通过实践来操作，发现无论如何都无法生成调用关系cdepn文件，实践环境：centos7 x64环境，使用gcc 7.4.0。有兴趣的同学，可以参考petersenna介绍的方式，安装试试。可以使用如下步骤：  

```bash
# 下载源码
git clone hhttps://github.com/petersenna/codeviz.git
cd codeviz

# 配置7.4.0版本安装
./configure --gcc=7.4.0 --gccgraph=/usr/local/gccgraph

# 如果下载失败，修改compilers/install_gcc-7.4.0.sh脚本中的FTP工具，为wget工具，当然，请确保已安装
# 这里会先下载gcc源码，然后打patch，然后安装gcc，并使用gcc编译codeviz。如果发现gcc编译失败，请先去compilers/gcc-graph/gcc-7.4.0/，然后运行contrib/download_pre.....脚本，下载依赖，然后重新make。这个时间非常久，建议睡前编译，当然，可以选择并行编译，请自行修改脚本。
make
make install
```

最后依旧没有成功，先记录一下，以后有需要再研究吧。

## doxygen+graphviz（非常赞）

doxygen是一个工程化工具，使用后，会感觉非常棒，不仅支持c/c++，还支持java等其他流行编程语言。基本使用方式非常简单。

```bash
# 安装
yum install graphviz -y
yum install doxygen -y

# 选择源码目录
cd squid-5

# 生成Doxyfile标准文件
doxygen -g

# 编辑Doxyfile，文件末尾增加如下：
HAVE_DOT               = YES
EXTRACT_ALL            = YES
EXTRACT_PRIVATE        = YES
EXTRACT_STATIC         = YES
CALL_GRAPH             = YES
CALLER_GRAPH           = YES
DISABLE_INDEX          = YES 
GENERATE_TREEVIEW      = YES
RECURSIVE              = YES

# 重新运行
doxygen Docxfile

# 生成html文件，在html文件夹，配合latex使用。我在虚拟机中生成的，为了外层方便使用，需要建立一个http server，使用python3非常方便
yum install python3 -y
cd squid-5/html/
python3 -m http.server 8080

# 在主机浏览器中，输入ip:8080，即可访问
```

效果如下：

![png](/images/post/squid/squid-viz.png)

