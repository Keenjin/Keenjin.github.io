---
layout: post
title: Windows问题Debug
date: 2019-01-07
tags: windows
---


# 符号表

最近微软貌似说将符号表服务器迁移到了Azure国际版，国内被墙了，符号下载不了，找了一番，在看雪论坛找到了这个替代品：http://sym.ax2401.com:9999/symbols/，感谢大神分享！！！！！！

# 问题1：卸载Visual Studio 2015卡死

最近安装了vs2019，对我这老机器有限的磁盘空间而言，vs2015占据7.5G，为此一直耿耿于怀，而vs2019本身对2015~2019编译器有插件集成，工程编译没有问题，我就不需要2015额外的空间占用了。都知道，对于vs2015，因为安装后，默认安装了一堆Tools、.Net版本等，想要尽可能完全的卸载所有相关的依赖组件，就最好使用vs2015的vs_community.exe集成的卸载器功能（vs2019比较好的一点，是增加了一个visual studio installer工具，将组件安装卸载都集成在此，并随安装包一起安装在本地，无需单独去寻找离线包），为此找到vs2015的离线包iso，加载到虚拟光驱，运行，开始了卸载历程。  

然而，没过多久，就一直卡在这个进度：

![PNG](/images/post/windbg/vs2015_uninst_hung.png)

很明显，卸载进程出现卡死。  

## 首先找到是哪个卸载进程卡死

这里使用工具ProcessExp，首先找到当下这个卸载界面是哪个进程，查找到如下：

![PNG](/images/post/windbg/vs2015_uninst_hung1.png)

需要打开ProcessExp的进程树，因为我们得找到具体是哪个进程卡死，可能是其中的子进程。通过进程树信息可以看见，实际卸载过程，是由众多进程参与的，我们首先查看最下面的子进程，因为按常规来看，是最终拉起的进程干活。抓minidump先分析一下，打开windbg调试器，设置好符号表，打开dmp文件，开始分析。  

首先，查看所有线程的栈信息，其他线程栈看起来都比较正常，最有怀疑的是0号线程

```bash
0:000> ~* kv

.  0  Id: 1944.2ac8 Suspend: 0 Teb: 00c06000 Unfrozen
 # ChildEBP RetAddr  Args to Child              
00 00f1f104 77c9e78a 759f231a 0000024c 00000000 ntdll!KiFastSystemCallRet (FPO: [0,0,0])
01 00f1f108 759f231a 0000024c 00000000 00000000 ntdll!NtReadFile+0xa (FPO: [9,0,0])
DBGHELP: e:/symbols*http://sym.ax2401.com:9999/symbols/ is not a valid store
*** ERROR: Module load completed but symbols could not be loaded for vssdk_full.exe
02 00f1f16c 0054306c 0000024c 00f1f194 00000008 KERNELBASE!ReadFile+0x7a (FPO: [Non-Fpo])
WARNING: Stack unwind information not available. Following frames may be wrong.
03 00f1f1a4 005434f0 0000024c 00f1f200 00000000 vssdk_full+0x306c
04 00f1f1d8 00543b0c 0000024c 0054ebde 00f1f230 vssdk_full+0x34f0
05 00f1f200 0054fe90 0000024c 0000000c 0b278e50 vssdk_full+0x3b0c
DBGHELP: e:/symbols*http://sym.ax2401.com:9999/symbols/ is not a valid store
06 00f1f244 0056311c 0000024c 00000000 0b24db48 vssdk_full+0xfe90
07 00f1f27c 00563c71 00000000 00f1f2ac 00f1f35c vssdk_full+0x2311c
08 00f1f2b4 00564097 00f1f3e4 00000000 00f1f2fc vssdk_full+0x23c71
09 00f1f30c 0054c713 00000000 00000000 000000f0 vssdk_full+0x24097
0a 00f1f36c 00541303 00000000 00000000 0054180f vssdk_full+0xc713
DBGHELP: e:/symbols*http://sym.ax2401.com:9999/symbols/ is not a valid store
0b 00f1f3b0 00541a67 00f1f3d4 00000000 00000000 vssdk_full+0x1303
0c 00f1f3cc 00541e34 00000000 00568732 00000000 vssdk_full+0x1a67
DBGHELP: e:/symbols*http://sym.ax2401.com:9999/symbols/ is not a valid store
0d 00f1fa0c 00541028 00540000 010b3310 0000000a vssdk_full+0x1e34
0e 00f1fa28 005686df 00540000 00000000 010b3310 vssdk_full+0x1028
0f 00f1fab8 77152369 00c05000 77152350 00f1fb24 vssdk_full+0x286df
10 00f1fac8 77c6e5bb 00c05000 20921238 00000000 kernel32!BaseThreadInitThunk+0x19 (FPO: [Non-Fpo])
11 00f1fb24 77c6e58f ffffffff 77cb3e63 00000000 ntdll!__RtlUserThreadStart+0x2b (FPO: [Non-Fpo])
12 00f1fb34 00000000 00568732 00c05000 00000000 ntdll!_RtlUserThreadStart+0x1b (FPO: [Non-Fpo])

```

