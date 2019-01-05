---
layout: post
title: Python笔记
date: 2019-01-04
tags: 语言  
---

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