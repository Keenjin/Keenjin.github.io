---
layout: post
title: 【操作指令】Windows命令行
date: 2019-12-30
tags: 操作指令
---

# 通用语法

```bat
rem ==================================================================
rem 关闭echo命令提示
@echo off
rem ==================================================================

rem ==================================================================
rem 获取当前脚本目录
set cur_dir=%~dp0
rem ==================================================================

rem ==================================================================
rem 命令行参数
echo "run script:%0" 
set param1=%1
rem ==================================================================

rem ==================================================================
rem 执行脚本1：在当前cmd环境，同步运行
call ../build.bat
if %errorlevel%==0 (
	echo "build succeed."
)

rem 执行脚本2：新起一个cmd环境，异步执行
start ../build.bat
rem ==================================================================

rem ==================================================================
rem 判断语句
set build_release=%1
if "%build_release%"=="" (
	echo "no build_release参数"
)
rem ==================================================================

rem ==================================================================
rem 文件/目录存在判断
if exist "%cur_dir%\\test" (
	echo "no %cur_dir%\\test exist"
)
if not exist "%cur_dir%\\test" (
	echo "no %cur_dir%\\test not exist"
)
rem ==================================================================

rem ==================================================================
rem for循环
rem 一般都需要利用变量延迟展开，否则并使用!xxx!来操作变量
setlocal EnableDelayedExpansion
for %%i in ('ipconfig /all') do (
	set out_str=%%i
	echo !out_str!
)
setlocal DisableDelayedExpansion
rem ==================================================================

rem ==================================================================
rem 标签跳转
set param1=%1
if "%param1%"=="" (
	goto failed
) else (
	goto succeed
)

:failed
echo "run label1"
goto finished

:succeed
echo "run label2"

:finished
echo "finished."
rem ==================================================================

rem ==================================================================
rem 函数
rem 函数没有返回值，只能通过全局变量来设置。可以传递参数，跟脚本一样传递
echo "run func1"
set param1=1
set param2=2
call :func1 %param1% %param2%
echo "run func1 finished."

rem 为何需要goto？不然就会去执行函数部分，这里需要能跳出去
goto finished

:func1
set p=%1
echo "run func1, param: %p%"
goto:eof

:finished
echo "finished"
rem ==================================================================

rem ==================================================================
rem 变量计算
set try_cnt=0
set /a try_cnt+=1
rem ==================================================================
```

# 文件操作

```bat
rem 拷贝文件或目录到某一个目录，注意：尾部反斜杠的使用
rem 其中，/Y表示不用确认，/E表示拷贝空目录
xcopy c:\test d:\test\ /Y /E
xcopy c:\test\ttt.txt d:\test\ /Y /E

rem 判断某个命令行执行结果返回值
python --version
if %errorlevel%==0 echo "success"
else echo "failed."
```


# 系统相关

```bash
# 关闭Win10 休眠管理
powercfg -h off

# 单网卡同时支持dhcp和静态ip
# Windows 10要求：1703以上版本
netsh int ipv4 set interface "以太网" dhcpstaticipcoexistence=enabled
netsh int ipv4 add address "以太网" 192.168.100.101 255.255.255.0

# 管理员模式
修改注册表：HKEY_LOCAL_MACHINE/SOFTWARE/MICROSOFT/WINDOWS NT/CURRENTVERSION/WINLOGON
（1）添加SpecialAccounts/UserList
（2）然后添加键值DWORD(32位)，名称Administrator，键值1
（3）开机进入带命令提示符安全模式，输入net user administrator  /active:yes;
```

# 编译相关

```bash
devenv mysln.sln /Rebuild "Release|x86" /Out %build_dir%\build_release_x86.log
devenv mysln.sln /Rebuild "Release|x64" /Out %build_dir%\build_release_x64.log

msbuild /m /p:Configuration=Release /p:Platform=x86 mysln.sln
```

# 示例1：检测环境配置，执行编译打包

![png](/images/post/win_cmd/bat_sample1.png)

- configure.bat：用于检测和准备环境  
- build_all.bat：用于vs编译
- make_publish.bat：用于构建发布包

