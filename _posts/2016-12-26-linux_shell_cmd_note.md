---
layout: post
title: linux常用命令手记
date: 2018-12-26 
tags: 经验  
---
  
> 这里，只有实际操作过程中的积累

# 1 文件目录操作命令
## 压缩tar
命令：
```
# 压缩testdir目录

# -z 表示gzip格式
# -c 表示create
# -v 表示verbose，展示详情
# -f 表示目标是文件（这个一般都是f）

$ tar -zcvf testdir.tar.gz testdir
```
结果：
```
testdir/
testdir/test.txt
testdir/test2.txt
```
## 解压tar
命令：
```
# 解压testdir.tar.gz到testdir目录

# -x 表示extract，解压

$ tar -zxvf testdir.tar.gz testdir
```
结果：
```
testdir/
testdir/test.txt
testdir/test2.txt
```

## 显示当前目录pwd
命令：
```
$ pwd
```
结果：
```
/usr/home/keenjin
```


# 2 Shell脚本常用环境变量
