---
layout: post
title: 使用vscode做c++项目开发
date: 2019-10-10
tags: C++
---

# 准备环境

编辑器：vscode
构建工具：cmake
c++标准库：mac上的clang中的libc++

vscode插件：cmake、cmaketool
cmake：用于辅助构建，cmakelist用于构建方式和链接库等作用，类似于makefile，但是比makefile简单
cmaketool：用于界面命令（shift+command+p，然后cmake quick start），自动生成cmake工程所需文件（主要是cmakelist，以及build目录）

