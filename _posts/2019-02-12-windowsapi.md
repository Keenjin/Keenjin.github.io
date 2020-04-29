---
layout: post
title: Windows API
date: 2019-02-12
tags: Windows
---

# 进程模块相关

## 获取当前调用栈模块

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