```bat
rem =================
rem configure.bat ===
rem =================

@echo off
echo "made by Keen."

echo "run: %0"

set cur_dir=%~dp0
set bin_dir=%cur_dir%bin
set bin_publish_dir=%cur_dir%publish\bin
set qt_ver=5.12.4
set quit_mode=%1

echo "makesure you have installed QT and Visual Studio 2017!"

rem ========================================================================================
rem 检查vs2017是否安装
rem ========================================================================================
echo ">>>>>>>>>>>>>>>> check Visual Studio 2017 installed!"
for /f "tokens=2,*" %%a in ('reg query HKEY_CURRENT_USER\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\254f0e11 /v "InstallLocation" 2^>NUL ^| findstr InstallLocation') do set "vs2017_dir=%%b"
if exist "%vs2017_dir%" (
	goto findedvs
)
for /f "tokens=2,*" %%a in ('reg query HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\254f0e11 /v "InstallLocation" 2^>NUL ^| findstr InstallLocation') do set "vs2017_dir=%%b"
if exist "%vs2017_dir%" (
	goto findedvs
)
for /f "tokens=2,*" %%a in ('reg query HKEY_CURRENT_USER\Software\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\254f0e11 /v "InstallLocation" 2^>NUL ^| findstr InstallLocation') do set "vs2017_dir=%%b"
if exist "%vs2017_dir%" (
	goto findedvs
)
for /f "tokens=2,*" %%a in ('reg query HKEY_LOCAL_MACHINE\Software\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\254f0e11 /v "InstallLocation" 2^>NUL ^| findstr InstallLocation') do set "vs2017_dir=%%b"
if exist "%vs2017_dir%" (
	goto findedvs
)
set vs2017_dir=C:\Program Files (x86)\Microsoft Visual Studio\2017\Community\
if exist "%vs2017_dir%" (
	goto findedvs
)
set vs2017_dir=D:\Program Files (x86)\Microsoft Visual Studio\2017\Community\
if exist "%vs2017_dir%" (
	goto findedvs
)
rem todo 后面再考虑其他可能性

echo "Visual Studio 2017 not founded!"
goto failed

:findedvs
echo "Visual Studio 2017 have installed! path: %vs2017_dir%"

rem ========================================================================================
rem 检查qt是否安装
rem ========================================================================================
echo ">>>>>>>>>>>>>>>> check QT%qt_ver% installed!"

for /f "tokens=1,2,*" %%i in ('reg query HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Uninstall\{05d11b2b-bddf-40c9-b2b8-b5a405429285} /v InstallLocation ^| find /i "InstallLocation"') do set "qt_dir=%%k"
if exist "%qt_dir%" (
	goto findedqt
)
for /f "tokens=1,2,*" %%i in ('reg query HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Uninstall\{05d11b2b-bddf-40c9-b2b8-b5a405429285} /v InstallLocation ^| find /i "InstallLocation"') do set "qt_dir=%%k"
if exist "%qt_dir%" (
	goto findedqt
)
for /f "tokens=1,2,*" %%i in ('reg query HKEY_CURRENT_USER\Software\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\{05d11b2b-bddf-40c9-b2b8-b5a405429285} /v InstallLocation ^| find /i "InstallLocation"') do set "qt_dir=%%k"
if exist "%qt_dir%" (
	goto findedqt
)
for /f "tokens=1,2,*" %%i in ('reg query HKEY_LOCAL_MACHINE\Software\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\{05d11b2b-bddf-40c9-b2b8-b5a405429285} /v InstallLocation ^| find /i "InstallLocation"') do set "qt_dir=%%k"
if exist "%qt_dir%" (
	goto findedqt
)
set qt_dir=C:\Qt\Qt5.12.4
if exist "%qt_dir%" (
	goto findedqt
)
set qt_dir=D:\Qt\Qt5.12.4
if exist "%qt_dir%" (
	goto findedqt
)

echo "QT%qt_ver% not founded!"
goto failed

:findedqt
echo "QT%qt_ver% have installed! path: %qt_dir%"

rem ========================================================================================
rem 拷贝QT依赖库
rem ========================================================================================
echo ">>>>>>>>>>>>>>>> copy QT%qt_ver%"
set qt_src_file_dir=%qt_dir%\%qt_ver%\msvc2017\bin
xcopy "%qt_src_file_dir%\Qt5Widgetsd.dll" %bin_dir%\ /Y /E
xcopy "%qt_src_file_dir%\Qt5Widgets.dll" %bin_dir%\ /Y /E
xcopy "%qt_src_file_dir%\Qt5Cored.dll" %bin_dir%\ /Y /E
xcopy "%qt_src_file_dir%\Qt5Core.dll" %bin_dir%\ /Y /E
xcopy "%qt_src_file_dir%\Qt5Guid.dll" %bin_dir%\ /Y /E
xcopy "%qt_src_file_dir%\Qt5Gui.dll" %bin_dir%\ /Y /E

rem ========================================================================================
rem 拷贝dependencies依赖库
rem ========================================================================================
echo ">>>>>>>>>>>>>>>> copy dependencies2017"
xcopy %cur_dir%deps\dependencies2017\win32\bin %bin_dir%\ /Y /E
if %errorlevel% NEQ 0 (
	echo "xcopy dependencies2017 failed!"
	goto failed
)

rem ========================================================================================
rem 拷贝obs-data
rem ========================================================================================
echo ">>>>>>>>>>>>>>>> copy obs-data"
xcopy %bin_publish_dir%\obs-data %bin_dir%\obs-data\ /Y /E
if %errorlevel% NEQ 0 (
	echo "xcopy obs-data failed!"
	goto failed
)

:successed
echo ">>>>>>>>>>>>>>>> run %0 successed!"
if "%quit_mode%" NEQ "/s" (
	timeout /T 3
)
exit 0

:failed
echo ">>>>>>>>>>>>>>>> run %0 failed!"
if "%quit_mode%" NEQ "/s" (
	pause
)
exit 1

```

