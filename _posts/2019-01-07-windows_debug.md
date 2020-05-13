---
layout: post
title: Windows问题Debug
date: 2019-01-07
tags: windows
---

<!-- TOC -->

- [1. 符号表](#1-符号表)
- [2. Windbg常用命令](#2-windbg常用命令)
    - [2.1. 应用层调试命令](#21-应用层调试命令)
    - [2.2. 内核层调试命令](#22-内核层调试命令)
- [3. 问题1：卸载Visual Studio 2015卡死](#3-问题1卸载visual-studio-2015卡死)
    - [3.1. 首先找到是哪个卸载进程卡死](#31-首先找到是哪个卸载进程卡死)
    - [3.2. 找到是哪个窗口](#32-找到是哪个窗口)
    - [3.3. 关闭对话框窗口](#33-关闭对话框窗口)
    - [3.4. 解决visual studio 2015卸载问题](#34-解决visual-studio-2015卸载问题)
- [4. 问题2：定位测试程序卡死](#4-问题2定位测试程序卡死)

<!-- /TOC -->


# 1. 符号表

最近微软貌似说将符号表服务器迁移到了Azure国际版，国内被墙了，符号下载不了，找了一番，在看雪论坛找到了这个替代品：http://sym.ax2401.com:9999/symbols/，感谢大神分享！！！！！！

# 2. Windbg常用命令

## 2.1. 应用层调试命令

```bash
# 查看模块列表，及模块信息，一般用来看pdb加载情况
lm
lmvm xxx.dll

# 强制加载不匹配符号。对于外发包经常由于时间戳或者签名不一致，虽然代码一致，但是也无法正常加载pdb，可以使用此方法加载
.reload /f /i xxxx.dll

# 查看符号，一般用来辅助bp下断点
x qtcore!*test*

# 下断点，bp对函数下断点，bl查看断点列表及id，bc则通过bl查看到的id，进行删除断点
bp qtcore!xxxxxx
bl
bc 1

# 查看调用栈
kv

# 查看所有线程调用栈
~* kv

# 查看局部变量及地址
dv /v

# 查看某个句柄信息
!handle 0x002345 f

# 查看异常，其中，0x002345表示ExceptionRecord地址，0x002349表示ContextRecord地址
.exr 0x002345
.cxr 0x002349

# 查看结构体（结构体名一般可以使用Local视图查看），可以用来看某个成员的偏移
dt 0x26665440 xxxstruct

# 计算表达式，也可以用来转换整数0nxxx到十六进制
? 0x26665440 +0x280

# 编辑内存，eb按照一个字节编辑、ew按照两个字节编辑、ed按照4字节编辑、ef和eD分别编辑浮点型、eza编辑null结尾的ascii字符串、ezu编辑null结尾unicode字符串、ea和eu则分别编辑非null结尾字符串
eb 0x26665448 1

# 条件断点，查看loadlibrary加载某个dll时中断（createfile等其他api类似）
ad dllname
bp Kernelbase!loadlibraryexw "as /mu ${/v:dllname} poi(@esp+4); .block{.if($spat(@\"${dllname}\",@\"*test.dll\")) {.echo ${dllname};kb;} .else {gc;}}"
```

## 2.2. 内核层调试命令

```bash
# 查看进程列表
!process 0 0

# 根据进程地址，查看调用栈
!process 0x002345 7

# 一键查看所有进程的所有线程调用栈（内核线程上千条，不太建议）
!process 0 7
```

# 3. 问题1：卸载Visual Studio 2015卡死

最近安装了vs2019，对我这老机器有限的磁盘空间而言，vs2015占据7.5G，为此一直耿耿于怀，而vs2019本身对2015~2019编译器有插件集成，工程编译没有问题，我就不需要2015额外的空间占用了。都知道，对于vs2015，因为安装后，默认安装了一堆Tools、.Net版本等，想要尽可能完全的卸载所有相关的依赖组件，就最好使用vs2015的vs_community.exe集成的卸载器功能（vs2019比较好的一点，是增加了一个visual studio installer工具，将组件安装卸载都集成在此，并随安装包一起安装在本地，无需单独去寻找离线包），为此找到vs2015的离线包iso，加载到虚拟光驱，运行，开始了卸载历程。  

然而，没过多久，就一直卡在这个进度：

![PNG](/images/post/windbg/vs2015_uninst_hung.png)

很明显，卸载进程出现卡死。  

## 3.1. 首先找到是哪个卸载进程卡死

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

## 3.2. 找到是哪个窗口

通过windbg分析栈去查找窗口比较麻烦，我们知道，DialogBox一定包含对话框窗口类属性，同时，通过上述栈信息，我们知道，0号线程的pid是2a2c，tid是1d9c，我们可以通过spy++去抓取进程的线程相关的所有窗口，就可以将窗口缩小到最小范围。  

![png](/images/post/windbg/vs2015_uninst_hung4.png)

这里边，比较容易找到的是窗口句柄为0270d14，它包含对话框属性。剩下比较意外的收获是可以看见一个Static控件上，写有一个相当熟悉的文案：查找某个msi失败了。通过历史经验，我们知道，这里一定是卸载时某些组件升级和版本不兼容导致卸载组件异常，而卡死流程，而vs出了这么个bug，无法展示这个对话框给我们。  
为了进一步确认窗口句柄为0270d14就是我们最终找到的窗口，Static控件是它的子窗口，spy++转到窗口层级就可以很明显了。选中窗口，点击右键菜单-属性，点击同步，即可转到窗口树形层级结构图上去看。  

![png](/images/post/windbg/vs2015_uninst_hung5.png)

## 3.3. 关闭对话框窗口

关闭窗口，可以发送WM_CLOSE，但是我们发现还有一个OK控件和一个Cancel控件，对于这类错误弹框，如果简单的close主窗口，会重复弹出。根据文案描述，这里我们需要去点击Cancel取消这个流程。我们可以写代码去模拟点击，这里我比较懒，想到另一个工具AccExplorer.exe，可以直接模拟点击（因为控件是窗口，所以这个工具可以正常工作，对于自绘窗口控件是不行的，这个工具有这个局限性）。  
选择Option-->Choose Window List，选择目标窗口（这里前面已经找到目标窗口的标题属性，如果有重复，可以一个个去确认尝试）

![png](/images/post/windbg/vs2015_uninst_hung6.png)
![png](/images/post/windbg/vs2015_uninst_hung7.png)

找到了Cancel控件，直接模拟点击即可。至此，流程发现已经往下继续走了。后续发现又有重复类型的问题，可以简单通过spy++和accexplorer来解决卡死问题。

## 3.4. 解决visual studio 2015卸载问题

前面的方法，可以找到为何卡死，以及如何解决卡死的一种方法。实际上，当分析到进程树时，当我们观察卸载过程，就会发现，它卸载也是调用不同进程。前面的方法，即使解决了弹框卡死，卸载流程实际是失败的，那么我们让当前插件的卸载进程继续执行已经没有意义了，此时，解决卡死问题最简单的方式，就是直接杀掉卡死组件的卸载进程，只让部分组件无法卸载。  

然而不幸的是，我机器上C:\ProgramData\PackageCache中的文件被某个不良工具清除掉了，所以导致大面积卸载异常，最后卸载失败。无奈尝试使用了微软官方的一系列卸载方法：

- 方法一：[常规强制卸载](https://visualstudio.microsoft.com/zh-hans/vs/support/vs2015/uninstall-visual-studio-2015/)。示例中使用的是企业版，我这里用的是社区版，需要使用vs_community.exe替代vs_enterprise.exe。
- 方法二：[不使用PackageCache](https://devblogs.microsoft.com/setup/moving-or-disabling-the-package-cache-for-visual-studio-2017/)。
- 方法三：[疑难杂症卸载](https://support.microsoft.com/en-us/help/17588/windows-fix-problems-that-block-programs-being-installed-or-removed)。

对于方法一和方法二，一般可以优先尝试，因为方法三非常耗时。然而，由于我的机器大面积异常，尝试前两种方法对我无效。方法三的基本使用方式如下：

- Step1：控制面板执行visual studio 2015卸载，等待卡死
- Step2：分析卡死的窗口，spy++查看对应窗口的msi组件文案，确认是哪个msi
- Step3：使用下载下来的工具MicrosoftProgram_Install_and_Uninstall.meta.diagcab，选择卸载，然后从Step2我们知道具体是哪个msi，从扫描到的列表中，选择该卸载异常的msi，执行卸载
- Step4：重复执行Step1，等待卡死，直到无任何卡死

# 4. 问题2：定位测试程序卡死

测试程序D3D12Test.exe，启动后卡死。使用Windbg Attach上去调试：

```bash
0:009> ~* kv

   0  Id: 543c.3968 Suspend: 1 Teb: 00219000 Unfrozen
 # ChildEBP RetAddr  Args to Child              
00 0055406c 770692be 005540e4 76f49482 00081cd8 win32u!NtUserDispatchMessage+0xc (FPO: [1,0,0])
01 005540c0 77068cf0 76a1d442 00554110 77086486 USER32!DispatchMessageWorker+0x5be (FPO: [Non-Fpo])
02 005540cc 77086486 005540e4 00081cd8 00000001 USER32!DispatchMessageW+0x10 (FPO: [Non-Fpo])
03 00554110 770869ea 00000000 00000001 00000000 USER32!DialogBox2+0x170 (FPO: [Non-Fpo])
04 00554140 770c089b 00081cd8 770a1bf0 00554390 USER32!InternalDialogBox+0xef (FPO: [4,5,4])
05 00554214 770a2523 00554390 00554518 00000000 USER32!SoftModalMessageBox+0x72b (FPO: [1,43,4])
06 00554378 770c0115 013a145b 00000000 0055e988 USER32!MessageBoxWorker+0x29a (FPO: [Non-Fpo])
07 00554400 770c015a 00081cd8 00554518 0fc3af50 USER32!MessageBoxTimeoutW+0x165 (FPO: [6,29,4])
08 00554420 0fcc1618 00081cd8 00554518 0fc3af50 USER32!MessageBoxW+0x1a (FPO: [Non-Fpo])
09 00554440 0fccbe32 00081cd8 00554518 0fc3af50 ucrtbased!__acrt_MessageBoxW+0x38 (FPO: [Non-Fpo]) (CONV: stdcall) [minkernel\crts\ucrt\src\appcrt\internal\winapi_thunks.cpp @ 696] 
0a 00554458 0fccbdd9 00554470 00554490 00554494 ucrtbased!__crt_char_traits<wchar_t>::message_box<HWND__ *,wchar_t const * const &,wchar_t const * const &,unsigned int const &>+0x22 (FPO: [Non-Fpo]) (CONV: cdecl) [minkernel\crts\ucrt\inc\corecrt_internal_traits.h @ 124] 
0b 00554488 0fccbeb6 00554518 0fc3af50 00012012 ucrtbased!common_show_message_box<wchar_t>+0x109 (FPO: [Non-Fpo]) (CONV: cdecl) [minkernel\crts\ucrt\src\appcrt\misc\crtmbox.cpp @ 75] 
0c 0055449c 0fccca00 00554518 0fc3af50 00012012 ucrtbased!__acrt_show_wide_message_box+0x16 (FPO: [Non-Fpo]) (CONV: cdecl) [minkernel\crts\ucrt\src\appcrt\misc\crtmbox.cpp @ 93] 
0d 00556728 0fccd3b2 00000001 0fcbfe4c 00000000 ucrtbased!common_message_window<wchar_t>+0x4a0 (FPO: [Non-Fpo]) (CONV: cdecl) [minkernel\crts\ucrt\src\appcrt\misc\dbgrpt.cpp @ 409] 
0e 00556748 0fcce8d4 00000001 0fcbfe4c 00000000 ucrtbased!__acrt_MessageWindowW+0x22 (FPO: [Non-Fpo]) (CONV: cdecl) [minkernel\crts\ucrt\src\appcrt\misc\dbgrpt.cpp @ 464] 
0f 0055e800 0fccd2ff 00000001 0fcbfe4c 00000000 ucrtbased!_VCrtDbgReportW+0x964 (FPO: [Non-Fpo]) (CONV: cdecl) [minkernel\crts\ucrt\src\appcrt\misc\dbgrptt.cpp @ 673] 
10 0055e82c 0fcbfe4c 00000001 00000000 00000000 ucrtbased!_CrtDbgReportW+0x2f (FPO: [Non-Fpo]) (CONV: cdecl) [minkernel\crts\ucrt\src\appcrt\misc\dbgrpt.cpp @ 278] 
11 0055e850 0fcbfff1 0fc43524 0071fb78 0055e890 ucrtbased!issue_debug_notification+0x1c (FPO: [Non-Fpo]) (CONV: cdecl) [minkernel\crts\ucrt\src\appcrt\internal\report_runtime_error.cpp @ 25] 
12 0055e868 0fcd12ca 0fc43524 6ff6335c 0055e8b0 ucrtbased!__acrt_report_runtime_error+0x11 (FPO: [Non-Fpo]) (CONV: cdecl) [minkernel\crts\ucrt\src\appcrt\internal\report_runtime_error.cpp @ 154] 
13 0055e878 0fcd076d 4a98ddad 013a145b 00000000 ucrtbased!abort+0x1a (FPO: [Non-Fpo]) (CONV: cdecl) [minkernel\crts\ucrt\src\appcrt\startup\abort.cpp @ 51] 
14 0055e8b0 013b3725 0055eac8 0055e958 7792de80 ucrtbased!terminate+0x7d (FPO: [Non-Fpo]) (CONV: cdecl) [minkernel\crts\ucrt\src\appcrt\misc\terminate.cpp @ 59] 
15 0055e8bc 7792de80 0055e988 abad65ee 00000000 D3D12Test!__scrt_unhandled_exception_filter+0x55 (FPO: [Non-Fpo]) (CONV: stdcall) [d:\agent\_work\3\s\src\vctools\crt\vcstartup\src\utility\utility_desktop.cpp @ 94] 
16 0055e958 77f032b2 0055e988 77ed3cf2 0055fc9c KERNELBASE!UnhandledExceptionFilter+0x1a0 (FPO: [Non-Fpo])
17 0055fcac 77ec5e97 ffffffff 77eeae74 00000000 ntdll!__RtlUserThreadStart+0x3d41a
18 0055fcbc 00000000 013a1a7d 00216000 00000000 ntdll!_RtlUserThreadStart+0x1b (FPO: [Non-Fpo])

```

可以知道是有一个MessageBox弹框出现导致0号线程卡死，而出现这个弹框，是因为一个异常出现。  
第16号栈帧可以看到捕获到异常KERNELBASE!UnhandledExceptionFilter，它的参数如下：

```C++
LONG WINAPI UnhandledExceptionFilter(
  _In_ struct _EXCEPTION_POINTERS *ExceptionInfo
);

typedef struct _EXCEPTION_POINTERS {
  PEXCEPTION_RECORD ExceptionRecord;
  PCONTEXT          ContextRecord;
} EXCEPTION_POINTERS, *PEXCEPTION_POINTERS;
```

直接查看结构，可以看到异常基本信息：

```bash
# 取KERNELBASE!UnhandledExceptionFilter第一个参数：0055e988
0:009> dt /r EXCEPTION_POINTERS 0055e988 
hookd3d9!EXCEPTION_POINTERS
   +0x000 ExceptionRecord  : 0x0055eac8 _EXCEPTION_RECORD
      +0x000 ExceptionCode    : 0xe06d7363
      +0x004 ExceptionFlags   : 1
      +0x008 ExceptionRecord  : (null) 
      +0x00c ExceptionAddress : 0x778a4bb2 Void
      +0x010 NumberParameters : 3
      +0x014 ExceptionInformation : [15] 0x19930520
   +0x004 ContextRecord    : 0x0055eb18 _CONTEXT
      +0x000 ContextFlags     : 0x1007f
      +0x004 Dr0              : 0
      +0x008 Dr1              : 0
      +0x00c Dr2              : 0
      +0x010 Dr3              : 0
      +0x014 Dr6              : 0
      +0x018 Dr7              : 0
      +0x01c FloatSave        : _FLOATING_SAVE_AREA
         +0x000 ControlWord      : 0x27f
         +0x004 StatusWord       : 0x120
         +0x008 TagWord          : 0xffff
         +0x00c ErrorOffset      : 0xfb443a1
         +0x010 ErrorSelector    : 0
         +0x014 DataOffset       : 0
         +0x018 DataSelector     : 0
         +0x01c RegisterArea     : [80]  ""
         +0x06c Spare0           : 0
      +0x08c SegGs            : 0x2b
      +0x090 SegFs            : 0x53
      +0x094 SegEs            : 0x2b
      +0x098 SegDs            : 0x2b
      +0x09c Edi              : 0x55f090
      +0x0a0 Esi              : 0xf2615a8
      +0x0a4 Ebx              : 0x216000
      +0x0a8 Edx              : 0
      +0x0ac Ecx              : 3
      +0x0b0 Eax              : 0x55eff8
      +0x0b4 Ebp              : 0x55f050
      +0x0b8 Eip              : 0x778a4bb2
      +0x0bc SegCs            : 0x23
      +0x0c0 EFlags           : 0x212
      +0x0c4 Esp              : 0x55eff8
      +0x0c8 SegSs            : 0x2b
      +0x0cc ExtendedRegisters : [512]  "???"

# .exr和.cxr是分别查看ExceptionRecord和ContextRecord的命令，同时，会将栈切换到异常时上下文
0:009> .exr 0x0055eac8 
ExceptionAddress: 778a4bb2 (KERNELBASE!RaiseException+0x00000062)
   ExceptionCode: e06d7363 (C++ EH exception)
  ExceptionFlags: 00000001
NumberParameters: 3
   Parameter[0]: 19930520
   Parameter[1]: 0055f0c4
   Parameter[2]: 013bd278
  pExceptionObject: 0055f0c4
  _s_ThrowInfo    : 013bd278
  Type            : class HrException
  Type            : class std::runtime_error
  Type            : class std::exception
0:009> .cxr 0x0055eb18 
eax=0055eff8 ebx=00216000 ecx=00000003 edx=00000000 esi=0f2615a8 edi=0055f090
eip=778a4bb2 esp=0055eff8 ebp=0055f050 iopl=0         nv up ei pl nz ac po nc
cs=0023  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00000212
KERNELBASE!RaiseException+0x62:
778a4bb2 8b4c2454        mov     ecx,dword ptr [esp+54h] ss:002b:0055f04c=abad634e

# 当前已经切换到异常上下文，直接查看调用栈信息
0:000> kv
  *** Stack trace for last set context - .thread/.cxr resets it
 # ChildEBP RetAddr  Args to Child              
00 0055f050 0f2697a9 e06d7363 00000001 00000003 KERNELBASE!RaiseException+0x62 (FPO: [4,22,0])
01 0055f0a4 013ae34e 0055f0c4 013bd278 0055f198 VCRUNTIME140D!_CxxThrowException+0xa9 (FPO: [Non-Fpo]) (CONV: stdcall) [d:\agent\_work\1\s\src\vctools\crt\vcruntime\src\eh\throw.cpp @ 133] 
02 0055f198 013abbcd 80070003 aff51556 0055fac0 D3D12Test!ThrowIfFailed+0x4e (FPO: [Non-Fpo]) (CONV: cdecl) [d:\keencode\d3dsample\d3d12test\d3drender.cpp @ 61] 
03 0055f9ec 013aabb9 0055fbcc 0055facc 00216000 D3D12Test!LoadAssets+0x1dd (FPO: [Non-Fpo]) (CONV: cdecl) [d:\keencode\d3dsample\d3d12test\d3drender.cpp @ 196] 
04 0055fac0 013a49ac 00081cd8 013a1a7d 013a1a7d D3D12Test!InitDevice+0x39 (FPO: [Non-Fpo]) (CONV: cdecl) [d:\keencode\d3dsample\d3d12test\d3drender.cpp @ 302] 
05 0055fbcc 013b1f4e 01390000 00000000 00702708 D3D12Test!wWinMain+0x9c (FPO: [Non-Fpo]) (CONV: stdcall) [d:\keencode\d3dsample\d3d12test\d3d12test.cpp @ 44] 
06 0055fbe4 013b1db7 aff510fa 013a1a7d 013a1a7d D3D12Test!invoke_main+0x1e (FPO: [Non-Fpo]) (CONV: cdecl) [d:\agent\_work\3\s\src\vctools\crt\vcstartup\src\startup\exe_common.inl @ 123] 
07 0055fc40 013b1c4d 0055fc50 013b1fc8 0055fc64 D3D12Test!__scrt_common_main_seh+0x157 (FPO: [Non-Fpo]) (CONV: cdecl) [d:\agent\_work\3\s\src\vctools\crt\vcstartup\src\startup\exe_common.inl @ 288] 
08 0055fc48 013b1fc8 0055fc64 77cc8674 00216000 D3D12Test!__scrt_common_main+0xd (FPO: [Non-Fpo]) (CONV: cdecl) [d:\agent\_work\3\s\src\vctools\crt\vcstartup\src\startup\exe_common.inl @ 331] 
09 0055fc50 77cc8674 00216000 77cc8650 ab8016bc D3D12Test!wWinMainCRTStartup+0x8 (FPO: [Non-Fpo]) (CONV: cdecl) [d:\agent\_work\3\s\src\vctools\crt\vcstartup\src\startup\exe_wwinmain.cpp @ 17] 
0a 0055fc64 77ec5ec7 00216000 6fa3cff0 00000000 KERNEL32!BaseThreadInitThunk+0x24 (FPO: [Non-Fpo])
0b 0055fcac 77ec5e97 ffffffff 77eeae74 00000000 ntdll!__RtlUserThreadStart+0x2f (FPO: [SEH])
0c 0055fcbc 00000000 013a1a7d 00216000 00000000 ntdll!_RtlUserThreadStart+0x1b (FPO: [Non-Fpo])

```

找到异常内容了。

