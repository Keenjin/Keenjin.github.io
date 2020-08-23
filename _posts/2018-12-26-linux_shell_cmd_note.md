---
layout: post
title: linux常用操作及问题笔记
date: 2018-12-26 
tags: linux  
---
  
> 实际操作过程中的积累

<!-- TOC -->

- [1 常用操作命令](#1-常用操作命令)
    - [压缩tar](#压缩tar)
    - [解压tar](#解压tar)
    - [显示当前目录pwd](#显示当前目录pwd)
    - [改变文件权限（可读、可写、可执行）chmod](#改变文件权限可读可写可执行chmod)
    - [运行某个sh脚本./](#运行某个sh脚本)
    - [打印内容echo](#打印内容echo)
    - [日期date](#日期date)
    - [时间同步](#时间同步)
    - [上传rz](#上传rz)
    - [下载sz](#下载sz)
    - [ubuntu设置代理](#ubuntu设置代理)
    - [查看系统设备的相关命令](#查看系统设备的相关命令)
    - [系统及性能监控相关工具](#系统及性能监控相关工具)
- [2 shell内使用的常用命令](#2-shell内使用的常用命令)
    - [$0表示当前sh脚本的文件路径（相对路径），这个本身输出的是一个字符串，可以用echo打印出来](#0表示当前sh脚本的文件路径相对路径这个本身输出的是一个字符串可以用echo打印出来)
    - [``或者$()，表示把命令框起来，shell解释的时候优先执行，并把结果作为输出](#或者表示把命令框起来shell解释的时候优先执行并把结果作为输出)
    - [dirname，取当前sh脚本](#dirname取当前sh脚本)
    - [$var或${var}，用于变量替换，var是一个变量，类似宏](#var或var用于变量替换var是一个变量类似宏)
    - [type、test辅助判断命令](#typetest辅助判断命令)
- [3 其他问题](#3-其他问题)
    - [windows下mingw64无法设置sh文件可执行（chmod失效）](#windows下mingw64无法设置sh文件可执行chmod失效)
- [4 vim常用操作](#4-vim常用操作)
    - [删除](#删除)
    - [复制粘贴](#复制粘贴)
    - [撤销操作](#撤销操作)
    - [撤销恢复](#撤销恢复)
    - [重复操作](#重复操作)
    - [移动光标](#移动光标)
    - [插入](#插入)
    - [替换字符串](#替换字符串)
- [5 ubuntu中python安装MYSQL模块问题](#5-ubuntu中python安装mysql模块问题)

<!-- /TOC -->

# 1 常用操作命令

## 压缩tar

命令：

```txt
# 压缩testdir目录

# -z 表示gzip格式
# -c 表示create
# -v 表示verbose，展示详情
# -f 表示目标是文件（这个一般都是f）

$ tar -zcvf testdir.tar.gz testdir
```

结果：

```txt
testdir/
testdir/test.txt
testdir/test2.txt
```

## 解压tar

命令：

```txt
# 解压testdir.tar.gz到testdir目录

# -x 表示extract，解压

$ tar -zxvf testdir.tar.gz testdir
```

结果：

```txt
testdir/
testdir/test.txt
testdir/test2.txt
```

## 显示当前目录pwd

命令：

```txt
pwd
```

结果：

```txt
/usr/home/keenjin
```

## 改变文件权限（可读、可写、可执行）chmod

命令：

```txt
chmod a+x test.sh
ll
```

结果：

```txt
-rwxr-xr-x 1 keenjin 1049089 26 12月 27 10:25 test.sh*
```

## 运行某个sh脚本./

命令：

```txt
chmod a+x test.sh
ll
./test.sh  #内容是mkdir test;ls
```

结果：

```txt
-rwxr-xr-x 1 keenjin 1049089 26 12月 27 10:25 test.sh*
test
```

## 打印内容echo

命令：

```txt
echo "test"
echo `pwd`
echo $0
```

结果：

```txt
test
/c/Users/keenjin/linux_shell
./shelltest.sh
```

## 日期date

命令：

```txt
date
date +%Y%m%d
date -d '+1 day ago' +%Y%m%d
date +%Y-%m-%d
```

结果：

```txt
2018年12月27日 15:13:15
20181227
20181226
2018-12-27
```

## 时间同步

命令：

```bash
# 查看时区
timedatectl status|grep 'Time zone'
# 设置与本地时间保持一致
timedatectl set-local-rtc 1
# 调整时区
timedatectl set-timezone Asia/Shanghai
date

# 如果发现时间还不同步，使用时间同步服务器
yum -y install ntpdate
ntpdate -u cn.ntp.org.cn
date

```

## 上传rz

命令：

```txt
rz
```

结果：

```txt
弹出一个对话框，选择要上传到xshell登录的服务器的文件
```

## 下载sz

命令：

```txt
sz test.py
```

结果：

```txt
弹出对话框，选择一个目录，用来存放下载的test.py文件
```

## ubuntu设置代理

/etc/apt/sources.list，修改这个文件里面所有deb 相关的，替换位自己的代理镜像源

## 查看系统设备的相关命令

lspci -v        查看PCI设备信息，比如设备驱动等

```txt
02:00.0 USB controller: VMware USB1.1 UHCI Controller (prog-if 00 [UHCI])
    Subsystem: VMware USB1.1 UHCI Controller
    Physical Slot: 32
    Flags: bus master, medium devsel, latency 64, IRQ 18
    I/O ports at 2080 [size=32]
    Capabilities: <access denied>
    Kernel driver in use: uhci_hcd
```

ifconfig        查看网卡信息，使用前，需要安装网卡设备工具。sudo apt install net-tools

```txt
ens33: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.2.129  netmask 255.255.255.0  broadcast 192.168.2.255
        inet6 fe80::aeca:330c:2369:d3fa  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:14:55:7a  txqueuelen 1000  (Ethernet)
        RX packets 297597  bytes 403950303 (403.9 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 125403  bytes 9052634 (9.0 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

## 系统及性能监控相关工具

```txt
top：Linux Process Monitoring
vmstat：Virtual Memory Statistics
lsof：List Open Files
tcpdump：Network Packet Analyzer
netstat：Network Statistics
Htop – Linux Process Monitoring
Iotop – Monitor Linux Disk I/O
Iostat – Input/Output Statistics
iptraf-ng – Real Time IP LAN Monitoring
```

# 2 shell内使用的常用命令

## $0表示当前sh脚本的文件路径（相对路径），这个本身输出的是一个字符串，可以用echo打印出来

```txt
echo $0
结果：
./shelltest.sh
```

## ``或者$()，表示把命令框起来，shell解释的时候优先执行，并把结果作为输出

```txt
$(dirname "$0")
结果：
/c/Users/keenjin/linux_shell
```

## dirname，取当前sh脚本

```txt
$(dirname "$0")
结果：
/c/Users/keenjin/linux_shell
```

## $var或${var}，用于变量替换，var是一个变量，类似宏

```txt
date=$(date +%Y%m%d)
echo "现在时间:${date}"
结果：
现在时间是:20181227
```

## type、test辅助判断命令

```bash
# type用于判断命令类型，是内部命令还是外部命令。如果命令不存在，type命令的退出错误码，就会执行if条件判断失败
# test用于测试后面的条件是否满足
print_cmd=""
if type printf > /dev/null; then
    print_cmd="printf"
elif test -x /usr/ucb/echo; then
    print_cmd="/usr/ucb/echo"
else
    print_cmd="echo"
fi
```

# 3 其他问题

## windows下mingw64无法设置sh文件可执行（chmod失效）

问题：

```txt
文件test.sh内容
---------------------------------------------------------
mkdir test
ls
---------------------------------------------------------
输入命令：
$ chmod a+x test.sh
$ ll
-rw-r--r-- 1 keenjin 1049089 15 12月 27 10:21 test.sh
```

解决：在sh文件头部，添加#!/bin/bash

```txt
文件test.sh内容
---------------------------------------------------------
#!/bin/bash
mkdir test
ls
---------------------------------------------------------
输入命令：
$ ll
-rwxr-xr-x 1 keenjin 1049089 26 12月 27 10:25 test.sh*
```

# 4 vim常用操作

## 删除

```txt
dd          删除当前行
D           删除光标到当前行末尾的字符
dG          向后删除到文本末尾
d^          向前删除到当前行开头
dgg         向前删除到文本开头
dw          向后删除一个单词（以空格间隔的，如果是空格，删除一个空格）
d10w        向后删除10个单词
```

## 复制粘贴

```txt
p           复制上一次删除的操作，粘贴到光标所在下一行
```

## 撤销操作

```txt
u           撤销前一次操作，可以不断往前回归撤销
```

## 撤销恢复

```txt
ctrl+r      恢复操作
```

## 重复操作

```txt
.           重复上一次操作，可以不断重复
```

## 移动光标

```txt
^           移动光标到行首
$           移动光标到行尾
G           移动到末行行首
w           移动到下一个单词首部，对于汉子，就是下一个汉子首部
e           移动到下一个单词尾部
```

## 插入

```txt
o           向下插入一行，并进入插入模式
O           向前插入一行，并进入插入模式
```

## 替换字符串

```txt
:%s/[str1]/[str2]/g      全局替换str1为str2，例如%s/deb http:\/\/us.archive.ubuntu.com\/ubuntu\//deb xxxxxxx/g，会将所有的http://us.archive.ubuntu.com/ubuntu/替换位xxxxxxx
```

# 5 ubuntu中python安装MYSQL模块问题

环境是python2.7  
安装命令：pip install mysql-python  
出现第一个问题：

```txt
Command "python setup.py egg_info" failed with error code 1 in /tmp/pip-build-_Livdw/mysql-python/
```

首先，需要升级setuptools，安装命令：pip install --upgrade setuptools  
然后，安装mysql环境，安装命令：sudo apt-get install libmysqlclient-dev