```bat
rem =================
rem build_all.bat ===
rem =================

@echo off
echo "made by Keen."

echo "run: %0"

set cur_dir=%~dp0
set script_configure=%cur_dir%\configure.bat /s
set build_dir=%cur_dir%\build.dir

rem ========================================================================================
rem 先执行configure.bat配置环境
rem ========================================================================================
start /wait %script_configure%
if %errorlevel% NEQ 0 (
	echo "run %script_configure% failed."
	goto failed
)

rem ========================================================================================
rem 然后找到vs的可执行程序路径
rem ========================================================================================
for /f "tokens=2,*" %%a in ('reg query HKEY_CURRENT_USER\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\254f0e11 /v "InstallLocation" 2^>NUL ^| findstr InstallLocation') do set "vs2017_dir=%%b"
if exist "%vs2017_dir%" (
	goto findedvs
)
for /f "tokens=2,*" %%a in ('reg query HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\254f0e11 /v "InstallLocation" 2^>NUL ^| findstr InstallLocation') do set "vs2017_dir=%%b"
if exist "%vs2017_dir%" (
	goto findedvs
)
for /f "tokens=2,*" %%a in ('reg query HKEY_CURRENT_USER\Software\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\254f0e11 /v "InstallLocation" 2^>NUL ^| findstr InstallLocation') do set "vs2017_dir=%%b"
if exist "%vs2017_dir%" (
	goto findedvs
)
for /f "tokens=2,*" %%a in ('reg query HKEY_LOCAL_MACHINE\Software\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\254f0e11 /v "InstallLocation" 2^>NUL ^| findstr InstallLocation') do set "vs2017_dir=%%b"
if exist "%vs2017_dir%" (
	goto findedvs
)
set vs2017_dir=C:\Program Files (x86)\Microsoft Visual Studio\2017\Community\
if exist "%vs2017_dir%" (
	goto findedvs
)
set vs2017_dir=D:\Program Files (x86)\Microsoft Visual Studio\2017\Community\
if exist "%vs2017_dir%" (
	goto findedvs
)
rem todo 后面再考虑其他可能性

echo "Visual Studio 2017 not founded!"
goto failed

:findedvs
echo "Visual Studio 2017 have installed! path: %vs2017_dir%"

rem ========================================================================================
rem 编译solution
rem ========================================================================================
set main_sln=%cur_dir%\project\obs-framework.sln
"%vs2017_dir%\Common7\IDE\devenv" %main_sln% /Rebuild "Release|x86" /Out %build_dir%\build_release_x86.log
"%vs2017_dir%\Common7\IDE\devenv" %main_sln% /Rebuild "Release|x64" /Out %build_dir%\build_release_x64.log

:successed
echo ">>>>>>>>>>>>>>>> run successed!"
timeout /T 3
exit 0

:failed
echo ">>>>>>>>>>>>>>>> run failed!"
pause
exit 1

```

