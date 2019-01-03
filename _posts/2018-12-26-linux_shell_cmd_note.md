---
layout: post
title: linux常用操作及问题笔记
date: 2018-12-26 
tags: 经验  
---
  
> 实际操作过程中的积累

# 1 常用操作命令
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
## 改变文件权限（可读、可写、可执行）chmod
命令：
```
$ chmod a+x test.sh
$ ll
```
结果：
```
-rwxr-xr-x 1 keenjin 1049089 26 12月 27 10:25 test.sh*
```
## 运行某个sh脚本./
命令：
```
$ chmod a+x test.sh
$ ll
$ ./test.sh  #内容是mkdir test;ls
```
结果：
```
-rwxr-xr-x 1 keenjin 1049089 26 12月 27 10:25 test.sh*
test
```
## 打印内容echo
命令：
```
$ echo "test"
$ echo `pwd`
$ echo $0
```
结果：
```
test
/c/Users/keenjin/linux_shell
./shelltest.sh
```
## 日期date
命令：
```
$ date
$ date +%Y%m%d
$ date -d '+1 day ago' +%Y%m%d
$ date +%Y-%m-%d
```
结果：
```
2018年12月27日 15:13:15
20181227
20181226
2018-12-27
```
## 上传rz
命令：
```
$ rz
```
结果：
```
弹出一个对话框，选择要上传到xshell登录的服务器的文件
```
## 下载sz
命令：
```
$ sz test.py
```
结果：
```
弹出对话框，选择一个目录，用来存放下载的test.py文件
```

# 2 shell内使用的常用命令
## $0表示当前sh脚本的文件路径（相对路径），这个本身输出的是一个字符串，可以用echo打印出来
```
echo $0
结果：
./shelltest.sh
```
## ``或者$()，表示把命令框起来，shell解释的时候优先执行，并把结果作为输出
```
$(dirname "$0")
结果：
/c/Users/keenjin/linux_shell
```
## dirname，取当前sh脚本
```
$(dirname "$0")
结果：
/c/Users/keenjin/linux_shell
```
## $var或${var}，用于变量替换，var是一个变量，类似宏
```
date=$(date +%Y%m%d)
echo "现在时间:${date}"
结果：
现在时间是:20181227
```

# 3 其他问题
## windows下mingw64无法设置sh文件可执行（chmod失效）
问题：
```
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
```
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
```
dd          删除当前行
D           删除光标到当前行末尾的字符
dG          向后删除到文本末尾
d^          向前删除到当前行开头
dgg         向前删除到文本开头
dw          向后删除一个单词（以空格间隔的，如果是空格，删除一个空格）
d10w        向后删除10个单词
```
## 复制粘贴
```
p           复制上一次删除的操作，粘贴到光标所在下一行
```
## 撤销操作
```
u           撤销前一次操作，可以不断往前回归撤销
```
## 撤销恢复
```
ctrl+r      恢复操作
```
## 重复操作
```
.           重复上一次操作，可以不断重复
```
## 移动光标
```
^           移动光标到行首
$           移动光标到行尾
G           移动到末行行首
w           移动到下一个单词首部，对于汉子，就是下一个汉子首部
e           移动到下一个单词尾部
```
## 插入
```
o           向下插入一行，并进入插入模式
O           向前插入一行，并进入插入模式
```
