---
layout: post
title: Linux下fork和exec的使用特性
date: 2019-05-21
tags: Linux
---

# fork

使用fork完，会创建一个子进程，内部处理逻辑，是让这俩进程共享代码段，同时，会新复制数据段和堆栈段（这里实际上没有做复制操作，只有在写新数据的时候，才会往不同地方写）  
调用完fork，返回pid，对于父进程而言，pid>0，是子进程的pid。对于子进程而言，pid=0  
调用完fork，对于父进程和子进程，会从下面一条语句开始执行。这里非常奇妙，相当于实时备份了一整个进程状态。  

# exec

一个进程一旦调用exec系列函数，它本身就相当于死亡了，系统会把代码段替换成新的程序的代码，废弃原有的数据段和堆栈段，并为新程序分配新的数据段和堆栈段。  
但是，exec完，进程pid不会变，复用了父进程的pid，整个过程相当于把父进程覆盖掉了。一旦exec执行成功，原来父进程后面的代码，相当于没有任何用了，不会再执行了，强制切换到了新进程中去从头开始执行代码。如果exec执行失败，还是会返回错误信息的。

# fork和exec组合使用

这俩组合使用，可以完美的实现管道通信，代码如下：  
这个代码里，成功将ls -l的执行内容，输出到wc命令的输入端

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main()
{
    int pfds[2];
    if ( pipe(pfds) == 0 ) {
        if ( fork() == 0 ) {
            close(1);
            dup2( pfds[1], 1 );
            close( pfds[0] );
            execlp( "ls", "ls", "-1", NULL );
        } else {
            close(0);
            dup2( pfds[0], 0 );
            close( pfds[1] );
            execlp( "wc", "wc", "-l", NULL );
        }

    }
    return 0;
}

```

```c++
int myshell::create()
{
    int nReadFds[2] = {-1, -1};
    int nWriteFds[2] = {-1, -1};
    do
    {
        if(pipe(nReadFds) < 0)
        {
            break;
        }

        if(pipe(nWriteFds) < 0)
        {
            break;
        }

        pid_t pid = -1;
        if((pid = fork()) < 0)
        {
            break;
        }
        else if(pid > 0)
        {// 父亲
            m_hRead = nReadFds[0];
            nReadFds[0] = -1;
            m_hWrite = nWriteFds[1];
            nWriteFds[1] = -1;
            m_pidTav = pid;
        }
        else
        {// 孩子
            close(nReadFds[0]);
            if(nReadFds[1] != STDOUT_FILENO)
            {
                if(dup2(nReadFds[1], STDOUT_FILENO) != STDOUT_FILENO)
                {
                    exit(1);
                }
                close(nReadFds[1]);
            }
            close(nWriteFds[1]);
            if(nWriteFds[0] != STDIN_FILENO)
            {
                if(dup2(nWriteFds[0], STDIN_FILENO) != STDIN_FILENO)
                {
                    exit(1);
                }
                close(nWriteFds[0]);
            }

            execlp( "wget", "wget", "-1", NULL );
            exit(1);
        }

    } while(FALSE);

    if(nReadFds[0] != -1)
    {
        close(nReadFds[0]);
        nReadFds[0] = -1;
    }

    if(nReadFds[1] != -1)
    {
        close(nReadFds[1]);
        nReadFds[1] = -1;
    }

    if(nWriteFds[0] != -1)
    {
        close(nWriteFds[0]);
        nWriteFds[0] = -1;
    }

    if(nWriteFds[1] != -1)
    {
        close(nWriteFds[1]);
        nWriteFds[1] = -1;
    }

    return 0;
}

int myshell::do()
{
    // 往命令行塞入指令
    char* pszLine = "http://www.baidu.com"
    DWORD dwLen = strlen(pszLine);
    write(m_hWrite, pszLine, dwLen);
    write(m_hWrite, "\n", 1);

    // 读取接收到的html内容
    const dwNeedSize = 1024;
    char buf[dwNeedSize] = {0};
    DWORD dwReadSize = (DWORD)read(m_hRead, buf, dwNeedSize);

    ...
}
```