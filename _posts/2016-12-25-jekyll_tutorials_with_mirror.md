---
layout: post
title: Jekyll搭建个人博客
date: 2018-12-25 
tags: 经验  
---

# 1 概述

> 本博客基于[Jekyll搭建个人博客](http://baixin.io/2016/10/jekyll_tutorials1/)作为模板创建而成，非常感谢作者！
    
   
# 2 准备工作
jekyll需要依赖ruby、msys、rubygem、bundle。基本安装步骤大致如下：  
* 安装Ruby（最新的集成了gem、bundle）   
* 安装msys  
* 在msys里面使用如下命令更新相关数据库（相关的依赖的东西，必须要执行这步骤，<span style="color:red;">不出意外基本上会失败</span>）：  
```
pacman -Syu  
```

* 运行如下命令安装jekyll（<span style="color:red;">不出意外基本上会失败</span>）  
```
gem install jekyll  
```

# 3 异常处理
这里软件的兼容性还是做得比较好的，唯独是因为有各种墙，数据无法获取。  
## 3.1 配置gem源  
要使gem能正常工作，需要配置gem的源（原始的<https://rubygems.org/>不能用）  
对于国内常规用户，将源指向<https://gems.ruby-china.com/>即可，使用如下命令行：  
```
gem sources --add https://gems.ruby-china.com/ --remove https://rubygems.org/  
gem sources -l  
```

对于公司用户，受公司访问限制影响，如果要使用比较麻烦，需要找对应公司是否有提供镜像源链接 http://xxxx.xx.xxx，替换上述的https://gems.ruby-china.com，即可

## 3.2 配置bundle源  
使用如下命令，配置bundle源  
```
bundle config mirror.https://rubygems.org https://gems.ruby-china.com
```

对于公司受限用户，如果能找到对应镜像源链接，也参照gem的配置即可

## 3.3 配置msys源  
msys的镜像源配置，不是通过命令进行的，需要了解几个关键文件。  
* 文件1：C:\msys64\home\keenjin\\.bashrc（配置代理）  
```
export ALL_PROXY=http://web-proxy.xxxxxx.com:8080  
export HTTP_PROXY=http://web-proxy.xxxxxx.com:8080  
export HTTPS_PROXY=https://web-proxy.xxxxxx.com:8080  export http_proxy=$HTTP_PROXY  
export https_proxy=$HTTPS_PROXY  
export ftp_proxy=$http_proxy  
export rsync_proxy=$http_proxy  
export no_proxy=localhost,.xxx.com
```
* 文件2：C:\msys64\etc\pacman.d\mirrorlist.msys（配置源）  
```
Server = http://mirror-xxx.xxx.com/msys2/msys/$arch/  
Server = http://repo.msys2.org/msys/$arch/  
Server = https://sourceforge.net/projects/msys2/files/REPOS/MSYS2/$arch/  
Server = http://www2.futureware.at/~nickoe/msys2-mirror/msys/$arch/  
Server = https://mirror.yandex.ru/mirrors/msys2/msys/$arch/
```
* 文件3：C:\msys64\etc\pacman.d\mirrorlist.mingw32（配置源）  
```
Server = http://mirror-xxx.xxx.com/msys2/mingw/i686/  
Server = http://repo.msys2.org/mingw/i686/  
Server = https://sourceforge.net/projects/msys2/files/REPOS/MINGW/i686/  
Server = http://www2.futureware.at/~nickoe/msys2-mirror/mingw/i686/  
Server = https://mirror.yandex.ru/mirrors/msys2/mingw/i686/
```

* 文件4： C:\msys64\etc\pacman.d\mirrorlist.mingw64（配置源）  
```
Server = http://mirror-xxx.xxx.com/msys2/mingw/x86_64/  
Server = http://repo.msys2.org/mingw/x86_64/  
Server = https://sourceforge.net/projects/msys2/files/REPOS/MINGW/x86_64/  
Server = http://www2.futureware.at/~nickoe/msys2-mirror/mingw/x86_64/  
Server = https://mirror.yandex.ru/mirrors/msys2/mingw/x86_64/  
```

## 3.4 生成blog以后，jekyll serve异常  
这里的异常，往往是因为gemfile和gemfile.lock跟当前环境不匹配导致的，某些版本号不一样了。这里需要重新运行如下bundle install，安装各种依赖的库（由于前面设置了bundle的安装源，所以这里就不会有问题了，要是没有可访问的镜像源，可能还是异常的）  

# 4 总结  
> 凡是无法执行成功，都可能是安装源失败。linux有个比较好的点，所有的依赖，都在安装的时候自动去下载安装  

# 5 评论系统
使用的评论系统，是github官方的gitment，具体过程，是参考[Jekyll博客添加Gitment评论系统](https://blog.csdn.net/zhangquan2015/article/details/80178794)