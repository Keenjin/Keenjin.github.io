---
layout: post
title: Win下的协程模拟
date: 2019-08-01
tags: win
---

![png](/images/post/go/win_coroutine.png)

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
    // 容量超过就扩容
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