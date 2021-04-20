---
layout: post
title: 【操作指令】CentOS下doc转docx
date: 2019-05-13
tags: 操作指令
---

# 参考

<https://stackoverflow.com/questions/52277264/convert-doc-to-docx-using-soffice-not-working>

# 安装

```shell
yum remove openoffice* libreoffice*
yum install libreoffice*
```

# 执行命令

```shell
soffice --headless --convert-to docx teste.doc
```

# python调用

```python
#!/usr/bin/env python
# coding:utf-8
import subprocess
output = subprocess.check_output(["soffice","--headless","--invisible","--convert-to","docx","/home/requiem/workspace/python3/test.doc","--outdir","/home/requiem/workspace/python3/"])
```
