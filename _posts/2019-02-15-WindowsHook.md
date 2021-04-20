---
layout: post
title: 【安全技术】Windows Hook总结
date: 2019-02-15
tags: 安全技术
---

<!-- TOC -->

- [1. 模块注入](#1-模块注入)
    - [1.1. 准备工作](#11-准备工作)
        - [1.1.1. 被注入DLL Demo](#111-被注入dll-demo)
        - [1.1.2. 提权](#112-提权)
    - [1.2. 注入方式](#12-注入方式)
        - [1.2.1. 远线程注入](#121-远线程注入)
        - [1.2.2. APC注入](#122-apc注入)
        - [1.2.3. 消息钩子注入](#123-消息钩子注入)
        - [1.2.4. 事件钩子注入](#124-事件钩子注入)
        - [1.2.5. 导入表注入](#125-导入表注入)
        - [1.2.6. 其他注入技术](#126-其他注入技术)
- [2. 函数Hook](#2-函数hook)
    - [2.1. 修改函数内存代码](#21-修改函数内存代码)
    - [2.2. 修改模块的导入表](#22-修改模块的导入表)

<!-- /TOC -->

# 1. 模块注入

## 1.1. 准备工作

### 1.1.1. 被注入DLL Demo

```C++
#include <process.h>
#include <Shlwapi.h>

// 用于64位远线程注入时共享句柄
#pragma data_seg("Shared")
HMODULE g_hDll = NULL;
#pragma data_seg()
#pragma comment(linker, "/section:Shared,rws")

extern "C" __declspec(dllexport) HMODULE GetCurrentModule()
{
	return g_hDll;
}

unsigned __stdcall Load(void*)
{
	// 获取当前进程
	TCHAR szPath[MAX_PATH] = { 0 };
	GetModuleFileName(NULL, szPath, MAX_PATH);
	PathStripPath(szPath);
	TCHAR szMsg[1024] = { 0 };
	wsprintf(szMsg, L"InjectTestDll.dl 注入成功，进程名：%s，PID：%d", szPath, GetCurrentProcessId());
	MessageBox(NULL, szMsg, NULL, 0);
	return 0;
}

BOOL APIENTRY DllMain( HMODULE hModule,
                       DWORD  ul_reason_for_call,
                       LPVOID lpReserved
                     )
{
    switch (ul_reason_for_call)
    {
    case DLL_PROCESS_ATTACH:
		_beginthreadex(NULL, 0, Load, NULL, 0, NULL);
    case DLL_THREAD_ATTACH:
    case DLL_THREAD_DETACH:
    case DLL_PROCESS_DETACH:
        break;
    }
    return TRUE;
}

// 给SetWindowsHookEx使用的
extern "C" __declspec(dllexport) LRESULT CALLBACK DummyMsgProc(int code, WPARAM wParam, LPARAM lParam)
{
	return CallNextHookEx(0, code, wParam, lParam);
}

// 给SetWinEventHook使用的
extern "C" __declspec(dllexport) VOID CALLBACK DummyEventProc(HWINEVENTHOOK hWinEventHook,
	DWORD         event,
	HWND          hwnd,
	LONG          idObject,
	LONG          idChild,
	DWORD         idEventThread,
	DWORD         dwmsEventTime)
{
	return;
}

// 给导入表注入使用的
extern "C" __declspec(dllexport) VOID DummyImportApi()
{
	return;
}
```

### 1.1.2. 提权

OpenProcess存在权限问题，打开目标进程需要具备一定的访问权限。但是，微软官方提供了这样一个声明：“If the caller has enabled the SeDebugPrivilege privilege, the requested access is granted regardless of the contents of the security descriptor.”。也就是，只要当前进程具备SE_DEBUG_NAME权限，那么相当于无视OpenProcess的访问限制，当然，前提是当前进程令牌具备SE_DEBUG_NAME的权限（Windows默认设置的话，必须是管理员才具备此权限级别），只是默认不打开，这里执行开启动作。

```C++
void load_debug_privilege(void)
{
	const DWORD flags = TOKEN_ADJUST_PRIVILEGES | TOKEN_QUERY;
	TOKEN_PRIVILEGES tp;
	HANDLE token;
	LUID val;

	if (!OpenProcessToken(GetCurrentProcess(), flags, &token)) {
		return;
	}

	if (!!LookupPrivilegeValue(NULL, SE_DEBUG_NAME, &val)) {
		tp.PrivilegeCount = 1;
		tp.Privileges[0].Luid = val;
		tp.Privileges[0].Attributes = SE_PRIVILEGE_ENABLED;

		if (!AdjustTokenPrivileges(token, false, &tp, sizeof(tp), NULL,
			NULL) || GetLastError() == ERROR_NOT_ALL_ASSIGNED)
		{
			// 调整令牌权限失败
		}
	}

	CloseHandle(token);
}
```

## 1.2. 注入方式

### 1.2.1. 远线程注入

系统机制：
- CreateRemoteThread可以在其他进程地址空间创建一个线程进行执行，创建后，将由内核模块切换到其他进程地址空间，执行线程创建操作
- 由于实际到执行过程，是在其他进程空间，因此参数也必须是其他地址空间的，系统提供VirtualAllocEx可以在其他地址空间分配虚拟内存（前提是我们打开目标进程时，需要具备虚拟地址空间读写能力）
- 由于LoadLibrary函数声明与CreateRemoteThread线程过程函数类似，因此可以直接作为线程过程函数
- 由于LoadLibrary函数在调用时，并不知道kernel32.dll加载的地址（地址随机化以及地址冲突决定的），本质上只是一个链接到kernel32.dll的一个符号，被记录到可执行程序导入表中的一个地址，在进程启动时会将导入表加载到进程地址空间，并根据kernel32.dll的地址，修复导入表。在代码调用LoadLibrary时，编译器实际只是通过导入表基地址的一个相对偏移，来查找进程地址空间的导入表的LoadLibrary项对应的地址，进而去调用真实的kernel32.dll的地址。也就是说，我们自己的进程，会由链接器创建一个“LoadLibrary”到我们自己的导入表的某个偏移（例如：32），当我们使用CreateRemoteThread并将LoadLibrary作为线程函数时，本质是从我们设定的自身导入表的偏移，在目标进程的导入表中，去查LoadLibrary的真实地址。如果目标进程LoadLibrary的偏移与我们要查找的偏移不一样，那么可能会出现任何问题。因此，需要使用GetProcAddress来获取LoadLibrary的真实地址，作为CreateRemoteThread的参数传递进去。为何这样做可以？因为GetProcAddress本质是直接读取目标模块的导出表（也就是，本进程地址空间中的kernel32.dll的导出表），计算出真实地址结果来作为参数的，而kernel32.dll被任何进程加载，都一定是相同虚拟地址（windows dll代码复用和数据区copy-on-write机制来支持这个特性的，以便于复用节省内存空间），所以在我们自身进程通过GetProcAddress计算出来的地址，与目标进程中LoadLibrary地址一定是一致的。

引申点：
- 我们可以用LoadLibrary作为参数去加载我们的地址，实际上，我们也可以去调用任何一个可能被目标进程加载的模块的导出函数（函数声明保持相同或类似，参数个数一致），来远程执行某些操作
- 既然LoadLibrary本质是查找的导入表，如果我们能够修改目标进程导入表，也就可以自定义注入和hook
- 卸载注入，同样的机制调用FreeLibrary。当然，如果注入dll还存在线程未执行完毕，FreeLibrary会失败。
- 对于管理员权限运行的目标进程，必须使用管理员权限的注入进程来执行注入。

```C++
HMODULE RemoteThreadInject(DWORD dwPID, LPCWSTR lpszDll)
{
	// Step1：OpenProcess打开目标进程（需要具备PROCESS_CREATE_THREAD|PROCESS_VM_OPERATION|PROCESS_VM_WRITE操作权限）
	// Step2：VirtualAllocEx在目标进程分配一段虚拟内存，长度为(dll路径+1)
	// Step3：WriteProcessMemory将dll路径写入Step2分配的虚拟地址空间
	// Step4：GetProcAddress获取到LoadLibrary的真实地址
	// Step5：CreateRemoteThread在目标进程中创建一个线程，并将Step4种查询到的地址作为函数过程参数传递进去
	// Step6：WaitForSingleObject等待远线程执行完毕
	// Step7：VirtualFreeEx释放Step2分配的虚拟地址，CloseHandle关闭线程进程句柄
	HANDLE hProcess = NULL;
	HANDLE hThread = NULL;
	LPVOID pRemoteDll = NULL;
    HMODULE hRemoteMod = NULL;

	do 
	{
		hProcess = OpenProcess(
			PROCESS_CREATE_THREAD |		// CreateRemoteThread
			PROCESS_VM_OPERATION |		// VirtualAllocEx | VirtualFreeEx
			PROCESS_VM_WRITE,			// WriteProcessMemory
			FALSE, dwPID
		);
		if (hProcess == NULL)		{
			break;
		}

		int nSize = (lstrlen(lpszDll) + 1) * sizeof(WCHAR);
		pRemoteDll = VirtualAllocEx(hProcess, NULL, nSize, MEM_COMMIT, PAGE_READWRITE);
		if (pRemoteDll == NULL)		{
			break;
		}

		if (!WriteProcessMemory(hProcess, pRemoteDll, lpszDll, nSize, NULL))		{
			break;
		}

		PTHREAD_START_ROUTINE pfnLoadLibrary = (PTHREAD_START_ROUTINE)GetProcAddress(GetModuleHandle(L"kernel32.dll"), "LoadLibraryW");
		if (pfnLoadLibrary == NULL)		{
			break;
		}

		hThread = CreateRemoteThread(hProcess, NULL, 0, pfnLoadLibrary, pRemoteDll, 0, NULL);
		if (hThread == NULL)		{
			break;
		}

		WaitForSingleObject(hThread, INFINITE);

#ifdef _WIN64
		HMODULE hMod = LoadLibraryW(lpszDll);       // 虽然大多数场景下，同一个dll在不同进程中的地址是一样的，但是为了防止地址随机化带来的可能问题，所以还是采用共享内存来实现模块地址读取。
		typedef HMODULE (*FPROC_GETCURRENTMODULE)();
		FPROC_GETCURRENTMODULE fnGetModule = (FPROC_GETCURRENTMODULE)GetProcAddress(hMod, "GetCurrentModule");
		hRemoteMod = fnGetModule();
		FreeLibrary(hMod);
#else
		DWORD hRemoteDll = NULL;
		GetExitCodeThread(hThread, &hRemoteDll);    // 这个只能返回一个DWORD值，本身存在截断，所以才用到了共享内存
		hRemoteMod = (HMODULE)hRemoteDll;
#endif
	} while (FALSE);

	if (pRemoteDll)	{
		VirtualFreeEx(hProcess, pRemoteDll, 0, MEM_RELEASE);
	}
	if (hThread)	{
		CloseHandle(hThread);
	}
	if (hProcess)	{
		CloseHandle(hProcess);
	}

    return hRemoteMod;
}

void RemoteThreadUnInject(DWORD dwPID, HMODULE hRemoteDll)
{
	HANDLE hProcess = NULL;
	HANDLE hThread = NULL;

	do
	{
		hProcess = OpenProcess(
			PROCESS_CREATE_THREAD |		// CreateRemoteThread
			PROCESS_VM_OPERATION |		// VirtualAllocEx | VirtualFreeEx
			PROCESS_VM_WRITE,			// WriteProcessMemory
			FALSE, dwPID
		);
		if (hProcess == NULL)	{
			break;
		}

		PTHREAD_START_ROUTINE pfnFreeLibrary = (PTHREAD_START_ROUTINE)GetProcAddress(GetModuleHandle(L"kernel32.dll"), "FreeLibrary");
		if (pfnFreeLibrary == NULL)	{
			break;
		}

		hThread = CreateRemoteThread(hProcess, NULL, 0, pfnFreeLibrary, hRemoteDll, 0, NULL);
		if (hThread == NULL){
			break;
		}

		WaitForSingleObject(hThread, INFINITE);

	} while (FALSE);

	if (hThread){
		CloseHandle(hThread);
	}
	if (hProcess){
		CloseHandle(hProcess);
	}
}
```

### 1.2.2. APC注入

系统机制：
- APC本质是软中断
- 每一个线程都有一个APC队列，当线程即将挂起时（也就是不得不等待某些事件时），会去检查这个APC队列，然后依次去执行
- 我们通常的多线程同步机制，举个例子：比如线程A和线程B，线程A执行的过程中，如果发现比较耗时的任务，就交给线程B去执行，线程A继续做自己的事情，但是线程A执行到一定过程后，需要得到线程B执行结果，也就是必须等待线程B完成，那么通常会使用一个Event去WaitForSingleObject。当线程B执行完毕，主动去触发Event（SetEvent），那样的话，线程A就可以完成结果同步继续往下走。而APC提供了不同的信号同步机制，做法是：当线程A执行过程中，发现耗时任务，交给线程B执行，然后线程A继续执行自己的事情，当发现需要线程B的结果，就SleepEx(alert=true)挂起线程A自己。如果线程B完成结果，就QueueUserAPC往线程A的APC队列中扔一个APC中断，此时线程A就会被唤醒。
- QueueUserAPC可以在任意进程中调用，对于注入原理，与CreateRemoteThread类似，但是前提是，它的触发条件，是注入的线程必须拥有警告状态，也就是通过SleepEx、SignalObjectAndWait、WaitForSingleObjectsEx、WaitForMultipleObjectsEx、MsgWaitForMultipleObjectsEx
- APC异步过程调用是异步的，相当于插入一个等待结束的回调，QueueUserAPC可以在任意时刻调用，不确定执行过程什么时候执行
- 使用APC还有一个好处，是可以通过参数传递

引申：
- 一般UI线程中不会使用上述能触发警告状态的API，因为这些api会导致线程挂起
- APC可以实现跨进程调用，当作信号触发器，而无需采用全局命名事件等

使用示例：

```C++
VOID NTAPI ApcFunc(ULONG_PTR pValue)
{
	printf("Hello, this is APC and the parameter value is %d\n", *((DWORD*)pValue));
	delete (DWORD*)pValue;
}

DWORD WINAPI Thread_A_Proc(LPVOID)
{
	printf("Thread A is waiting...\n");

    // Do Something
    // 等待B线程的结果
	SleepEx(INFINITE, TRUE);

    // B线程唤醒了我
	
	return 0L;
}

DWORD WINAPI Thread_B_Proc(LPVOID hThread_A)
{
	printf("Thread B will wake up Thread A in 3 sec...\n");

	Sleep(3000);
	DWORD* dwData = new DWORD(2013);
	QueueUserAPC(ApcFunc, (HANDLE)hThread_A, (ULONG_PTR)dwData);
	return 0L;
}

int main()
{
    HANDLE hThread_A = handles[0] = CreateThread(NULL, 0, Thread_A_Proc, NULL, 0, NULL);
	HANDLE hThread_B = handles[1] = CreateThread(NULL, 0, Thread_B_Proc, hThread_A, 0, NULL);

	WaitForMultipleObjects(2, handles, TRUE, INFINITE);
}
```

注入方法：

```C++
HMODULE ApcInject(HWND hWnd, LPCWSTR lpszDll)
{
	HANDLE hProcess = NULL;
	HANDLE hThread = NULL;
	LPVOID pRemoteDll = NULL;
	HMODULE hRemoteMod = NULL;

	DWORD dwPID = 0;
	DWORD dwTID = GetWindowThreadProcessId(hWnd, &dwPID);

	do
	{
		hProcess = OpenProcess(
			PROCESS_VM_OPERATION |		// VirtualAllocEx | VirtualFreeEx
			PROCESS_VM_WRITE,			// WriteProcessMemory
			FALSE, dwPID
		);
		if (hProcess == NULL)
		{
			std::cout << "OpenProcess err: " << GetLastError() << std::endl;
			break;
		}

		int nSize = (lstrlen(lpszDll) + 1) * sizeof(WCHAR);
		pRemoteDll = VirtualAllocEx(hProcess, NULL, nSize, MEM_COMMIT, PAGE_READWRITE);
		if (pRemoteDll == NULL)
		{
			std::cout << "VirtualAllocEx err: " << GetLastError() << std::endl;
			break;
		}

		if (!WriteProcessMemory(hProcess, pRemoteDll, lpszDll, nSize, NULL))
		{
			std::cout << "WriteProcessMemory err: " << GetLastError() << std::endl;
			break;
		}

		PAPCFUNC pfnLoadLibrary = (PAPCFUNC)GetProcAddress(GetModuleHandle(L"kernel32.dll"), "LoadLibraryW");
		if (pfnLoadLibrary == NULL)
		{
			std::cout << "GetProcAddress err: " << GetLastError() << std::endl;
			break;
		}

		hThread = OpenThread(THREAD_SET_CONTEXT, FALSE, dwTID);
		if (hThread == NULL)
		{
			std::cout << "OpenThread err: " << GetLastError() << std::endl;
			break;
		}

		if (0 == QueueUserAPC(pfnLoadLibrary, hThread, (ULONG_PTR)pRemoteDll))
		{
			std::cout << "QueueUserAPC err: " << GetLastError() << std::endl;
			break;
		}

		WaitForSingleObject(hThread, INFINITE);
#ifdef _WIN64
		HMODULE hMod = LoadLibraryW(lpszDll);
		typedef HMODULE(*FPROC_GETCURRENTMODULE)();
		FPROC_GETCURRENTMODULE fnGetModule = (FPROC_GETCURRENTMODULE)GetProcAddress(hMod, "GetCurrentModule");
		hRemoteMod = fnGetModule();
		FreeLibrary(hMod);
#else
		DWORD hRemoteDll = NULL;
		GetExitCodeThread(hThread, &hRemoteDll);
		hRemoteMod = (HMODULE)hRemoteDll;
#endif

	} while (FALSE);

	if (pRemoteDll)
	{
		VirtualFreeEx(hProcess, pRemoteDll, 0, MEM_RELEASE);
	}
	if (hThread)
	{
		CloseHandle(hThread);
	}
	if (hProcess)
	{
		CloseHandle(hProcess);
	}

	return hRemoteMod;
}
```

### 1.2.3. 消息钩子注入

系统机制：
- Windows在User32.dll中提供了非常灵活的消息钩子机制，在处理一些消息事件时，有一些钩子队列，执行某些api时（例如GetMessage），会先通过钩子执行一些动作
- 每一个钩子绑定了线程、模块、钩子函数地址
- 任意进程，可以通过SetWindowsHookEx，往任意窗口线程中，安装一个钩子，由于安装的钩子函数传递的地址，是调用进程地址空间的地址，而这个dll在调用进程和被调用进程中的偏移地址可能不一样，那么系统实际是会通过相对偏移，来求解实际的钩子函数地址，绑定到钩子上，求解方式：GetMsgProc B = hInstDll B + （GetMsgProc A - hInstDll A）。这一点比远线程调用要好，不用我们自己保证地址一致。
- 卸载使用UnhookWindowsHookEx即可。但是对于全局钩子失效。

引申：
- 对于管理员权限运行的目标进程，必须使用管理员权限的注入进程来执行注入。

```C++
HHOOK WinHookInject(HWND hWnd, LPCWSTR lpszDll)
{
	HMODULE hDll = ::LoadLibrary(lpszDll);
#ifdef _WIN64
	HOOKPROC fnProc = (HOOKPROC)GetProcAddress(hDll, "DummyMsgProc");
#else
	HOOKPROC fnProc = (HOOKPROC)GetProcAddress(hDll, "_DummyMsgProc@12");
#endif
	DWORD dwTID = GetWindowThreadProcessId(hWnd, NULL);
	HHOOK hook = SetWindowsHookExW(WH_GETMESSAGE, fnProc, hDll, dwTID);
	if (hook != NULL)
	{
		PostThreadMessage(dwTID, WM_NULL, 0, 0);
	}
	return hook;
}

void WinHookUnInject(HHOOK hook)
{
	UnhookWindowsHookEx(hook);
}
```

### 1.2.4. 事件钩子注入

系统机制：
- 同SetWindowsHookEx类似，不同之处在于，此方式监控的是Event，而SetWindowsHookEx监控的是Message

引申：
- 对于管理员权限运行的目标进程，必须使用管理员权限的注入进程来执行注入。

```C++
HWINEVENTHOOK WinEventHookInject(DWORD dwPID, LPCWSTR lpszDll)
{
	HMODULE hDll = ::LoadLibrary(lpszDll);
#ifdef _WIN64
	WINEVENTPROC fnProc = (WINEVENTPROC)GetProcAddress(hDll, "DummyEventProc");
#else
	WINEVENTPROC fnProc = (WINEVENTPROC)GetProcAddress(hDll, "_DummyEventProc@28");
#endif
	SetWinEventHook(
		EVENT_SYSTEM_FOREGROUND,
		//EVENT_OBJECT_DESCRIPTIONCHANGE,
		EVENT_OBJECT_END,
		hDll, fnProc, dwPID, 0,
		WINEVENT_INCONTEXT | WINEVENT_SKIPOWNPROCESS);
	MSG msg;
	while (GetMessageW(&msg, NULL, 0, 0))
	{
		TranslateMessage(&msg);
		DispatchMessage(&msg);
	}
	return NULL;    // 懒得写线程了，后面为了能够退出，可以用线程方式，设置while循环退出事件
}

void WinEventHookUnInject(HWINEVENTHOOK hook)
{
	UnhookWinEvent(hook);
}
```

### 1.2.5. 导入表注入

系统机制：
- 基于PE文件的结构而来的
- PE文件中存在导入表、导出表等，在PE Loader加载时，会根据导入表中包含的dll路径进行dll加载
- 此处采用的是静态修改PE文件的方式，此方式属于常见感染型病毒采用的技术，会破坏PE文件签名，无法校验。当然，可以利用驱动在PE加载时修改导入表，而不修改PE文件本身
- 卸载的话，直接将拷贝文件替换回去即可，这个需要在下一次exe启动后生效

引申：
- 此处只利用了其加载dll的能力，实际上，可以遍历导入表，将对应的函数替换成我们自己的，直接实现函数Hook

```C++
class CImportTableInject
{
public:
	CImportTableInject()
		: m_hPEFile(NULL)
		, m_hPEMapping(NULL)
		, m_pPEImageBase(NULL)
	{
	}
	~CImportTableInject()
	{
		Clean();
	}

	void Inject(LPCWSTR lpszPEFile, LPCWSTR lpszDll)
	{
		do 
		{
			if (!OpenPE(lpszPEFile))
			{
				RestorePE();
				break;
			}

			if (!IsPE())
			{
				RestorePE();
				break;
			}

			std::string strDllPath = WStringToString(lpszDll);
			if (!Add("Inject", strDllPath.c_str(), "DummyImportApi"))
			{
				RestorePE();
				break;
			}

		} while (FALSE);
	}

    void UnInject()
    {
        RestorePE();
    }
protected:
	std::string WStringToString(const std::wstring& str)
	{
		unsigned int len = str.size() * 4;
		setlocale(LC_CTYPE, "");
		char *p = new char[len];
		unsigned int converted = 0;
		wcstombs_s(&converted, p, len, str.c_str(), len);
		std::string str1(p);
		delete[] p;
		return str1;
	}

	void BackupPE(LPCWSTR lpszPEFile)
	{
		m_strOldPE = lpszPEFile;
		// 备份一下，防止损坏需要修复
		m_strBakPE = m_strOldPE + L".bak";
		CopyFile(lpszPEFile, m_strBakPE.c_str(), TRUE);
	}

	void RestorePE()
	{
		if (PathFileExists(m_strBakPE.c_str()))
		{
			DeleteFile(m_strOldPE.c_str());
			CopyFile(m_strBakPE.c_str(), m_strOldPE.c_str(), FALSE);
		}
	}

	BOOL OpenPE(LPCWSTR lpszPEFile)
	{
		Clean();

		BOOL bRet = FALSE;

		do 
		{
			BackupPE(lpszPEFile);

			HANDLE hPEFile = CreateFile(
				lpszPEFile,
				GENERIC_READ | GENERIC_WRITE,
				FILE_SHARE_READ | FILE_SHARE_WRITE,
				NULL,
				OPEN_EXISTING,
				FILE_ATTRIBUTE_NORMAL,
				NULL
			);
			if (INVALID_HANDLE_VALUE == hPEFile || NULL == hPEFile)
			{
				break;
			}

			m_hPEFile = hPEFile;

			DWORD dwFileLength = GetFileSize(hPEFile, NULL);
			HANDLE hPEMapping = CreateFileMapping(
				hPEFile, 
				NULL, 
				PAGE_READWRITE, 
				0, 0, 0);
			if (NULL == hPEMapping)
			{
				break;
			}

			m_hPEMapping = hPEMapping;

			LPVOID lpData = MapViewOfFile(hPEMapping, FILE_MAP_ALL_ACCESS, 0, 0, dwFileLength);
			if (NULL == lpData)
			{
				break;
			}

			m_pPEImageBase = (LPBYTE)lpData;

			bRet = TRUE;

		} while (FALSE);

		return bRet;
	}

	BOOL IsPE()
	{
		BOOL bRet = FALSE;

		do 
		{
			if (NULL == m_pPEImageBase)
			{
				break;
			}

			PIMAGE_DOS_HEADER pDosHeader = (PIMAGE_DOS_HEADER)m_pPEImageBase;
			if (pDosHeader->e_magic != IMAGE_DOS_SIGNATURE)
			{
				break;
			}

			PIMAGE_NT_HEADERS pNtHeader = (PIMAGE_NT_HEADERS)(m_pPEImageBase + pDosHeader->e_lfanew);
			if (pNtHeader->Signature != IMAGE_NT_SIGNATURE)
			{
				break;
			}

			bRet = TRUE;

		} while (FALSE);
		
		return bRet;
	}

	BOOL Add(const char* szSectionName, const char* szDll, const char* szFunctionName)
	{
		BOOL bRet = FALSE;

		do 
		{
			PIMAGE_DOS_HEADER pDosHeader = (PIMAGE_DOS_HEADER)m_pPEImageBase;
			PIMAGE_NT_HEADERS pNtHeader = (PIMAGE_NT_HEADERS)(m_pPEImageBase + pDosHeader->e_lfanew);

			PIMAGE_SECTION_HEADER pNewSection = NULL;
			DWORD rSize, rOffset, vSize, vOffset;
			{
				// Step1：先判断一下，是否可以增加新Section
				if (((pNtHeader->FileHeader.NumberOfSections + 1) * sizeof(IMAGE_SECTION_HEADER)) > (pNtHeader->OptionalHeader.SizeOfHeaders))
				{
					break;
				}

				// Step2：定位到Section Table（在NT头尾部），并找到最后一个Section Header，作为新Section
				pNewSection = (PIMAGE_SECTION_HEADER)(pNtHeader + 1) + pNtHeader->FileHeader.NumberOfSections;
				PIMAGE_SECTION_HEADER pLastSection = pNewSection - 1;

				// Step3：新Section大小只需要256即可。新Section 放到上一个Section的后面，需要对齐偏移和RVA
				DWORD dwNewSectionSize = 256;
				rSize = PEAlign(dwNewSectionSize, pNtHeader->OptionalHeader.FileAlignment);
				rOffset = PEAlign(pLastSection->PointerToRawData + pLastSection->SizeOfRawData, pNtHeader->OptionalHeader.FileAlignment);
				vSize = PEAlign(dwNewSectionSize, pNtHeader->OptionalHeader.SectionAlignment);
				vOffset = PEAlign(pLastSection->VirtualAddress + pLastSection->Misc.VirtualSize, pNtHeader->OptionalHeader.SectionAlignment);

				// Step4：计算好的RVA用来填充Section Header。注意，Section Name最多只有8字节可存储，字符串长度不能超过7
				memcpy(pNewSection->Name, szSectionName, min(strlen(szSectionName) + 1, 8));
				pNewSection->VirtualAddress = vOffset;
				pNewSection->PointerToRawData = rOffset;
				pNewSection->Misc.VirtualSize = vSize;
				pNewSection->SizeOfRawData = rSize;
				pNewSection->Characteristics = IMAGE_SCN_MEM_READ | IMAGE_SCN_MEM_WRITE;

				// Step5：修改IMAGE_NT_HEADERS，增加新Section Table
				pNtHeader->FileHeader.NumberOfSections++;
				pNtHeader->OptionalHeader.SizeOfImage += vSize;
				pNtHeader->OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_BOUND_IMPORT].Size = 0;
				pNtHeader->OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_BOUND_IMPORT].VirtualAddress = 0;
			}

			// Step6：构造新Section
			PBYTE pbNewSection = new BYTE[rSize];
			memset(pbNewSection, 0, rSize);
			int nImportCnt = 0;
			{
				PBYTE pbNewSectionContent = pbNewSection;

				// Step6.1：通过新Section重定向导入表，需要将老的内容都拷贝到新Section中
				PIMAGE_IMPORT_DESCRIPTOR pImportTable = (PIMAGE_IMPORT_DESCRIPTOR)(m_pPEImageBase + RVAToOffset(pNtHeader, pNtHeader->OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_IMPORT].VirtualAddress));
				BOOL bBoundImport = FALSE;
				if (pImportTable->Characteristics == 0 && pImportTable->FirstThunk != 0)
				{
					bBoundImport = TRUE;
					pNtHeader->OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_BOUND_IMPORT].Size = 0;
					pNtHeader->OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_BOUND_IMPORT].VirtualAddress = 0;
				}

				while (pImportTable->FirstThunk != 0)
				{
					memcpy(pbNewSectionContent, pImportTable, sizeof(IMAGE_IMPORT_DESCRIPTOR));
					pImportTable++;
					pbNewSectionContent += sizeof(IMAGE_IMPORT_DESCRIPTOR);
					nImportCnt++;
				}

				memcpy(pbNewSectionContent, (pbNewSectionContent - sizeof(IMAGE_IMPORT_DESCRIPTOR)), sizeof(IMAGE_IMPORT_DESCRIPTOR));

				DWORD dwDelt = pNewSection->VirtualAddress - pNewSection->PointerToRawData;

				// Step6.2：填充ThunkData，包括dll名字、偏移
				PIMAGE_THUNK_DATA pImgThunkData = (PIMAGE_THUNK_DATA)(pbNewSectionContent + sizeof(IMAGE_IMPORT_DESCRIPTOR) * 2);

				// 导入dll的名字
				PBYTE pszDllNamePosition = (PBYTE)(pImgThunkData + 2);
				memcpy(pszDllNamePosition, szDll, strlen(szDll));
				pszDllNamePosition[strlen(szDll)] = 0;

				// dll的导入函数
				PIMAGE_IMPORT_BY_NAME pImgImportByName = (PIMAGE_IMPORT_BY_NAME)(pszDllNamePosition + strlen(szDll) + 1);
				pImgThunkData->u1.Ordinal = dwDelt + (DWORD)pImgImportByName - (DWORD)pbNewSection + rOffset;

				pImgImportByName->Hint = 1;
				memcpy(pImgImportByName->Name, szFunctionName, strlen(szFunctionName)); //== dwDelt + (DWORD)pszFuncNamePosition - (DWORD)lpData ;
				pImgImportByName->Name[strlen(szFunctionName)] = 0;

				// 导入项的基本描述
				PIMAGE_IMPORT_DESCRIPTOR pImgIportDesc = (PIMAGE_IMPORT_DESCRIPTOR)pbNewSectionContent;
				if (bBoundImport)
				{
					pImgIportDesc->OriginalFirstThunk = 0;
				}
				else
				{
					pImgIportDesc->OriginalFirstThunk = dwDelt + (DWORD)pImgThunkData - (DWORD)pbNewSection + rOffset;
				}
				pImgIportDesc->FirstThunk = dwDelt + (DWORD)pImgThunkData - (DWORD)pbNewSection + rOffset;
				pImgIportDesc->Name = dwDelt + (DWORD)pszDllNamePosition - (DWORD)pbNewSection + rOffset;

			}
			
			// Step7：改变导入表位置
			pNtHeader->OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_IMPORT].VirtualAddress = pNewSection->VirtualAddress;
			pNtHeader->OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_IMPORT].Size = (nImportCnt + 1) * sizeof(IMAGE_IMPORT_DESCRIPTOR);

			// Step8：将新Section写入文件结尾。注意，需要先取消映射
			UnmapViewOfFile(m_hPEMapping);
			m_hPEMapping = NULL;
			DWORD dwWriteBytes;
			SetFilePointer(m_hPEFile, 0, 0, FILE_END);
			if (!WriteFile(m_hPEFile, pbNewSection, rSize, &dwWriteBytes, NULL))
			{
				delete []pbNewSection;
				break;
			}
			delete[]pbNewSection;

			bRet = TRUE;

		} while (FALSE);
		
		return bRet;
	}

	void Clean()
	{
		if (m_hPEFile)
		{
			CloseHandle(m_hPEFile);
			m_hPEFile = NULL;
		}
		if (m_pPEImageBase)
		{
			UnmapViewOfFile(m_pPEImageBase);
			m_pPEImageBase = NULL;
		}
		if (m_hPEMapping)
		{
			CloseHandle(m_hPEMapping);
			m_hPEMapping = NULL;
		}
	}

	DWORD PEAlign(DWORD dwTarNumber, DWORD dwAlignTo)
	{
		return(((dwTarNumber + dwAlignTo - 1) / dwAlignTo)*dwAlignTo);
	}

	DWORD RVAToOffset(PIMAGE_NT_HEADERS pImageNTHeader, DWORD dwRVA)
	{
		DWORD _offset;
		PIMAGE_SECTION_HEADER section;
		section = ImageRVAToSection(pImageNTHeader, dwRVA);
		if (section == NULL)
		{
			return(0);
		}
		_offset = dwRVA + section->PointerToRawData - section->VirtualAddress;
		return(_offset);
	}

	PIMAGE_SECTION_HEADER ImageRVAToSection(PIMAGE_NT_HEADERS pImageNTHeader, DWORD dwRVA)
	{
		int i;
		PIMAGE_SECTION_HEADER pSectionHeader = (PIMAGE_SECTION_HEADER)(pImageNTHeader + 1);
		for (i = 0; i < pImageNTHeader->FileHeader.NumberOfSections; i++)
		{
			if ((dwRVA >= (pSectionHeader + i)->VirtualAddress) && (dwRVA <= ((pSectionHeader + i)->VirtualAddress + (pSectionHeader + i)->SizeOfRawData)))
			{
				return ((PIMAGE_SECTION_HEADER)(pSectionHeader + i));
			}
		}
		return(NULL);
	}

private:
	HANDLE	m_hPEFile;
	HANDLE	m_hPEMapping;
	LPBYTE	m_pPEImageBase;
	std::wstring	m_strOldPE;
	std::wstring	m_strBakPE;
};
```

### 1.2.6. 其他注入技术

前面介绍的，都是应用层的注入方法，当然还有一种注册表注入的方法（HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Windows\AppInit_DLLs），当User32.dll被加载到用户进程时，User32.dll的DLL_PROCESS_ATTACH处理过程会去读取上述注册表键值，查询dll，进行加载。  

当然，如果是内核层，则无所谓注入，因为内核空间本身就在任意一个进程空间中。

# 2. 函数Hook

## 2.1. 修改函数内存代码

要修改函数内存代码来达到Hook的目的，首先需要了解函数执行的基本原理：
```C++
	Test(3);
011D61A2 6A 03                push        3  // 首先压入参数到栈中
011D61A4 E8 2A B3 FF FF       call        Test (011D14D3h)  
// E8是call指令二进制编码，2A B3 FF FF是负数，须转换成补码4CD6（FFFF-B32A+1），它的计算方式是：目标地址-下一跳eip地址（011D14D3-011D61A9）。call指令会将下一跳eip地址存储到栈上，然后再进行跳转。查看内存如下：
// ESP = 010FF6D4 EBP = 010FF808
// esp          函数返回地址   参数  
// 0x010FF6D4:  011d61a9 00000003

011D61A9 83 C4 04             add         esp,4  

Test:
011D14D3 E9 88 08 00 00       jmp         Test (011D1D60h)  // E9是jmp指令二进制编码，88 08 00 00应该反过来，相对偏移是00 00 08 88，计算方式是：目标地址-下一跳eip地址（011D1D60-011D14D8=0000888）
011D14D8 CC                   int         3 

void Test(int a)
{
011D1D60  push        ebp  // 上一跳的ebp入栈
011D1D61  mov         ebp,esp  
011D1D63  sub         esp,0CCh  
011D1D69  push        ebx  
011D1D6A  push        esi  
011D1D6B  push        edi  
011D1D6C  lea         edi,[ebp-0CCh]  
011D1D72  mov         ecx,33h  
011D1D77  mov         eax,0CCCCCCCCh  
011D1D7C  rep stos    dword ptr es:[edi]  
011D1D7E  mov         ecx,offset _DE6249E3_injectapctest@cpp (011DD019h)  
011D1D83  call        @__CheckForDebuggerJustMyCode@4 (011D12E4h)  
	int b = a + 1;
011D1D88  mov         eax,dword ptr [a]  
011D1D8B  add         eax,1  
011D1D8E  mov         dword ptr [b],eax  
}
011D1D91  pop         edi  
011D1D92  pop         esi  
011D1D93  pop         ebx  
011D1D94  add         esp,0CCh  
011D1D9A  cmp         ebp,esp  
011D1D9C  call        __RTC_CheckEsp (011D12F3h)  
011D1DA1  mov         esp,ebp  
011D1DA3  pop         ebp  
011D1DA4  ret
```

也就是说，函数调用的入栈顺序是：参数、函数返回地址、上一跳的ebp  

函数Hook基本原理：
- 在内存中对要拦截的函数进行定位，得到内存地址
- 保存函数起始汇编指令
- 用一条JMP指令（E9 + 相对eip偏移，x86和x64下需要占用5个字节，其他CPU指令集就不知道了）覆盖掉起始地址，替换成我们自己的函数虚拟地址（前提是注入到目标进程）
- 执行我们的函数后，需要取出之前保存的汇编指令继续执行，返回原始调用路径

引申：
- 也就是说，要覆盖jmp指令，需要明确覆盖多少字节，这就依赖jmp指令本身需要占用多少字节，而这个在不同体系架构下可能不一样

```C++

```

## 2.2. 修改模块的导入表
