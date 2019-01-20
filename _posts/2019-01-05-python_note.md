---
layout: post
title: Python笔记杂项
date: 2019-01-04
tags: python  
---

# py资源

python流行库：https://github.com/jobbole/awesome-python-cn/blob/master/README.md
官方模块查询（支持的python版本等）：https://pypi.org/

# py符号

![关键字](https://s2.ax1x.com/2019/01/05/F7M4G4.jpg)  

![数据类型](https://s2.ax1x.com/2019/01/05/F7M7s1.jpg)  

![操作符](https://s2.ax1x.com/2019/01/05/F7MLdK.jpg)  

![字符串转义符](https://s2.ax1x.com/2019/01/05/F7MqZ6.jpg)  

![字符串格式化](https://s2.ax1x.com/2019/01/05/F7MHqx.jpg)  

# py脚本传参（类似exe传参）

```python
#!/bin/python
from sys import argv

script, first, second, third = argv

print("The script is called:", script)
print("Your first variable is:", first)
print("Your second variable is:", second)
print("Your third variable is:", third)

```

# py读文件

```python
#!/bin/python
from sys import argv

script, filename = argv

txt = open(filename)

print(f"Here's your file {filename}:")
method = input("""
choose one method:
1.read
2.readline
3.readlines
""")

if method == "1":
    print(f"{txt.read()}")          # 读取整个文本，适合文件比较小的场景
elif method == "2":
    print(f"{txt.readline()}")      # 读取一行，内部ptr会自动定位到下一行
    print(f"{txt.readline()}")
    print(f"{txt.readline()}")
elif method == "3":
    print(f"{txt.readlines()}")     # 按照列表格式，把所有的按行存储成字符串数组['This is stuff I typed into a file.\n', 'It is really cool stuff.\n', 'Lots and lots of fun to have in here.\n']

```

# py写文件

```python
#!/bin/python
from sys import argv

script, filename = argv

print(f"We're going to erase {filename}")
print("If you don't want that, hit CTRL-C (^C).")
print("If you do want that, hit RETRUN.")

input("?")

print("Opening the file...")
target = open(filename, 'w')        # 'w' for "write"; 'r' for "read"; 'a' for "append"。默认是'r'

print("Truncating the file. Goodbye!")
target.truncate()       # 将文件清空，从头开始写（如果是'w'，这里是没有必要的，因为一定会把全部文档刷没了的）

print("Now I'm going to ask you for three lines.")

line1 = input("line 1: ")
line2 = input("line 2: ")
line3 = input("line 3: ")

print("I'm going to write these to the file.")

target.write(line1)
target.write("\n")
target.write(line2)
target.write("\n")
target.write(line3)
target.write("\n")

print("And finally, we close it.")
target.close()
```

# py函数加doc说明，以便help(xxxfunc)能展示出来。另外，演示另一个文件如何引用前一个文件写的函数

```python
#!/bin/python
# five.py
def break_words(stuff):
    """This function will break up words for us.""" # help(five.break_words)的时候会展示出来
    words = stuff.split(' ')    # 按照空格拆分语句
    return words
```

```python
import five.py
sentence = "All is right."
words = five.break_words(sentence)
help(five)
help(five.break_words)
```

# py中cls和self的区别

例子：

```python
class A(object):
    a = 'a'
    @staticmethod
    def foo1(name):
        print('hello', name)
        print(A.a) # 正常
        print(A.foo2('mamq')) # 报错: unbound method foo2() must be called with A instance as first argument (got str instance instead)
    def foo2(self, name):
        print('hello', name)
    @classmethod
    def foo3(cls, name):
        print('hello', name)
        print(A.a)
        print(cls().foo2(name))
```

cls用在classmethod方法中，内部可以调用静态方法（此时跟staticmethod方法一样），也可以调用非静态方法。它内部会生成临时对象，来调用非静态方法，所以可以直接用类调用：A.foo3('testname')
self只用在对象调用中：A a; a.foo2('testname')

# virtualenv创建python工程

## Step1:安装virtualenv

```bash
> pip install virtual
```

## Step2:创建总工程根目录（根据特定python版本）

```bash
> mkdir KeenProj
> virtualenv --system-site-packages KeenProj/Proj1
```

## Step3:激活Proj1

```bash
> .\KeenProj\Proj1\Scripts\activate.bat
```

## Step4:安装nose（自动化单元测试）

```bash
> pip install nose
```

## Step5:进入Proj1，创建一个主线版本trunk

```bash
> cd KeenProj/Proj1
> mkdir Proj1_trunk
> cd Proj1_trunk
```

## Step6:创建项目框架

```bash
> mkdir bin
> mkdir MyModule
> mkdir tests
> mkdir docs
> touch MyModule/__init__.py
> touch tests/__init__.py
> touch setup.py
> touch tests/MyModule_tests.py
> tree /f
```

## Step7:编辑setup.py

```python
# 参考：https://docs.python.org/3/distutils/setupscript.html
try:
    from setuptools import setup
except ImportError:
    from distutils.core import setup

config = {
    'description': 'My Project',
    'author': 'My Name',
    'url': 'URL to get it at.',
    'download_url': 'Where to download it.',
    'author_email': 'My email.',
    'version': '0.1',
    'install_requires': ['nose'],
    'packages': ['NAME'],
    'scripts': [],
    'name': 'projectname'
}

setup(**config)
```

## Step8:编辑tests/MyModule_tests.py

```python
from nose.tools import *
import MyModule

def setup():
    print("SETUP!")

def teardown():
    print("TEAR DOWN!")

def test_basic():
    print("I RAN!")
```

## Step9:测试nose有效

```bash
> nosetests
```

