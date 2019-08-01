---
layout: post
title: 对Go的理解
date: 2019-07-31
tags: go
---

<!-- TOC -->

- [1. Go语言优势和特点](#1-go语言优势和特点)
    - [1.1. 与Python对比](#11-与python对比)
    - [1.2. 与C/C++对比](#12-与cc对比)
    - [1.3. 与Java对比](#13-与java对比)
- [2. Go的优势是如何实现的](#2-go的优势是如何实现的)
    - [2.1. 语法设计层面](#21-语法设计层面)
    - [2.2. rumtime](#22-rumtime)
        - [2.2.1. 协程调度机制](#221-协程调度机制)
        - [2.2.2. 内存管理机制](#222-内存管理机制)
        - [2.2.3. GC机制](#223-gc机制)

<!-- /TOC -->

# 1. Go语言优势和特点

## 1.1. 与Python对比

- Go部署简单，整个编译生成一个静态的可执行文件，除了glibc外没有其他依赖，一个进程文件，也比较好做监控和服务发现等；Python需要维护各种库、包依赖关系，部署的时候相当麻烦，不同版本的源不兼容问题会导致需要私建镜像
- Go并发性好，单个Go应用也能有效利用多个CPU核；Python的并发性是其本源劣势之一，脚本执行的全局所GIL导致Python的多线程无法有效利用多核，只能用多进程的方式，这样对进程的监控管理又有极大问题
- Go的源码，相比Python更容易理解，没有Python那般灵活的语法特性，无类型的特性，代码管理也是编译器级别就支持的，比Python更规范

## 1.2. 与C/C++对比

- Go的学习难度相比C/C++简单太多
- Go的原生库就支持几乎开发用到的所有必要组件，包括http、json、rpc等，开发难度很低
- Go原生语法级别支持的Goroutine使用协程，相比C/C++的线程，更容易使用，channel通信方式在多协程之间也是无锁的方式，相比C/C++的线程同步简单太多
- Go天然支持GC，无需程序员管理内存，无需关心内存是堆还是栈，解决了C/C++最容易出错的点，程序相比而言更容易稳定
- Go支持类型反射，使用Any类型（interface{}）使得开发灵活性接近解析型语言
- Go天然支持pprof、trace等性能分析工具，极方便的解决性能瓶颈；C/C++得通过第三方性能分析工具+pdb来一一分析

## 1.3. 与Java对比

- Go在语言级别上，就支持高性能的http/tcp/udp的server，极容易编写的协程，极容易部署，天生适合后台服务
- 相比Java小巧轻量，占用内存小
- Go直接编译成机器码运行，而非Java利用虚拟机解释执行

# 2. Go的优势是如何实现的

## 2.1. 语法设计层面

- 关键字少，控制在几个核心关键字内，去除了C++里面一些不重要的关键字
- 语言设计层面强调并发编程，直接将协程作为语法层提供支持，go和channel
- 标准库囊括了业务开发的几个必要的组件，http、rpc、json、net等
- 编译器将代码的规范作为严格标准

## 2.2. rumtime

### 2.2.1. 协程调度机制

Windows下实现的协程

```c
// coroutine.h
#pragma once

#define COROUTINE_DEAD     0
#define COROUTINE_READY    1
#define COROUTINE_RUNNING  2
#define COROUTINE_SUSPEND  3

typedef struct schedule schedule;
typedef void(*coroutine_func)(schedule *s, void *ud);

schedule *coroutine_open();
void coroutine_close(schedule *s);
int coroutine_new(schedule *s, coroutine_func *, void *ud);
void coroutine_resume(schedule *s, int id);
void coroutine_yield(schedule *s);
int coroutine_status(schedule *s, int id);
int coroutine_running(schedule *s);

```

```c
// coroutine.c
#include "stdafx.h"
#include <stdio.h>
#include <stdlib.h>
#include <windows.h>
#include <assert.h>
#include "coroutine.h"

/* windows fiber版本的协程yield的时候直接切换到主协程(main)，
而不是swapcontext的切换到上次运行协程,但最后达到的结果却一样
*/

// 默认容量
#define DEFAULT_CAP   8
// 堆栈大小
#define INIT_STACK    1048576 //(1024*1024)

typedef struct schedule schedule;
typedef struct coroutine coroutine;
typedef struct coroutine_para coroutine_para;

struct schedule
{
	int  cap;     // 容量
	int  conums;
	int  curID;   // 当前协程ID
	LPVOID    main;
	coroutine **co;
};

struct coroutine
{
	schedule  *s;
	void      *ud;
	int       status;
	LPVOID    ctx;
	coroutine_func func;
};

static int co_putin(schedule *s, coroutine *co)
{
	if (s->conums >= s->cap)
	{
		int id = s->cap;
		s->co = realloc(s->co, sizeof(coroutine *) * s->cap * 2);
		memset(&s->co[s->cap], 0, sizeof(coroutine *) * s->cap);
		s->co[s->cap] = co;
		s->cap *= 2;
		++s->conums;
		return id;
	}
	else
	{
		for (int i = 0; i < s->cap; i++)
		{
			int id = (i + s->conums) % s->cap;
			if (s->co[id] == NULL)
			{
				s->co[id] = co;
				++s->conums;
				return id;
			}
		}
	}
	assert(0);
	return -1;
}

static void co_delete(coroutine *co)
{
	//If the currently running fiber calls DeleteFiber, its thread calls ExitThread and terminates.
	//However, if a currently running fiber is deleted by another fiber, the thread running the 
	//deleted fiber is likely to terminate abnormally because the fiber stack has been freed.
	DeleteFiber(co->ctx);
	free(co);
}

schedule *coroutine_open()
{
	schedule *s = malloc(sizeof(schedule));
	s->cap = DEFAULT_CAP;
	s->conums = 0;
	s->curID = -1;
	s->co = malloc(sizeof(coroutine *) * s->cap);
	memset(s->co, 0, sizeof(coroutine *) * s->cap);
	s->main = ConvertThreadToFiberEx(NULL, FIBER_FLAG_FLOAT_SWITCH);
	return s;
}

void coroutine_close(schedule *s)
{
	for (int i = 0; i < s->cap; i++)
	{
		coroutine *co = s->co[i];
		if (co) co_delete(co);
	}
	free(s->co);
	s->co = NULL;
	free(s);
}

void __stdcall coroutine_main(LPVOID lpParameter)
{
	schedule* s = (schedule*)lpParameter;
	int id = s->curID;
	coroutine *co = s->co[id];

	(co->func)(s, co->ud);

	s->curID = -1;
	--s->conums;
	s->co[id] = NULL;
	//co_delete(co);

	SwitchToFiber(s->main);
}

int coroutine_new(schedule *s, coroutine_func *func, void *ud)
{
	coroutine *co = malloc(sizeof(coroutine));
	co->s = s;
	co->status = COROUTINE_READY;
	co->func = func;
	co->ud = ud;
	int id = co_putin(s, co);
	co->ctx = CreateFiberEx(INIT_STACK, 0, FIBER_FLAG_FLOAT_SWITCH, coroutine_main, s);
	co->status = COROUTINE_READY;

	return id;
}

void coroutine_resume(schedule *s, int id)
{
	assert(id >= 0 && id < s->cap);
	if (id < 0 || id >= s->cap) return;
	coroutine *co = s->co[id];
	if (co == NULL) return;
	switch (co->status)
	{
	case COROUTINE_READY:case COROUTINE_SUSPEND:
		co->status = COROUTINE_RUNNING;
		s->curID = id;
		SwitchToFiber(co->ctx);
		if (!s->co[id]) co_delete(co);
		break;
	default:
		assert(0);
		break;
	}
}

void coroutine_yield(schedule *s)
{
	int id = s->curID;
	assert(id >= 0 && id < s->cap);
	if (id < 0) return;

	coroutine *co = s->co[id];
	co->status = COROUTINE_SUSPEND;
	s->curID = -1;

	SwitchToFiber(s->main);
}

int coroutine_status(schedule *s, int id)
{
	assert(id >= 0 && id < s->cap);
	if (id < 0) return;
	if (s->co[id] == NULL) {
		return COROUTINE_DEAD;
	}
	return s->co[id]->status;
}

int coroutine_running(schedule *s)
{
	return s->curID;

}
```

```c
// main.c
// ConsoleApplication1.cpp : Defines the entry point for the console application.
//

#include "stdafx.h"


#include "coroutine.h"

void test3(schedule *s, void *ud)
{
	int *data = (int*)ud;
	for (int i = 0; i < 3; i++)
	{
		printf("test3 i=%d\n", i);
		coroutine_yield(s);
		printf("yield co id = %d.\n", *data);
	}
}

void coroutine_test()
{
	printf("coroutine_test3 begin\n");
	schedule *s = coroutine_open();

	int a = 11;
	int id1 = coroutine_new(s, test3, &a);
	int id2 = coroutine_new(s, test3, &a);

	while (coroutine_status(s, id1) && coroutine_status(s, id2))
	{
		printf("\nresume co id = %d.\n", id1);
		coroutine_resume(s, id1);
		//printf("resume co id = %d.\n", id2);
		//coroutine_resume(s, id2);
	}

	int id3 = coroutine_new(s, test3, &a);
	while (coroutine_status(s, id3))
	{
		printf("\nresume co id = %d.\n", id3);
		coroutine_resume(s, id3);
	}

	printf("coroutine_test3 end\n");
	coroutine_close(s);
}


int main()
{
	coroutine_test();
	return 0;

}
```

### 2.2.2. 内存管理机制

### 2.2.3. GC机制