从栈信息可以看到：线程正在等待ReadFile返回结果  
ReadFile卡死，就可以YY出很多可能性，是读某个过大文件？文件系统异常？
不确定ReadFile的对象，就无法明确它出现的问题。ReadFile第一个参数，是读取对象的HANDLE，如果是文件，用windbg是可以找到文件或注册表路径的，也可以分析出handle的具体类型。  

![png](/images/post/windbg/vs2015_uninst_hung2.png)

可以看出，这个ReadFile是在等某个管道的输入，那么猜测只能是跟父进程在通信等待某个结果完成，一直等不到就一直卡死。因此，还需要继续看一下父进程信息，继续用ProcessExp抓取父进程winsdk_full.exe的minidump。  

![png](/images/post/windbg/vs2015_uninst_hung3.png)

其他线程看起来比较正常，就0号线程，发现有一个DialogBox？很奇怪，这种模式对话框不应该show出windows出来么？这直接卡死在这个线程出不去了，也就是，卡死肯定是它造成的。  

问题比较容易发现，怎么解决呢？有很多种方式，比如找到这个窗口句柄，然后Post一个关闭消息，将窗口关闭掉，或者尝试修改窗口隐藏属性，将窗口显示出来。  

## 找到是哪个窗口

通过windbg分析栈去查找窗口比较麻烦，我们知道，DialogBox一定包含对话框窗口类属性，同时，通过上述栈信息，我们知道，0号线程的pid是2a2c，tid是1d9c，我们可以通过spy++去抓取进程的线程相关的所有窗口，就可以将窗口缩小到最小范围。  

![png](/images/post/windbg/vs2015_uninst_hung4.png)

这里边，比较容易找到的是窗口句柄为0270d14，它包含对话框属性。剩下比较意外的收获是可以看见一个Static控件上，写有一个相当熟悉的文案：查找某个msi失败了。通过历史经验，我们知道，这里一定是卸载时某些组件升级和版本不兼容导致卸载组件异常，而卡死流程，而vs出了这么个bug，无法展示这个对话框给我们。  
为了进一步确认窗口句柄为0270d14就是我们最终找到的窗口，Static控件是它的子窗口，spy++转到窗口层级就可以很明显了。选中窗口，点击右键菜单-属性，点击同步，即可转到窗口树形层级结构图上去看。  

![png](/images/post/windbg/vs2015_uninst_hung5.png)

## 关闭对话框窗口

关闭窗口，可以发送WM_CLOSE，但是我们发现还有一个OK控件和一个Cancel控件，对于这类错误弹框，如果简单的close主窗口，会重复弹出。根据文案描述，这里我们需要去点击Cancel取消这个流程。我们可以写代码去模拟点击，这里我比较懒，想到另一个工具AccExplorer.exe，可以直接模拟点击（因为控件是窗口，所以这个工具可以正常工作，对于自绘窗口控件是不行的，这个工具有这个局限性）。  
选择Option-->Choose Window List，选择目标窗口（这里前面已经找到目标窗口的标题属性，如果有重复，可以一个个去确认尝试）

![png](/images/post/windbg/vs2015_uninst_hung6.png)
![png](/images/post/windbg/vs2015_uninst_hung7.png)

找到了Cancel控件，直接模拟点击即可。至此，流程发现已经往下继续走了。后续发现又有重复类型的问题，可以简单通过spy++和accexplorer来解决卡死问题。

## 解决卡死问题

前面的方法，可以找到为何卡死，以及如何解决卡死的一种方法。实际上，当分析到进程树时，当我们观察卸载过程，就会发现，它卸载也是调用不同进程。前面的方法，即使解决了弹框卡死，卸载流程实际是失败的，那么我们让当前插件的卸载进程继续执行已经没有意义了，此时，解决卡死问题最简单的方式，就是直接杀掉卡死组件的进程。