---
layout: post
title: Python笔记杂项
date: 2019-01-04
tags: python  
---

<!-- TOC -->

- [py资源](#py资源)
- [py符号](#py符号)
- [py脚本传参（类似exe传参）](#py脚本传参类似exe传参)
- [py读文件](#py读文件)
- [py写文件](#py写文件)
- [py函数加doc说明，以便help(xxxfunc)能展示出来。另外，演示另一个文件如何引用前一个文件写的函数](#py函数加doc说明以便helpxxxfunc能展示出来另外演示另一个文件如何引用前一个文件写的函数)
- [py中cls和self的区别](#py中cls和self的区别)
- [virtualenv创建python工程](#virtualenv创建python工程)
    - [Step1:安装virtualenv](#step1安装virtualenv)
    - [Step2:创建总工程根目录（根据特定python版本）](#step2创建总工程根目录根据特定python版本)
    - [Step3:激活Proj1](#step3激活proj1)
    - [Step4:安装nose（自动化单元测试）](#step4安装nose自动化单元测试)
    - [Step5:进入Proj1，创建一个主线版本trunk](#step5进入proj1创建一个主线版本trunk)
    - [Step6:创建项目框架](#step6创建项目框架)
    - [Step7:编辑setup.py](#step7编辑setuppy)
    - [Step8:编辑tests/MyModule_tests.py](#step8编辑testsmymodule_testspy)
    - [Step9:测试nose有效](#step9测试nose有效)
- [Python学习的未来方向](#python学习的未来方向)
- [python中logging模块的使用](#python中logging模块的使用)
- [python中ConfigParser模块的使用](#python中configparser模块的使用)
- [python中MySQLdb模块的使用](#python中mysqldb模块的使用)
- [python中os模块的使用](#python中os模块的使用)
- [python中hashlib模块的使用](#python中hashlib模块的使用)
- [python中json模块的使用](#python中json模块的使用)
- [python中xlsx相关模块的使用](#python中xlsx相关模块的使用)
- [python中threading模块的使用](#python中threading模块的使用)
- [Python中subprocess模块的使用](#python中subprocess模块的使用)
- [Python中urllib模块的使用](#python中urllib模块的使用)
- [python中大文件MD5](#python中大文件md5)
- [python文件上传](#python文件上传)
- [python中文件遍历及重命名](#python中文件遍历及重命名)
- [Python中select和epoll异步网络模型的使用](#python中select和epoll异步网络模型的使用)
- [Python中scapy模块的使用](#python中scapy模块的使用)
- [CentOS7.2中安装Python3.6.8](#centos72中安装python368)
- [Python解析xml](#python解析xml)

<!-- /TOC -->

# py资源

python流行库：[https://github.com/jobbole/awesome-python-cn/blob/master/README.md](https://github.com/jobbole/awesome-python-cn/blob/master/README.md)
官方模块查询（支持的python版本等）：[https://pypi.org/](https://pypi.org/)

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
> pip install virtualenv
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

# Python学习的未来方向

```txt
数据分析
自然语言处理
社交网络分析
人工智能
深度学习
计算机视觉
网络爬虫
量化交易
```

# python中logging模块的使用

```python
import logging

# 全局单实例，通过它来打日志
logger = logging.getLogger()

# 觉得日志文件是输出到控制台，还是文件
def add_log_handler(log_file_name=''):
    logger.setLevel(logging.DEBUG)

    log_handler = None
    if log_file_name != '':
        dirpath = os.path.dirname(log_file_name)
        if not os.path.exists(dirpath):
            os.mkdir(dirpath)

        log_handler = logging.FileHandler(log_file_name)
        log_handler.setLevel(logging.INFO)
    else:
        log_handler = logging.StreamHandler()
        log_handler.setLevel(logging.DEBUG)

    log_handler.setFormatter(logging.Formatter('%(asctime)-15s (%(filename)s)[%(levelname)s] %(message)s'))
    logger.addHandler(log_handler)

# 使用方式
def main():
    logger.error('errorxxxx')
    logger.info('info....')
    logger.warning('warning....')
```

# python中ConfigParser模块的使用

ConfigParser解析的配置文件，类似ini文件  

```ini
[db]
section1 = 10
section2 = testttttt

[cfg]
key1 = 3
key2 = test
```

```python
#!/usr/bin/env python2
# coding=utf-8
from ConfigParser import ConfigParser

Config = {}

def load_cfg(conf):
    global Config

    try:
        parser = ConfigParser()
        parser.read(conf)

        Config['section1'] = int(parser.get('db', 'section1'))
    except Exception as e:
        logger.error(str(e))
```

```python
#!/usr/bin/env python3
# coding=utf-8
import configParser

Config = {}

def load_cfg(conf):
    global Config

    try:
        parser = configParser.ConfigParser()
        parser.read(conf)

        Config['section1'] = int(parser.get('db', 'section1'))
    except Exception, e:
        logger.error(str(e))
```

# python中MySQLdb模块的使用

```python
#!/usr/bin/env python2
# coding=utf-8
import MySQLdb

def load_db():
    conn = MySQLdb.connect(host='xxxxxxx', port=xxxx, user='xxxx', passwd='xxxxxx', db='xxxx', charset='utf8')
    cursor = conn.cursor()
    try:
        cursor.execute('select xxx from xx where xxx')
        result = cursor.fetchall()
    except:
        logger.error('error db')
    conn.close()
    return result
```

# python中os模块的使用

```python
os.path.dirname('c:\\test.txt')     # c:\\
if not os.path.exists('c:\\test.txt'):
    pass
os.mkdir('c:\\test\\')
if not os.path.isfile('c:\\test.txt'):
    pass
os.remove('c:\\test.txt')
os.symlink('c:\\test1','d:\\test1')
os.system('python c:\\test.py')
os.path.join('c:\\test\\','file.txt')
```

# python中hashlib模块的使用

```python
import hashlib

a = "I am huoty"
print hashlib.md5(a).hexdigest()
print hashlib.sha1(a).hexdigest()
print hashlib.sha224(a).hexdigest()
print hashlib.sha256(a).hexdigest()
print hashlib.sha384(a).hexdigest()
print hashlib.sha512(a).hexdigest()
```

# python中json模块的使用

```python
import json

a = "{key1:1,key2:3}"
new_dict = json.loads(a)    # 将字符串转换为字典

file1.json:
{key1:1,key2:3}
通过读json文件
with open('file1.json','r') as f:
    new_dict = json.loads(f)

写一个json文件
with open('file1.json','w') as f:
    json.dump(new_dict, f)
```

# python中xlsx相关模块的使用

```python
from openpyxl.workbook import Workbook
from openpyxl.writer.excel import ExcelWriter

def main():
    excel_wb = Workbook()
    excel_ws = excel_wb.active

    excel_ws.cell(row=1, column=1).value = "head1"
    excel_ws.cell(row=1, column=2).value = "head2"
    excel_ws.cell(row=1, column=3).value = "head3"

    excel_ws.cell(row=2, column=1).value = "xxxx"
    excel_ws.cell(row=2, column=2).value = "xx"
    excel_ws.cell(row=2, column=3).value = "xxxxxx"

    excel_wb.save(filename='xxx.xlsx')
```

# python中threading模块的使用

```python
import threading

class WorkThread(threading.Thread):
    def __init__(self):
        threading.Thread.__init__(self)
    def run(self):
        while True:
            xxxx

def main():
    w = WorkThread()
    w.setDaemon(True)   # 当主线程结束时，子线程也跟着结束，无论是否执行完毕。另外，join则是等待子线程结束
    w.Start()
```

# Python中subprocess模块的使用

```python
import subprocess

class TcpDump():
    def __init__(self, program, name, num):
        self.program = program
        self.tcpdump_pcap = name
        self.tcpdump_limit = num
        self.p_tcpdump = None

    def start_tcpdump(self):
        cmd = [self.program, "-iany", "-n", "-s0", "-c%d" % (self.tcpdump_limit), "-w%s" % (self.tcpdump_pcap)]
        log = 'start tcpdump with cmd: %s' % (' '.join(cmd))
        logger.info(log)
        self.p_tcpdump = subprocess.Popen(cmd, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
        return self.p_tcpdump.pid

    def stop_tcpdump(self):
        self.p_tcpdump.send_signal(signal.SIGINT)
        (stdoutdata, stderrdata) = self.p_tcpdump.communicate()
        retcode = self.p_tcpdump.returncode
        logger.info('tcpdump exit with retcode:%d', retcode)
        return retcode

    def caculate_sha256(self):
        fd = open(self.tcpdump_pcap, 'rb')
        content = fd.read()
        p_sha256 = hashlib.sha256(content).hexdigest()
        return p_sha256

def main():
    pTcpDump = TcpDump("/usr/bin/tcpdump", "/home/testtcpdump.pcap", 5000)
    pTcpDump.start_tcpdump()
    while True:
        xxx
    pTcpDump.stop_tcpdump()
```

# Python中urllib模块的使用

```python
#!/usr/bin/env python2
#coding=utf-8
import urllib, urllib2

class FileUpload():
    def __init(self, host, port):

url = "http://xxxxxx?param"
req = urllib2.Request(url)
boundary = hashlib.md5('%s' % hex(int(time.time() * 1000))).hexdigest()
boundary = boundary.encode('base64').strip('=\r\n')[:16]
mpdata = '\r\n'.join([
        '--%s' % boundary,
        'Content-Disposition: form-data; name="file"; filename="hidden"',
        'Content-Type: application/octet-stream',
        '',
        data,
        '--%s--' % boundary,
        ''
        ])
req.add_header('Content-Type', 'multipart/form-data; boundary=%s' % boundary)
req.add_header('Content-length', str(len(mpdata)))
req.add_data(mpdata)
resp = urllib2.urlopen(req,timeout=5).read()
logger.info('Post data of channel "%s" to API with response of: %s', channel, resp)
```

# python中大文件MD5

```python
def big_file_md5(file):
    md5_value = hashlib.md5()
    with open(file, 'rb') as f:
        while True:
            data_flow = f.read(8096)
            if not data_flow:
                break
            md5_value.update(data_flow)
    return md5_value.hexdigest()
```

# python文件上传

client：

```python
#!/usr/bin/env python3
# coding=utf-8

import os
import logging
import urllib
import json
import codecs

logger = logging.getLogger(__name__)

class FileSend(object):
    def __init__(self, host, port):
        self.host = host
        self.port = port
        self.url = host + ":" + str(port)
    def post_file(self, channel, filename, encoding, offset):
        if not os.path.exists(filename):
            logger.warning("file:%s not exist.", filename)
            return None
        data = {}
        data["channel"] = channel
        with codecs.open(filename, 'r', encoding) as f:
            f.seek(offset, 0)
            data["data"] = f.read()
        req = urllib.request.Request(self.url)
        req.add_header('Content-Type', 'application/json')
        sendbytes = urllib.parse.urlencode(data).encode("utf-8")

        try:
            resq = urllib.request.urlopen(req, data = sendbytes, timeout = 5).read()
            logger.info("Post channel:%s File to Url:%s Respond:%s. filename:%s, encoding:%s, offset:%d", channel, self.url, resq, filename, encoding, offset)
            return resq
        except Exception as e:
            logger.warning("Post channel:%s File to Url:%s failed. err(%s), filename:%s, encoding:%s, offset:%d", channel, self.url, str(e), filename, encoding, offset)
            return None
```

server：

```python
#!/usr/bin/env python3
# coding=utf-8

from http.server import BaseHTTPRequestHandler, HTTPServer
import json
import logging

logger = logging.getLogger(__name__)

g_OnLogReceive = None

class RequestHandler(BaseHTTPRequestHandler):
    def _set_headers(self):
        self.send_response(200)
        self.send_header('Content-type', 'application/json')
        self.end_headers()

    def set_CallBack(self, OnFileReceive):
        self.OnFileReceive = OnFileReceive

    def do_GET(self):
        response = {
            'status':'SUCCESS',
            'data':'hello from server'
        }

        self._set_headers()
        self.wfile.write(json.dumps(response))

    def do_POST(self):
        content_length = int(self.headers['Content-Length'])
        post_data = self.rfile.read(content_length).decode('utf-8')
        logger.debug('获取到的数据：%s', post_data)
        if g_OnLogReceive != None:
            g_OnLogReceive(post_data)

        response = {
            'status':'SUCCESS',
            'data':'server got your post data'
        }
        self._set_headers()
        self.wfile.write(json.dumps(response).encode("utf-8"))

def run(OnLogReceive, port):
    global g_OnLogReceive
    g_OnLogReceive = OnLogReceive
    print('Listening on port:%s' % port)
    server = HTTPServer(('', port), RequestHandler)
    server.serve_forever()

```

# python中文件遍历及重命名

```python
#!/usr/bin/env python3
#coding=utf-8

'''
注意，调试的话，vscode需要参数，可以直接在.vscode所在的launch.json配置中，配置一下参数
"configurations": [
        {
            "name": "Python: Current File (Integrated Terminal)",
            "type": "python",
            "request": "launch",
            "program": "${file}",
            "console": "integratedTerminal",
            "args":[
                "C:\\Users\\xxxxx\\Desktop\\sssss",
                "keen",
                "lml"
            ]
        }
'''

import os
from sys import argv

def main(argv):
    path = argv[1]
    if not os.path.exists(path):
        return
    if len(argv) < 4:
        return
    if argv[2] == None or argv[2] == "":
        return
    if argv[3] == None or argv[3] == "":
        return
    for dirpath,dirnames,filenames in os.walk(path):
        for file in filenames:
            if file[0:len(argv[2])] == argv[2]:
                src = os.path.join(dirpath, file)
                dst = os.path.join(dirpath, argv[3] + file[len(argv[2]):])
                print("src:",src,"dst:",dst)
                os.rename(src,dst)

# rename_filexxx.py c:\test\ keen lml
if __name__ == "__main__":
    main(argv)

```

# Python中select和epoll异步网络模型的使用

<https://www.cnblogs.com/JohnABC/p/6076006.html>

# Python中scapy模块的使用

scapy是一个万能的网络工具  
安装命令：pip install scapy --proxy=http://web-proxy.oa.com:8080  

# CentOS7.2中安装Python3.6.8

<https://blog.csdn.net/lovefengruoqing/article/details/79284573>

# Python解析xml

```python
#!python3

import os
from sys import argv
import sys
import platform
import logging
import xml.etree.ElementTree as ET
import json
import time

logger = logging.getLogger()


def InitLogger(log_file_name=''):
    logger.setLevel(logging.DEBUG)

    log_handler = None
    if log_file_name != '':
        dirpath = os.path.dirname(log_file_name)
        if not os.path.exists(dirpath):
            os.mkdir(dirpath)
        log_handler = logging.FileHandler(log_file_name)
    log_handler.setFormatter(logging.Formatter('%(asctime)-15s (%(filename)s)[%(levelname)s] %(message)s'))
    logger.addHandler(log_handler)
    log_handler = logging.StreamHandler()
    log_handler.setFormatter(logging.Formatter('%(asctime)-15s (%(filename)s)[%(levelname)s] %(message)s'))
    logger.addHandler(log_handler)
    logger.info('logfile is %s', log_file_name)


def ParseVCProj(proj):
    logger.info('parse vcproj %s', proj)
    xmlParser = ET.XMLParser(encoding='utf-8')
    tree = ET.parse(proj, parser=xmlParser)
    root = tree.getroot()
    for cfg in root.iterfind('Configurations/Configuration'):
        if cfg.get('Name') == 'Release|Win32':
            for tool in cfg.iterfind('Tool'):
                if tool.get('Name') == 'VCCLCompilerTool':
                    clInfo = {'Project': proj, 'VCCLCompilerTool': tool.attrib}
                    return clInfo


def RemoveCodeAnalysis(proj):
    logger.info('parse vcproj %s', proj)
    shutil.copyfile(proj, proj+'.bak')
    xmlParser = ET.XMLParser(encoding='utf-8')
    tree = ET.parse(proj, parser=xmlParser)
    root = tree.getroot()
    for cfg in root.iterfind('Configurations/Configuration'):
        if cfg.get('Name') == 'Release|Win32':
            for tool in cfg.iterfind('Tool'):
                if tool.get('Name') == 'VCCLCompilerTool':
                    preprocessor = tool.get('PreprocessorDefinitions')
                    newPreprocessor = ''
                    pos = preprocessor.find('CODE_ANALYSIS')
                    if pos != -1 :
                        pl = preprocessor.split(';')
                        for item in pl:
                            if item != 'CODE_ANALYSIS':
                                newPreprocessor = newPreprocessor + item + ';'
                    if newPreprocessor.endswith(';'):
                        newPreprocessor = newPreprocessor[0:-1]
                    tool.set('PreprocessorDefinitions', newPreprocessor)
                    tree.write(proj, 'gb2312', True)
                    return


def Run():
    # get dir
    path = ''
    if len(argv) > 1:
        path = argv[1]
    else:
        path = os.path.dirname(script)
    logger.info('path: %s', path)

    # iterate dir
    projFiles = []
    for dirpath, dirnames, filenames in os.walk(path):
        # do not search like .vscode path
        if os.path.basename(dirpath).startswith('.'):
            continue
        for f in filenames:
            if f.endswith('.vcproj'):
                projFiles.append(os.path.join(dirpath, f))

    # parse
    clInfos = []
    for proj in projFiles:
        clInfo = ParseVCProj(proj)
        clInfos.append(clInfo)

    # output
    outFile = os.path.join(os.path.dirname(script), os.path.basename(script).split('.')[0] + '.json')
    with open(outFile, 'w') as f:
        f.writelines(json.dumps(clInfos, indent=4, separators=(', ', ': ')))
        logger.info('output: %s', outFile)


if __name__ == "__main__":
    print(sys.version)
    pythonV = platform.python_version()
    if pythonV[0] != '3':
        print('error, please use python3 !!!!!!')
        time.sleep(3)
        exit(-1)

    # print log
    script = argv[0]
    InitLogger(os.path.join(os.path.dirname(script), os.path.basename(script).split('.')[0] + '.log'))

    try:
        Run()
    except Exception as e:
        logger.error(str(e))

    logger.info('success finish !!!!!!')
    time.sleep(3)
    exit(0)

```