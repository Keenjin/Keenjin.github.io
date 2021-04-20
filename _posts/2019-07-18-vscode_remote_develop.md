---
layout: post
title: 【调试技术】VSCode远程开发
date: 2019-07-18
tags: 调试技术
---

# VSCode远程开发

<!-- TOC -->

- [VSCode远程开发](#vscode远程开发)
    - [1. 开发环境](#1-开发环境)
    - [2. 基于launch.json的远程调试方法](#2-基于launchjson的远程调试方法)
        - [2.1 在目标机，安装delve调试器](#21-在目标机安装delve调试器)
        - [2.2 在目标机上，启动delve debug服务](#22-在目标机上启动delve-debug服务)
        - [2.3 在宿主机上，配置远程调试](#23-在宿主机上配置远程调试)
        - [2.4 选择Connect to server，启动调试](#24-选择connect-to-server启动调试)
        - [2.5 如何确保两边代码同步](#25-如何确保两边代码同步)
    - [3. 基于Remote-SSH插件的远程调试方法](#3-基于remote-ssh插件的远程调试方法)
        - [3.1 在宿主机上，安装OpenSSH](#31-在宿主机上安装openssh)
        - [3.2 目标机器，需要开启SSH服务](#32-目标机器需要开启ssh服务)
        - [3.3 在宿主机的VSCode中，安装并配置Remote - SSH插件](#33-在宿主机的vscode中安装并配置remote---ssh插件)
        - [3.4 SSH免密登录](#34-ssh免密登录)

<!-- /TOC -->

## 1. 开发环境

最近做后台需求比较多，使用到了CentOS Mini版本的系统。由于没有界面，开发需求都是在Windows通过VSCode开发完成，然后通过ssh传输源代码到CentOS系统上，进而进行日志级别调试。开发同学都知道，这种靠日志行为的调试，效率极低。微软也为此大费精力，为了提升大家开发效率，设计了多种远程调试方法。  

launch.json是传统的远程调试方法，通过更改配置文件，能把调试目标重定向到远端系统上。  
Remote-SSH插件（当然，还有Remote-Docker、Remote-WSL插件，这里我只涉及到最常规的Windows + 本地VMWare虚拟机内Linux系统的开发环境，所以使用Remote-SSH在我工作中最为常见），这个相比launch.json的方式，更为炫酷，也是微软最近才推出的一款官方插件（2019.5月）。[Remote - SSH](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-ssh)

- 宿主机：Windows 10 + VSCode + go环境  
- 目标机：VMWare + CentOS7.3 Mini + go环境，IP：192.168.2.132，源码路径：/opt/go/src/test_remote

源代码（test_remote\main.go）

```go
package main

import (
	"fmt"
	"strings"
)

func printsss(s string) {
	if strings.Contains(s, "/") {
		// 找到s的上一层
		pos := strings.LastIndex(s, "/")
		printsss(s[:pos])
	}

	fmt.Println(s)
}

func main() {
	a := "/abc/ddd/dede/"
	a = strings.Trim(a, "/")
	printsss(a)
}

```

## 2. 基于launch.json的远程调试方法

### 2.1 在目标机，安装delve调试器

```bash
go get -u github.com/derekparker/delve/cmd/dlv
```

### 2.2 在目标机上，启动delve debug服务

```bash
cd /opt/go/src/test_remote
dlv debug --headless --listen=:2345 --log --api-version=2
```

![jpg](/images/post/vscode/dlv_server.jpg)

### 2.3 在宿主机上，配置远程调试

选择：debug ——> Add Configuration ——> Go: Connect to server  

launch.json

```json
{
    // Use IntelliSense to learn about possible attributes.
    // Hover to view descriptions of existing attributes.
    // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Connect to server",
            "type": "go",
            "request": "attach",
            "mode": "remote",
            "remotePath": "/opt/go/src/test_remote",    // 设置虚拟机内的源码路径目录
            "port": 2345,
            "host": "192.168.2.132",
            "program": "${fileDirname}",    // 注意，这里设置的是当前本机的源码路径目录，与remotePath对应上，且两边的代码必须保持同步。虽然提示说不兼容这个设置项，但是一定要设置
        },
        {
            "name": "Launch",
            "type": "go",
            "request": "launch",
            "mode": "auto",
            "program": "${fileDirname}",
            "env": {},
            "args": []
        }
    ]
}
```

### 2.4 选择Connect to server，启动调试

选择Connect to server，然后F5，启动调试，这里宿主机上，已经可以看到调试运行起来了。

![jpg](/images/post/vscode/debug.jpg)

目标机上，调试也正在执行

![jpg](/images/post/vscode/debug_target.jpg)

### 2.5 如何确保两边代码同步

注意：由于这里的调试，非常依赖两边代码完全一样，所以代码同步，是比较常见的动作。常规的拷贝代码，会比较繁琐，这里可以采用自动代码同步方案  

VSCode中，搜索并安装[SFTP插件](https://github.com/liximomo/vscode-sftp)，使用步骤如下：

```bash
# ctrl+shift+p，然后输入SFTP：config
# 编辑如下：
{
    "name": "My Server",
    "host": "192.168.2.132",
    "protocol": "sftp",
    "port": 22,
    "username": "root",
    "remotePath": "/opt/go/src/test_remote",
    "uploadOnSave": true,
    "password": "xxxxxx"
}
# ctrl+shift+p，然后输入SFTP：Sync Local ——> Remote，可以把本机内容，同步到远端。相反，使用SFTP：Sync Remote ——> Local，可以把远端的内容，同步到本地
```

## 3. 基于Remote-SSH插件的远程调试方法

使用Remote-SSH插件，完全的远程开发概念，可以让代码跑在远端Linux，而实际调试则利用Windows开发机的VSCode的可视化调试方案。完全将代码和调试环境分离，无需保持本地和远端代码同步，因为代码无需放在本地，直接放在远端机器即可。基本原理，可用如下图表示：

![png](/images/post/vscode/architecture-ssh.png)

基本使用方法，参考<https://code.visualstudio.com/docs/remote/ssh>。这里以当前（Windows + VSCode） + （VMWare + CentOS）为例，记录使用步骤。

### 3.1 在宿主机上，安装OpenSSH

需要先在宿主机（Windows 10）上，安装[OpenSSH](https://docs.microsoft.com/zh-cn/windows-server/administration/openssh/openssh_install_firstuse)。这里，如果是win10以前的版本，需要安装[Git](https://git-scm.com/download/win)  

### 3.2 目标机器，需要开启SSH服务

CentOS默认支持SSH服务，这时需要开启SSHD

```bash
systemctl enable sshd.service
systemctl start sshd.service
```

### 3.3 在宿主机的VSCode中，安装并配置Remote - SSH插件

安装完Remote - SSH插件后，会在左边侧边栏，产生一个远程入口，点击配置，会进行选择一个.ssh的config，进入配置后如下：

![png](/images/post/vscode/Remote-SSH.png)

配置准备好后，就可以双击配置项，会新启动一个窗口，然后使用Open的时候，会从远端打开某个目录进行开发调试，跟在本地是一样的。打开一个/opt/go/src/test_remote目录后，如下：

![jpg](/images/post/vscode/remote_ssh_debug.jpg)

这个时候，运行F5，是找不到go调试环境的，因为远端还没有安装vscode-go的调试环境。

![jpg](/images/post/vscode/remote_vscode_go.jpg)

安装结束后，reload一下go插件，这样，就可以正常调试里，跟在本地调试代码完全一样的体验。查看此时的terminal，可以看到链接到远端去了。

![jpg](/images/post/vscode/remote_debugging.jpg)

### 3.4 SSH免密登录

生成密钥对，我这里选择在Win10机器上生成。

```bash
# 生成MyLinuxRsa私钥和MyLinuxRsa.pub公钥
ssh-keygen -t rsa -f c:\Users\keenjin\.ssh\MyLinuxRsa

# 将公钥，拷贝到服务器上
cd ~/.ssh/
ls
# 如果authorized_keys文件不存在，则创建
vim authorized_keys
chmod 600 authorized_keys

rz  # 在CentOS中执行rz命令，然后将c:\Users\keenjin\.ssh\MyLinuxRsa.pub文件，上传过去
cat MyLinuxRsa.pub >> authorized_keys

# 更改CentOS上的ssh的配置，确保支持密钥登录/etc/ssh/sshd_config
# 注意，以下几项必须要有
···sshd_config
PubkeyAuthentication yes
AuthorizedKeysFile  .ssh/authorized_keys
···

# 在Win10机器上，配置ssh的私钥文件
# 编辑c:\users\keenjin\.ssh\config，设置私钥文件
···config
# Read more about SSH config files: https://linux.die.net/man/5/ssh_config
Host Linux
    HostName 192.168.2.132
    User root
    IdentityFile C:\Users\keenjin\.ssh\MyLinuxRsa
···

# 远程连接选用c:\users\keenjin\.ssh\config配置登录
```