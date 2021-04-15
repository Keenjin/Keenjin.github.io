---
layout: post
title: Windows API
date: 2019-02-12
tags: Windows
---

<!-- TOC -->

- [1. 进程模块相关](#1-进程模块相关)
    - [1.1. 获取当前调用栈模块](#11-获取当前调用栈模块)
    - [1.2. 获取模块签名](#12-获取模块签名)
    - [1.3. 创建进程dump](#13-创建进程dump)
    - [1.4. 判断系统和进程是32位还是64位](#14-判断系统和进程是32位还是64位)
- [2. 格式转换](#2-格式转换)
    - [2.1. 宽窄字符串互转](#21-宽窄字符串互转)
- [3. 服务相关](#3-服务相关)
    - [3.1. 服务进程阻断调试](#31-服务进程阻断调试)
    - [3.2. 服务切换身份](#32-服务切换身份)

<!-- /TOC -->

# 1. 进程模块相关

## 1.1. 获取当前调用栈模块

```C++
std::wstring GetCallStackModule()
{
	std::wstring strModuleName;

	//void* Address = NULL;
	//_asm {
	//	mov eax,[ebp + 4]
	//	mov Address,eax
	//}
	void * Address = _ReturnAddress();
	HMODULE hModule = NULL;
	if (GetModuleHandleExW(GET_MODULE_HANDLE_EX_FLAG_FROM_ADDRESS, (LPCWSTR)Address, &hModule) && hModule)
	{
		TCHAR szModuleName[MAX_PATH + 1] = { 0 };
		GetModuleFileNameW(hModule, szModuleName, MAX_PATH);
		strModuleName = szModuleName;
	}

	return strModuleName;
}
```

## 1.2. 获取模块签名

```C++
#include <wincrypt.h>
#pragma  comment(lib, "crypt32.lib")

LONG GetSoftSign(WCHAR* v_pszFilePath, char * v_pszSign, int v_iBufSize)
{
	//首先判断参数是否正确
	if (v_pszFilePath == NULL) return -1;

	HCERTSTORE		  hStore = NULL;
	HCRYPTMSG		  hMsg = NULL;
	PCCERT_CONTEXT    pCertContext = NULL;
	BOOL			  bResult;
	DWORD dwEncoding, dwContentType, dwFormatType;
	PCMSG_SIGNER_INFO pSignerInfo = NULL;
	PCMSG_SIGNER_INFO pCounterSignerInfo = NULL;
	DWORD			  dwSignerInfo;
	CERT_INFO		  CertInfo;
	SYSTEMTIME        st;
	LONG              lRet;
	DWORD             dwDataSize = 0;

	char   chTemp[MAX_PATH] = { 0 };
	do
	{
		//从签名文件中获取存储句柄
		bResult = CryptQueryObject(
			CERT_QUERY_OBJECT_FILE,
			v_pszFilePath,
			CERT_QUERY_CONTENT_FLAG_PKCS7_SIGNED_EMBED,
			CERT_QUERY_FORMAT_FLAG_BINARY,
			0,
			&dwEncoding,
			&dwContentType,
			&dwFormatType,
			&hStore,
			&hMsg,
			NULL
		);

		if (!bResult){
			lRet = -1;
			break;
		}


		//获取签名信息所需的缓冲区大小
		bResult = CryptMsgGetParam(
			hMsg,
			CMSG_SIGNER_INFO_PARAM,
			0,
			NULL,
			&dwSignerInfo
		);
		if (!bResult){
			lRet = -1;
			break;
		}

		//分配缓冲区
		pSignerInfo = (PCMSG_SIGNER_INFO)LocalAlloc(LPTR, dwSignerInfo);
		if (pSignerInfo == NULL){
			lRet = -1;
			break;
		}


		//获取签名信息
		bResult = CryptMsgGetParam(
			hMsg,
			CMSG_SIGNER_INFO_PARAM,
			0,
			pSignerInfo,
			&dwSignerInfo
		);
		if (!bResult){
			lRet = -1;
			break;
		}

		CertInfo.Issuer = pSignerInfo->Issuer;
		CertInfo.SerialNumber = pSignerInfo->SerialNumber;

		pCertContext = CertFindCertificateInStore(
			hStore,
			dwEncoding,
			0,
			CERT_FIND_SUBJECT_CERT,
			(PVOID)&CertInfo,
			NULL
		);
		if (pCertContext == NULL){
			lRet = -1;
			break;
		}


		//获取数字键名
		//没有给定缓冲区，那么说明只要获取下需要的长度
		if (v_pszSign == NULL)
		{
			dwDataSize = CertGetNameString(
				pCertContext,
				CERT_NAME_SIMPLE_DISPLAY_TYPE,
				0,
				NULL,
				NULL,
				0
			);
			if (dwDataSize != 0){
				lRet = dwDataSize;
			}
			else{
				lRet = -1;
			}

			break;
		}

		if (!(CertGetNameString(
			pCertContext,
			CERT_NAME_SIMPLE_DISPLAY_TYPE,
			0,
			NULL,
			v_pszSign,
			v_iBufSize))){

			lRet = -1;
			break;
		}

		lRet = 0;

	} while (FALSE);

	if (pSignerInfo != NULL){
		LocalFree((HLOCAL)pSignerInfo);
	}

	return lRet;
}
```

## 1.3. 创建进程dump

```C++
bool CDumpHelper::CreateDump(UINT uPID, LPCWSTR lpszFile, MINIDUMP_TYPE type/* = MINIDUMP_TYPE::MiniDumpNormal*/)
{
	bool bRet = false;
	DWORD dwErr = 0;
	HANDLE hFile = NULL;

	do
	{
		HMODULE hModule = GetModule();
		if (!hModule)
		{
			break;
		}

		HANDLE hFile = ::CreateFile(lpszFile, GENERIC_WRITE, FILE_SHARE_READ, NULL, CREATE_ALWAYS, 0, NULL);
		if (hFile == INVALID_HANDLE_VALUE || hFile == NULL)
		{
			break;
		}

		FARPROC miniDumpWriteDump = ::GetProcAddress(hModule, "MiniDumpWriteDump");
		if (miniDumpWriteDump == NULL)
		{
			break;
		}

		typedef BOOL(__stdcall *PROC_MiniDumpWriteDump)(HANDLE, DWORD, HANDLE, MINIDUMP_TYPE, PMINIDUMP_EXCEPTION_INFORMATION, PMINIDUMP_USER_STREAM_INFORMATION, PMINIDUMP_CALLBACK_INFORMATION);
		PROC_MiniDumpWriteDump pfnMiniDumpWriteDump = (PROC_MiniDumpWriteDump)miniDumpWriteDump;

		DWORD dwDesiredAccess = PROCESS_QUERY_INFORMATION | PROCESS_VM_READ | THREAD_ALL_ACCESS;
		if (type & MiniDumpWithHandleData)
		{
			dwDesiredAccess |= PROCESS_DUP_HANDLE;
		}
		HANDLE hProcess = OpenProcess(dwDesiredAccess, FALSE, uPID);
		if (INVALID_HANDLE_VALUE == hProcess)
		{
			break;
		}

		bRet = pfnMiniDumpWriteDump(hProcess, uPID, hFile, type, NULL, NULL, NULL);
		dwErr = GetLastError();
		if (hProcess)
		{
			CloseHandle(hProcess);
		}

	} while (false);

	if (hFile)
	{
		CloseHandle(hFile);
		hFile = NULL;
	}
	
	return bRet;
}

HMODULE CDumpHelper::GetModule()
{
	if (m_hModule)
	{
		return m_hModule;
	}

	TCHAR szDbgDllPath[MAX_PATH] = { 0 };
	GetModuleFileName(NULL, szDbgDllPath, MAX_PATH);
	PathRemoveFileSpec(szDbgDllPath);
	lstrcat(szDbgDllPath, L"\\dbghelp.dll");
	HMODULE hModule = ::LoadLibrary(szDbgDllPath);
	if (NULL == hModule)
	{
		WCHAR szFilePath[MAX_PATH] = { 0 };
		DWORD dwLen = GetSystemDirectoryW(szFilePath, MAX_PATH);
		if (dwLen == 0 || dwLen >= MAX_PATH - 16)
			return NULL;

		if (szFilePath[dwLen - 1] != L'\\')
		{
			szFilePath[dwLen++] = L'\\';
		}

		lstrcat(szFilePath, L"\\dbghelp.dll");
		hModule = ::LoadLibrary(szFilePath);
	}

	m_hModule = hModule;
	return m_hModule;
}

void Test()
{
	DWORD dwPID = 0;
	HWND hWnd = (HWND)0x9906f8;
	GetWindowThreadProcessId(hWnd, &dwPID);
	std::wstring strFilePath = L"D:\\test.dmp";
	MINIDUMP_TYPE type = (MINIDUMP_TYPE)(
		MiniDumpWithDataSegs |
		MiniDumpWithHandleData |
		/*MiniDumpWithModuleHeaders |*/
		/*MiniDumpWithThreadInfo |*/
		/*MiniDumpWithFullMemory |*/
		MiniDumpWithPrivateReadWriteMemory |
		MiniDumpWithUnloadedModules);
	
	CDumpHelper::Instance().CreateDump(dwPID, strFilePath.c_str(), type);
}

```

## 1.4. 判断系统和进程是32位还是64位

```C++
static bool is_64bit_windows(void)
{
#ifdef _WIN64
	return true;
#else
	BOOL x86 = false;
	bool success = !!IsWow64Process(GetCurrentProcess(), &x86);
	return success && !!x86;
#endif
}

static bool is_64bit_process(HANDLE process)
{
	BOOL x86 = true;
	if (is_64bit_windows()) {
		bool success = !!IsWow64Process(process, &x86);
		if (!success) {
			return false;
		}
	}

	return !x86;
}
```

# 2. 格式转换

## 2.1. 宽窄字符串互转

```C++
// string转wstring
std::wstring StringToWString(const std::string& str)
{
	unsigned len = str.size() * 2;// 预留字节数
	setlocale(LC_CTYPE, "");     //必须调用此函数
	wchar_t *p = new wchar_t[len];// 申请一段内存存放转换后的字符串
	mbstowcs(p, str.c_str(), len);// 转换
	std::wstring str1(p);
	delete[] p;// 释放申请的内存
	return str1;
}

// wstring转string
std::string WStringToString(const std::wstring& str)
{
	unsigned len = str.size() * 4;
	setlocale(LC_CTYPE, "");
	char *p = new char[len];
	wcstombs(p, str.c_str(), len);
	std::string str1(p);
	delete[] p;
	return str1;
}
```

# 3. 服务相关

## 3.1. 服务进程阻断调试

```c++
void ServiceMsgBox(const std::wstring& caption, const std::wstring& text, DWORD dwType = MB_OK|MB_TOPMOST) {
	DWORD dwResponse = 0;

	wchar_t* pCaption = new wchar_t[caption.size() + 1];
	wcscpy_s(pCaption, caption.size() * 2, caption.c_str());
	pCaption[caption.size()] = L'\0';

	wchar_t* pText = new wchar_t[text.size() + 1];
	wcscpy_s(pText, text.size() * 2, text.c_str());
	pText[text.size()] = L'\0';

	::WTSSendMessage(
		WTS_CURRENT_SERVER_HANDLE, 
		::WTSGetActiveConsoleSessionId(), 
		(LPWSTR)pText, 
		(text.size() + 1)*2,
		(LPWSTR)pCaption, 
		(caption.size() + 1)*2, 
		dwType,
		0,
		&dwResponse,
		TRUE);

	delete []pCaption;
	delete []pText;
}

```

## 3.2. 服务切换身份

```c++
class CAutoImpersonUser {
public:
	CAutoImpersonUser() {
		ImpersonUser();
	}

	~CAutoImpersonUser() {
		EndImperson();
	}

private:
	BOOL ImpersonUser()	{
		HANDLE hToken = NULL;
		BOOL bImperson = FALSE;
		
		do {
			BOOL bRet = WTSQueryUserToken(WTSGetActiveConsoleSessionId(), &hToken);
			if (!bRet || NULL == hToken)
				break;
			
			bImperson = ImpersonateLoggedOnUser(hToken);

		} while (false);

		if (NULL != hToken)	{
			CloseHandle(hToken);
			hToken = NULL;
		}

		return bImperson;
	}

	void EndImperson() {
		RevertToSelf();
	}
}
```