```bat
rem ====================
rem make_publish.bat ===
rem ====================

@echo off
echo "made by Keen."

echo "run: %0"

set cur_dir=%~dp0
set bin_dir=%cur_dir%bin
set deps_dir=%cur_dir%deps\deps\dependencies2017\win32\bin
set publish_bin_dir=%cur_dir%\publish\bin
set publish_pdb_dir=%cur_dir%\publish\pdbs

rem 拷贝Dependencies2017依赖dll（现在完全拷贝，后续根据需要来拷贝）
xcopy %deps_dir% %publish_bin_dir%\ /Y /E

rem 拷贝lib库pdb
xcopy %bin_dir%\ipc-util.pdb %publish_pdb_dir%\ /Y /E
xcopy %bin_dir%\jansson.pdb %publish_pdb_dir%\ /Y /E

rem 拷贝必要的二进制和pdb
xcopy %bin_dir%\w32-pthreads.dll %publish_bin_dir%\ /Y /E
xcopy %bin_dir%\w32-pthreads.pdb %publish_pdb_dir%\ /Y /E

xcopy %bin_dir%\libobs-d3d11.dll %publish_bin_dir%\ /Y /E
xcopy %bin_dir%\libobs-d3d11.pdb %publish_pdb_dir%\ /Y /E

xcopy %bin_dir%\libobs-opengl.dll %publish_bin_dir%\ /Y /E
xcopy %bin_dir%\libobs-opengl.pdb %publish_pdb_dir%\ /Y /E

xcopy %bin_dir%\obs.dll %publish_bin_dir%\ /Y /E
xcopy %bin_dir%\obs.pdb %publish_pdb_dir%\ /Y /E

xcopy %bin_dir%\obsglad.dll %publish_bin_dir%\ /Y /E
xcopy %bin_dir%\obsglad.pdb %publish_pdb_dir%\ /Y /E

xcopy %bin_dir%\obs-plugins\win-capture.dll %publish_bin_dir%\ /Y /E
xcopy %bin_dir%\obs-plugins\win-capture.pdb %publish_pdb_dir%\ /Y /E

xcopy %bin_dir%\obs-data\plugins\get-graphics-offsets32.exe %publish_bin_dir%\ /Y /E
xcopy %bin_dir%\obs-data\plugins\get-graphics-offsets32.pdb %publish_pdb_dir%\ /Y /E
xcopy %bin_dir%\obs-data\plugins\get-graphics-offsets64.exe %publish_bin_dir%\ /Y /E
xcopy %bin_dir%\obs-data\plugins\get-graphics-offsets64.pdb %publish_pdb_dir%\ /Y /E

xcopy %bin_dir%\obs-data\plugins\inject-helper32.exe %publish_bin_dir%\ /Y /E
xcopy %bin_dir%\obs-data\plugins\inject-helper32.pdb %publish_pdb_dir%\ /Y /E
xcopy %bin_dir%\obs-data\plugins\inject-helper64.exe %publish_bin_dir%\ /Y /E
xcopy %bin_dir%\obs-data\plugins\inject-helper64.pdb %publish_pdb_dir%\ /Y /E

:successed
echo ">>>>>>>>>>>>>>>> run successed!"
timeout /T 3
exit

:failed
echo ">>>>>>>>>>>>>>>> run failed!"
pause

```