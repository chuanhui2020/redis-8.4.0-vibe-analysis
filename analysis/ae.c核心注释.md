# Redis ae.c 事件循环实现详解（中文注释版）

> 📘 **说明**：ae.c 是 Redis 事件循环的实现文件，包含了文件事件、定时事件的核心逻辑。这是 Redis 高性能的关键所在！

---

## 📚 目录

1. [文件概述](#文件概述)
2. [头文件和平台适配](#头文件和平台适配)
3. [核心函数详解](#核心函数详解)
   - [aeCreateEventLoop：创建事件循环](#aecreate事件循环)
   - [aeCreateFileEvent：注册文件事件](#注册文件事件)
   - [aeCreateTimeEvent：创建定时事件](#创建定时事件)
   - [aeProcessEvents：处理事件](#处理事件核心)
4. [内部实现细节](#内部实现细节)
5. [典型执行流程](#典型执行流程)

---

## 文件概述

`ae.c` 是 Redis 事件循环的完整实现。

**文件位置**：`src/ae.c`（512行）

**核心功能**：
1. 事件循环的创建和销毁
2. 文件事件的注册、删除、触发
3. 定时事件的创建、删除、执行
4. 事件处理的主循环

**依赖关系**：
```
ae.c (事件循环核心)
  ├─ ae.h (接口定义)
  ├─ ae_epoll.c (Linux: epoll 实现)
  ├─ ae_kqueue.c (BSD/Mac: kqueue 实现)
  ├─ ae_select.c (通用: select 实现)
  └─ ae_evport.c (Solaris: event ports 实现)
```

---

## 头文件和平台适配

```c
// 文件位置：ae.c:1-11
/* A simple event-driven programming library. Originally I wrote this code
 * for the Jim's event-loop (Jim is a Tcl interpreter) but later translated
 * it in form of a library for easy reuse.
 *
 * 【历史】
 * 这个库最初是为 Jim（Tcl 解释器）的事件循环写的，
 * 后来改写成独立的库，方便复用。
 *
 * Copyright (c) 2006-Present, Redis Ltd.
 * All rights reserved.
 */

#include "ae.h"
#include "anet.h"        /* 网络相关工具函数 */
#include "redisassert.h" /* assert 宏 */

#include <stdio.h>
#include <sys/time.h>
#include <sys/types.h>
#include <unistd.h>
#include <stdlib.h>
#include <poll.h>
#include <string.h>
#include <time.h>
#include <errno.h>

#include "zmalloc.h"  /* Redis 的内存分配器 */
#include "config.h"   /* 编译配置 */
```

### I/O 多路复用平台适配

```c
// 文件位置：ae.c:30-44

/*
 * 【平台适配】
 * 根据编译时的宏定义，选择最优的 I/O 多路复用实现
 *
 * 优先级（性能从高到低）：
 * 1. evport (Solaris event ports) - 最快
 * 2. epoll (Linux) - 非常快
 * 3. kqueue (BSD/Mac) - 非常快
 * 4. select (所有平台) - 较慢，但兼容性最好
 *
 * 为什么这样排序？
 * - evport/epoll/kqueue 都是 O(1) 复杂度
 * - select 是 O(n) 复杂度，且有 1024 fd 限制
 */

/* Include the best multiplexing layer supported by this system.
 * The following should be ordered by performances, descending. */
#ifdef HAVE_EVPORT
#include "ae_evport.c"   /* Solaris */
#else
    #ifdef HAVE_EPOLL
    #include "ae_epoll.c"   /* Linux */
    #else
        #ifdef HAVE_KQUEUE
        #include "ae_kqueue.c"   /* BSD/Mac */
        #else
        #include "ae_select.c"   /* 通用兜底方案 */
        #endif
    #endif
#endif
```

**每个 ae_*.c 文件都必须实现这些接口**：
- `aeApiCreate()`：初始化（创建 epoll fd 等）
- `aeApiFree()`：清理资源
- `aeApiAddEvent()`：添加事件（epoll_ctl ADD）
- `aeApiDelEvent()`：删除事件（epoll_ctl DEL）
- `aeApiPoll()`：等待事件（epoll_wait）
- `aeApiName()`：返回名称（"epoll"）
- `aeApiResize()`：调整大小

---

## 核心函数详解

### aeCreateEventLoop：创建事件循环

```c
// 文件位置：ae.c:46-81

#define INITIAL_EVENT 1024  /* events 数组的初始大小 */

/*
 * aeCreateEventLoop：创建事件循环
 *
 * 【作用】
 * 初始化事件循环的所有资源，包括：
 * 1. 分配 aeEventLoop 结构
 * 2. 分配 events 和 fired 数组
 * 3. 初始化底层 I/O 多路复用（epoll_create）
 * 4. 初始化所有字段
 *
 * 【参数】
 * - setsize：最大文件描述符数量（通常是 maxclients + CONFIG_FDSET_INCR）
 *
 * 【返回值】
 * - 成功：返回 aeEventLoop 指针
 * - 失败：返回 NULL
 *
 * 【使用场景】
 * Redis 启动时调用一次：
 * server.el = aeCreateEventLoop(server.maxclients + CONFIG_FDSET_INCR);
 */
aeEventLoop *aeCreateEventLoop(int setsize) {
    aeEventLoop *eventLoop;
    int i;

    /*
     * 初始化单调时钟（monotonic clock）
     * 防止系统时间被修改影响定时器
     */
    monotonicInit();    /* just in case the calling app didn't initialize */

    /* 1. 分配 eventLoop 结构 */
    if ((eventLoop = zmalloc(sizeof(*eventLoop))) == NULL) goto err;

    /*
     * 2. 分配 events 和 fired 数组
     *
     * nevents 初始值：min(setsize, 1024)
     * 为什么不直接分配 setsize 大小？
     * - 节省内存：大多数情况下不会用满所有 fd
     * - 动态扩容：当 fd >= nevents 时再扩容
     *
     * 例子：
     * 如果 setsize=10000，初始只分配 1024 个槽位
     * 只有当 fd >= 1024 时才扩容到更大
     */
    eventLoop->nevents = setsize < INITIAL_EVENT ? setsize : INITIAL_EVENT;
    eventLoop->events = zmalloc(sizeof(aeFileEvent)*eventLoop->nevents);
    eventLoop->fired = zmalloc(sizeof(aeFiredEvent)*eventLoop->nevents);
    if (eventLoop->events == NULL || eventLoop->fired == NULL) goto err;

    /* 3. 初始化字段 */
    eventLoop->setsize = setsize;         /* 最大 fd 数 */
    eventLoop->timeEventHead = NULL;      /* 定时事件链表为空 */
    eventLoop->timeEventNextId = 0;       /* 定时事件 ID 从 0 开始 */
    eventLoop->stop = 0;                  /* 不停止 */
    eventLoop->maxfd = -1;                /* 当前没有注册任何 fd */
    eventLoop->beforesleep = NULL;        /* 没有 beforesleep 回调 */
    eventLoop->aftersleep = NULL;         /* 没有 aftersleep 回调 */
    eventLoop->flags = 0;                 /* 标志位清零 */
    memset(eventLoop->privdata, 0, sizeof(eventLoop->privdata));

    /*
     * 4. 初始化底层 I/O 多路复用
     *
     * aeApiCreate() 的实现因平台而异：
     * - Linux：创建 epoll fd (epoll_create)
     * - BSD/Mac：创建 kqueue fd (kqueue)
     * - 其他：初始化 fd_set
     */
    if (aeApiCreate(eventLoop) == -1) goto err;

    /*
     * 5. 初始化 events 数组
     *
     * 所有 fd 的 mask 设置为 AE_NONE，表示未注册事件
     */
    /* Events with mask == AE_NONE are not set. So let's initialize the
     * vector with it. */
    for (i = 0; i < eventLoop->nevents; i++)
        eventLoop->events[i].mask = AE_NONE;

    return eventLoop;

err:
    /* 失败时清理资源 */
    if (eventLoop) {
        zfree(eventLoop->events);
        zfree(eventLoop->fired);
        zfree(eventLoop);
    }
    return NULL;
}
```

### aeGetSetSize / aeSetDontWait / aeResizeSetSize

```c
// 文件位置：ae.c:83-122

/*
 * aeGetSetSize：获取 setsize
 *
 * 【返回值】
 * 事件循环能处理的最大 fd 数量
 */
/* Return the current set size. */
int aeGetSetSize(aeEventLoop *eventLoop) {
    return eventLoop->setsize;
}

/*
 * aeSetDontWait：设置不等待标志
 *
 * 【作用】
 * 告诉事件处理：尽快完成处理，不要阻塞等待
 *
 * 【参数】
 * - noWait：1=不等待，0=正常等待
 *
 * 【实现】
 * 设置或清除 AE_DONT_WAIT 标志位
 *
 * 【使用场景】
 * 当有紧急任务需要处理时：
 * - SHUTDOWN 命令
 * - 需要快速响应的信号
 *
 * Note: it just means you turn on/off the global AE_DONT_WAIT.
 */
void aeSetDontWait(aeEventLoop *eventLoop, int noWait) {
    if (noWait)
        eventLoop->flags |= AE_DONT_WAIT;   /* 设置标志 */
    else
        eventLoop->flags &= ~AE_DONT_WAIT;  /* 清除标志 */
}

/*
 * aeResizeSetSize：调整 setsize
 *
 * 【作用】
 * 动态调整事件循环能处理的最大 fd 数量
 *
 * 【参数】
 * - setsize：新的最大 fd 数量
 *
 * 【返回值】
 * - AE_OK (0)：成功
 * - AE_ERR (-1)：失败
 *
 * 【失败条件】
 * 1. 新 setsize == 旧 setsize（不需要调整，直接返回 AE_OK）
 * 2. maxfd >= setsize（当前有 fd >= setsize 在使用，不能缩小）
 * 3. aeApiResize() 失败（底层 epoll/kqueue 调整失败）
 *
 * 【使用场景】
 * 动态调整最大客户端连接数：
 * CONFIG SET maxclients 20000
 *
 * If the requested set size is smaller than the current set size, but
 * there is already a file descriptor in use that is >= the requested
 * set size minus one, AE_ERR is returned and the operation is not
 * performed at all.
 *
 * Otherwise AE_OK is returned and the operation is successful.
 */
int aeResizeSetSize(aeEventLoop *eventLoop, int setsize) {
    /* 1. 如果 setsize 没变，直接返回 */
    if (setsize == eventLoop->setsize) return AE_OK;

    /* 2. 如果当前有 fd >= setsize 在使用，不能缩小 */
    if (eventLoop->maxfd >= setsize) return AE_ERR;

    /* 3. 调整底层 I/O 多路复用的大小 */
    if (aeApiResize(eventLoop,setsize) == -1) return AE_ERR;

    /* 4. 更新 setsize */
    eventLoop->setsize = setsize;

    /*
     * 5. 如果需要缩小 events 和 fired 数组
     *
     * 例子：
     * 旧 nevents=2048, 新 setsize=1024
     * 需要缩小数组到 1024
     *
     * If the current allocated space is larger than the requested size,
     * we need to shrink it to the requested size.
     */
    if (setsize < eventLoop->nevents) {
        eventLoop->events = zrealloc(eventLoop->events,sizeof(aeFileEvent)*setsize);
        eventLoop->fired = zrealloc(eventLoop->fired,sizeof(aeFiredEvent)*setsize);
        eventLoop->nevents = setsize;
    }
    return AE_OK;
}
```

### aeDeleteEventLoop：销毁事件循环

```c
// 文件位置：ae.c:124-139

/*
 * aeDeleteEventLoop：销毁事件循环
 *
 * 【作用】
 * 释放事件循环的所有资源
 *
 * 【清理内容】
 * 1. 关闭底层 I/O 多路复用 fd（epoll/kqueue）
 * 2. 释放 events 和 fired 数组
 * 3. 释放所有定时事件（调用 finalizerProc）
 * 4. 释放 eventLoop 本身
 *
 * 【使用场景】
 * Redis 退出时调用
 */
void aeDeleteEventLoop(aeEventLoop *eventLoop) {
    /* 1. 释放底层 I/O 多路复用资源 */
    aeApiFree(eventLoop);

    /* 2. 释放 events 和 fired 数组 */
    zfree(eventLoop->events);
    zfree(eventLoop->fired);

    /*
     * 3. 释放所有定时事件
     *
     * 遍历定时事件链表，调用每个事件的 finalizerProc（如果有）
     */
    /* Free the time events list. */
    aeTimeEvent *next_te, *te = eventLoop->timeEventHead;
    while (te) {
        next_te = te->next;
        if (te->finalizerProc)
            te->finalizerProc(eventLoop, te->clientData);
        zfree(te);
        te = next_te;
    }

    /* 4. 释放 eventLoop 本身 */
    zfree(eventLoop);
}
```

### aeStop：停止事件循环

```c
// 文件位置：ae.c:141-143

/*
 * aeStop：停止事件循环
 *
 * 【作用】
 * 设置 stop 标志，aeMain() 会在下次循环时退出
 *
 * 【使用场景】
 * - 收到 SHUTDOWN 命令
 * - 收到 SIGTERM 信号
 */
void aeStop(aeEventLoop *eventLoop) {
    eventLoop->stop = 1;
}
```

---

### 注册文件事件

```c
// 文件位置：ae.c:145-179

/*
 * aeCreateFileEvent：注册文件事件
 *
 * 【作用】
 * 监听指定 fd 的 I/O 事件（可读/可写）
 *
 * 【参数】
 * - eventLoop：事件循环对象
 * - fd：文件描述符（socket）
 * - mask：事件类型（AE_READABLE、AE_WRITABLE）
 * - proc：回调函数
 * - clientData：自定义数据（通常是 client 结构）
 *
 * 【返回值】
 * - AE_OK (0)：成功
 * - AE_ERR (-1)：失败
 *
 * 【使用场景】
 * 1. 监听客户端连接：
 *    aeCreateFileEvent(server.el, server.ipfd[0], AE_READABLE, acceptTcpHandler, NULL);
 * 2. 读取客户端命令：
 *    aeCreateFileEvent(server.el, c->fd, AE_READABLE, readQueryFromClient, c);
 * 3. 发送响应给客户端：
 *    aeCreateFileEvent(server.el, c->fd, AE_WRITABLE, sendReplyToClient, c);
 */
int aeCreateFileEvent(aeEventLoop *eventLoop, int fd, int mask,
        aeFileProc *proc, void *clientData)
{
    /* 1. 检查 fd 是否超出范围 */
    if (fd >= eventLoop->setsize) {
        errno = ERANGE;  /* 超出范围 */
        return AE_ERR;
    }

    /*
     * 2. 动态扩容 events 和 fired 数组（如果需要）
     *
     * 如果 fd >= nevents，需要扩容
     *
     * 扩容策略：
     * - 方案 1：扩容到 nevents * 2（如果够用）
     * - 方案 2：扩容到 fd + 1（如果 nevents * 2 不够）
     * - 限制：不能超过 setsize
     *
     * 例子：
     * 当前 nevents=1024，fd=1500
     * 计算：max(1024*2, 1500+1) = 2048
     * 但不能超过 setsize（假设 10000），所以扩容到 2048
     *
     * Resize the events and fired arrays if the file
     * descriptor exceeds the current number of events.
     */
    if (unlikely(fd >= eventLoop->nevents)) {
        int newnevents = eventLoop->nevents;
        newnevents = (newnevents * 2 > fd + 1) ? newnevents * 2 : fd + 1;
        newnevents = (newnevents > eventLoop->setsize) ? eventLoop->setsize : newnevents;

        eventLoop->events = zrealloc(eventLoop->events, sizeof(aeFileEvent) * newnevents);
        eventLoop->fired = zrealloc(eventLoop->fired, sizeof(aeFiredEvent) * newnevents);

        /* 初始化新槽位 */
        /* Initialize new slots with an AE_NONE mask */
        for (int i = eventLoop->nevents; i < newnevents; i++)
            eventLoop->events[i].mask = AE_NONE;

        eventLoop->nevents = newnevents;
    }

    /* 3. 获取 fd 对应的 aeFileEvent 结构 */
    aeFileEvent *fe = &eventLoop->events[fd];

    /*
     * 4. 添加到底层 I/O 多路复用
     *
     * aeApiAddEvent() 的实现因平台而异：
     * - Linux：epoll_ctl(epfd, EPOLL_CTL_ADD/MOD, fd, &ev)
     * - BSD/Mac：kevent(kqfd, &kev, 1, NULL, 0, NULL)
     * - 其他：FD_SET(fd, &readfds) 或 FD_SET(fd, &writefds)
     */
    if (aeApiAddEvent(eventLoop, fd, mask) == -1)
        return AE_ERR;

    /*
     * 5. 更新 aeFileEvent 结构
     *
     * 注意：使用 |= 是因为可能已经注册了另一个方向的事件
     * 例如：已经注册了 AE_READABLE，现在注册 AE_WRITABLE
     * mask = AE_READABLE | AE_WRITABLE
     */
    fe->mask |= mask;
    if (mask & AE_READABLE) fe->rfileProc = proc;  /* 设置可读回调 */
    if (mask & AE_WRITABLE) fe->wfileProc = proc;  /* 设置可写回调 */
    fe->clientData = clientData;

    /* 6. 更新 maxfd */
    if (fd > eventLoop->maxfd)
        eventLoop->maxfd = fd;

    return AE_OK;
}
```

### aeDeleteFileEvent：删除文件事件

```c
// 文件位置：ae.c:181-201

/*
 * aeDeleteFileEvent：删除文件事件
 *
 * 【作用】
 * 取消监听指定 fd 的 I/O 事件
 *
 * 【参数】
 * - eventLoop：事件循环对象
 * - fd：文件描述符
 * - mask：要删除的事件类型（AE_READABLE、AE_WRITABLE）
 *
 * 【使用场景】
 * 1. 发送完响应后，删除可写事件：
 *    aeDeleteFileEvent(server.el, c->fd, AE_WRITABLE);
 * 2. 关闭客户端连接时，删除所有事件：
 *    aeDeleteFileEvent(server.el, c->fd, AE_READABLE | AE_WRITABLE);
 */
void aeDeleteFileEvent(aeEventLoop *eventLoop, int fd, int mask)
{
    /* 1. 检查 fd 是否超出范围 */
    if (fd >= eventLoop->setsize) return;

    aeFileEvent *fe = &eventLoop->events[fd];
    if (fe->mask == AE_NONE) return;  /* 没有注册事件，直接返回 */

    /*
     * 2. 如果删除 AE_WRITABLE，也删除 AE_BARRIER
     *
     * 为什么？
     * AE_BARRIER 总是和 AE_WRITABLE 一起使用
     * 如果删除 AE_WRITABLE，AE_BARRIER 就没意义了
     *
     * We want to always remove AE_BARRIER if set when AE_WRITABLE
     * is removed.
     */
    if (mask & AE_WRITABLE) mask |= AE_BARRIER;

    /*
     * 3. 从底层 I/O 多路复用删除
     *
     * aeApiDelEvent() 的实现因平台而异：
     * - Linux：epoll_ctl(epfd, EPOLL_CTL_MOD/DEL, fd, &ev)
     * - BSD/Mac：kevent(kqfd, &kev, 1, NULL, 0, NULL)
     * - 其他：FD_CLR(fd, &readfds) 或 FD_CLR(fd, &writefds)
     */
    aeApiDelEvent(eventLoop, fd, mask);

    /* 4. 更新 mask（清除指定的位） */
    fe->mask = fe->mask & (~mask);

    /*
     * 5. 更新 maxfd（如果删除的是最大的 fd）
     *
     * 为什么需要更新 maxfd？
     * 优化事件遍历：只遍历 0 到 maxfd
     *
     * 如何更新？
     * 从 maxfd-1 向下查找第一个 mask != AE_NONE 的 fd
     */
    if (fd == eventLoop->maxfd && fe->mask == AE_NONE) {
        /* Update the max fd */
        int j;

        for (j = eventLoop->maxfd-1; j >= 0; j--)
            if (eventLoop->events[j].mask != AE_NONE) break;
        eventLoop->maxfd = j;
    }
}
```

### aeGetFileClientData / aeGetFileEvents

```c
// 文件位置：ae.c:203-216

/*
 * aeGetFileClientData：获取 fd 的自定义数据
 *
 * 【返回值】
 * 注册事件时传入的 clientData（通常是 client 结构）
 */
void *aeGetFileClientData(aeEventLoop *eventLoop, int fd) {
    if (fd >= eventLoop->setsize) return NULL;
    aeFileEvent *fe = &eventLoop->events[fd];
    if (fe->mask == AE_NONE) return NULL;

    return fe->clientData;
}

/*
 * aeGetFileEvents：获取 fd 注册的事件
 *
 * 【返回值】
 * 事件掩码（AE_READABLE、AE_WRITABLE 或组合）
 */
int aeGetFileEvents(aeEventLoop *eventLoop, int fd) {
    if (fd >= eventLoop->setsize) return 0;
    aeFileEvent *fe = &eventLoop->events[fd];

    return fe->mask;
}
```

---

### 创建定时事件

```c
// 文件位置：ae.c:218-239

/*
 * aeCreateTimeEvent：创建定时事件
 *
 * 【作用】
 * 注册一个定时任务，在指定时间后执行
 *
 * 【参数】
 * - eventLoop：事件循环对象
 * - milliseconds：多少毫秒后执行
 * - proc：回调函数
 * - clientData：自定义数据
 * - finalizerProc：清理回调（可以是 NULL）
 *
 * 【返回值】
 * - 成功：返回定时事件的 ID（>= 0）
 * - 失败：返回 AE_ERR (-1)
 *
 * 【使用场景】
 * 创建 serverCron 定时任务（每 100ms 执行一次）：
 * aeCreateTimeEvent(server.el, 1, serverCron, NULL, NULL);
 */
long long aeCreateTimeEvent(aeEventLoop *eventLoop, long long milliseconds,
        aeTimeProc *proc, void *clientData,
        aeEventFinalizerProc *finalizerProc)
{
    /* 1. 分配定时事件 ID */
    long long id = eventLoop->timeEventNextId++;

    /* 2. 分配 aeTimeEvent 结构 */
    aeTimeEvent *te;
    te = zmalloc(sizeof(*te));
    if (te == NULL) return AE_ERR;

    /* 3. 初始化字段 */
    te->id = id;

    /*
     * 4. 计算下次执行时间
     *
     * getMonotonicUs()：获取当前单调时钟（微秒）
     * milliseconds * 1000：转换为微秒
     *
     * 例子：
     * 当前时间 = 1000000 us (1秒)
     * milliseconds = 100
     * when = 1000000 + 100*1000 = 1100000 us (1.1秒)
     */
    te->when = getMonotonicUs() + milliseconds * 1000;

    te->timeProc = proc;
    te->finalizerProc = finalizerProc;
    te->clientData = clientData;
    te->refcount = 0;

    /*
     * 5. 插入到链表头部
     *
     * 注意：链表是无序的（不按时间排序）
     * 为什么不排序？
     * - 插入更快（O(1) vs O(n)）
     * - Redis 定时事件很少（通常只有 serverCron）
     * - 查找最早的定时器时再遍历（usUntilEarliestTimer）
     */
    te->prev = NULL;
    te->next = eventLoop->timeEventHead;
    if (te->next)
        te->next->prev = te;
    eventLoop->timeEventHead = te;

    return id;
}
```

### aeDeleteTimeEvent：删除定时事件

```c
// 文件位置：ae.c:241-252

/*
 * aeDeleteTimeEvent：删除定时事件
 *
 * 【作用】
 * 标记定时事件为删除，真正的删除会在 processTimeEvents() 中完成
 *
 * 【参数】
 * - eventLoop：事件循环对象
 * - id：定时事件的 ID
 *
 * 【返回值】
 * - AE_OK (0)：成功
 * - AE_ERR (-1)：失败（找不到这个 ID）
 *
 * 【为什么不立即删除？】
 * 防止在执行定时事件回调时，回调内部删除了这个定时事件，导致崩溃。
 * 使用延迟删除：先标记 id = AE_DELETED_EVENT_ID，下次遍历时再释放。
 */
int aeDeleteTimeEvent(aeEventLoop *eventLoop, long long id)
{
    aeTimeEvent *te = eventLoop->timeEventHead;

    /* 遍历链表查找 id */
    while(te) {
        if (te->id == id) {
            te->id = AE_DELETED_EVENT_ID;  /* 标记为删除 */
            return AE_OK;
        }
        te = te->next;
    }

    return AE_ERR; /* NO event with the specified ID found */
}
```

### usUntilEarliestTimer：计算最早定时器的剩余时间

```c
// 文件位置：ae.c:254-276

/*
 * usUntilEarliestTimer：计算距离最早定时器的微秒数
 *
 * 【作用】
 * 计算到下一个定时器触发还有多少微秒
 * 用于设置 epoll_wait 的超时时间
 *
 * 【返回值】
 * - -1：没有定时事件
 * - 0：有定时事件已经到期
 * - >0：距离最早定时器的微秒数
 *
 * 【复杂度】
 * O(n)，n 是定时事件数量
 *
 * 【可能的优化】
 * 1. 按时间排序插入：查找最早的变成 O(1)，但插入变成 O(n)
 * 2. 使用跳表（skiplist）：插入和查找都是 O(log n)
 *
 * Redis 没有这样做，因为定时事件很少（通常只有 serverCron）
 *
 * How many microseconds until the first timer should fire.
 * If there are no timers, -1 is returned.
 *
 * Note that's O(N) since time events are unsorted.
 * Possible optimizations (not needed by Redis so far, but...):
 * 1) Insert the event in order, so that the nearest is just the head.
 *    Much better but still insertion or deletion of timers is O(N).
 * 2) Use a skiplist to have this operation as O(1) and insertion as O(log(N)).
 */
static int64_t usUntilEarliestTimer(aeEventLoop *eventLoop) {
    aeTimeEvent *te = eventLoop->timeEventHead;
    if (te == NULL) return -1;  /* 没有定时事件 */

    /* 遍历链表，找到最早的定时器 */
    aeTimeEvent *earliest = NULL;
    while (te) {
        if ((!earliest || te->when < earliest->when) && te->id != AE_DELETED_EVENT_ID)
            earliest = te;
        te = te->next;
    }

    /* 计算剩余时间 */
    monotime now = getMonotonicUs();
    return (now >= earliest->when) ? 0 : earliest->when - now;
}
```

### processTimeEvents：处理定时事件

```c
// 文件位置：ae.c:278-343

/*
 * processTimeEvents：处理所有到期的定时事件
 *
 * 【作用】
 * 遍历定时事件链表，执行所有已到期的定时器
 *
 * 【返回值】
 * 处理的定时事件数量
 *
 * 【处理流程】
 * 1. 删除标记为 AE_DELETED_EVENT_ID 的定时器
 * 2. 执行到期的定时器回调
 * 3. 根据回调返回值决定：
 *    - 返回 AE_NOMORE：标记为删除
 *    - 返回 N：N 毫秒后再次执行
 *
 * Process time events
 */
static int processTimeEvents(aeEventLoop *eventLoop) {
    int processed = 0;  /* 处理的事件数量 */
    aeTimeEvent *te;
    long long maxId;

    te = eventLoop->timeEventHead;

    /*
     * maxId：本次迭代前的最大 ID
     *
     * 作用：防止处理本次迭代中创建的定时器
     * 例如：serverCron 中创建了新的定时器，不应该在本次迭代中处理
     */
    maxId = eventLoop->timeEventNextId-1;

    monotime now = getMonotonicUs();  /* 当前时间 */

    while(te) {
        long long id;

        /*
         * 1. 删除标记为删除的定时器
         *
         * Remove events scheduled for deletion.
         */
        if (te->id == AE_DELETED_EVENT_ID) {
            aeTimeEvent *next = te->next;

            /*
             * 检查引用计数
             *
             * 如果 refcount > 0，说明正在执行这个定时器的回调
             * 不能删除，跳过
             *
             * If a reference exists for this timer event,
             * don't free it. This is currently incremented
             * for recursive timerProc calls
             */
            if (te->refcount) {
                te = next;
                continue;
            }

            /* 从链表中移除 */
            if (te->prev)
                te->prev->next = te->next;
            else
                eventLoop->timeEventHead = te->next;
            if (te->next)
                te->next->prev = te->prev;

            /* 调用清理回调 */
            if (te->finalizerProc) {
                te->finalizerProc(eventLoop, te->clientData);
                now = getMonotonicUs();  /* 重新获取时间（finalizerProc 可能耗时） */
            }

            /* 释放内存 */
            zfree(te);
            te = next;
            continue;
        }

        /*
         * 2. 跳过本次迭代中创建的定时器
         *
         * Make sure we don't process time events created by time events in
         * this iteration. Note that this check is currently useless: we always
         * add new timers on the head, however if we change the implementation
         * detail, this check may be useful again: we keep it here for future
         * defense.
         */
        if (te->id > maxId) {
            te = te->next;
            continue;
        }

        /*
         * 3. 执行到期的定时器
         */
        if (te->when <= now) {
            int retval;

            id = te->id;

            /*
             * 增加引用计数
             * 防止回调中删除这个定时器导致崩溃
             */
            te->refcount++;

            /* 调用定时器回调 */
            retval = te->timeProc(eventLoop, id, te->clientData);

            /* 减少引用计数 */
            te->refcount--;

            processed++;  /* 统计处理数量 */

            /* 重新获取时间（回调可能耗时） */
            now = getMonotonicUs();

            /*
             * 根据返回值决定下一步
             *
             * - AE_NOMORE (-1)：标记为删除
             * - N (正整数)：N 毫秒后再次执行
             *
             * 例子：
             * serverCron 返回 100，表示 100ms 后再次执行
             */
            if (retval != AE_NOMORE) {
                te->when = now + (monotime)retval * 1000;
            } else {
                te->id = AE_DELETED_EVENT_ID;
            }
        }
        te = te->next;
    }

    return processed;
}
```

---

### 处理事件（核心）

```c
// 文件位置：ae.c:345-468

/*
 * aeProcessEvents：处理事件（核心函数！）
 *
 * 【作用】
 * 这是事件循环的核心！处理文件事件和定时事件。
 *
 * 【参数】
 * - eventLoop：事件循环对象
 * - flags：处理标志
 *   - AE_FILE_EVENTS：处理文件事件
 *   - AE_TIME_EVENTS：处理定时事件
 *   - AE_ALL_EVENTS：处理所有事件
 *   - AE_DONT_WAIT：不阻塞等待
 *   - AE_CALL_BEFORE_SLEEP：调用 beforesleep
 *   - AE_CALL_AFTER_SLEEP：调用 aftersleep
 *
 * 【返回值】
 * 处理的事件数量
 *
 * 【处理流程】
 * 1. 如果需要处理文件事件或定时事件：
 *    a. 调用 beforesleep（如果需要）
 *    b. 调用 aeApiPoll() 等待事件（epoll_wait）
 *    c. 调用 aftersleep（如果需要）
 *    d. 处理触发的文件事件（调用回调函数）
 * 2. 处理到期的定时事件
 * 3. 返回处理的事件数量
 *
 * Process every pending file event, then every pending time event
 * (that may be registered by file event callbacks just processed).
 * Without special flags the function sleeps until some file event
 * fires, or when the next time event occurs (if any).
 *
 * If flags is 0, the function does nothing and returns.
 * if flags has AE_ALL_EVENTS set, all the kind of events are processed.
 * if flags has AE_FILE_EVENTS set, file events are processed.
 * if flags has AE_TIME_EVENTS set, time events are processed.
 * if flags has AE_DONT_WAIT set, the function returns ASAP once all
 * the events that can be handled without a wait are processed.
 * if flags has AE_CALL_AFTER_SLEEP set, the aftersleep callback is called.
 * if flags has AE_CALL_BEFORE_SLEEP set, the beforesleep callback is called.
 *
 * The function returns the number of events processed.
 */
int aeProcessEvents(aeEventLoop *eventLoop, int flags)
{
    int processed = 0, numevents;

    /* Nothing to do? return ASAP */
    if (!(flags & AE_TIME_EVENTS) && !(flags & AE_FILE_EVENTS)) return 0;

    /*
     * Note that we want to call aeApiPoll() even if there are no
     * file events to process as long as we want to process time
     * events, in order to sleep until the next time event is ready
     * to fire.
     *
     * 即使没有文件事件，也要调用 aeApiPoll()，
     * 因为需要睡眠到下一个定时器触发。
     */
    if (eventLoop->maxfd != -1 ||
        ((flags & AE_TIME_EVENTS) && !(flags & AE_DONT_WAIT))) {
        int j;
        struct timeval tv, *tvp = NULL; /* NULL means infinite wait. */
        int64_t usUntilTimer;

        /*
         * 1. 调用 beforesleep 回调
         *
         * Redis 在这里做很多重要工作：
         * - 处理客户端输出缓冲区
         * - AOF 刷盘
         * - 处理异步操作
         */
        if (eventLoop->beforesleep != NULL && (flags & AE_CALL_BEFORE_SLEEP))
            eventLoop->beforesleep(eventLoop);

        /*
         * 2. 计算 epoll_wait 的超时时间
         *
         * 优先级：
         * 1. 如果 flags 或 eventLoop->flags 包含 AE_DONT_WAIT：timeout=0（不等待）
         * 2. 如果需要处理定时事件：timeout=到最早定时器的时间
         * 3. 否则：timeout=NULL（无限等待）
         *
         * The eventLoop->flags may be changed inside beforesleep.
         * So we should check it after beforesleep be called. At the same time,
         * the parameter flags always should have the highest priority.
         * That is to say, once the parameter flag is set to AE_DONT_WAIT,
         * no matter what value eventLoop->flags is set to, we should ignore it.
         */
        if ((flags & AE_DONT_WAIT) || (eventLoop->flags & AE_DONT_WAIT)) {
            /* 不等待 */
            tv.tv_sec = tv.tv_usec = 0;
            tvp = &tv;
        } else if (flags & AE_TIME_EVENTS) {
            /* 等待到下一个定时器 */
            usUntilTimer = usUntilEarliestTimer(eventLoop);
            if (usUntilTimer >= 0) {
                tv.tv_sec = usUntilTimer / 1000000;
                tv.tv_usec = usUntilTimer % 1000000;
                tvp = &tv;
            }
            /* 如果 usUntilTimer < 0（没有定时器），tvp=NULL（无限等待） */
        }

        /*
         * 3. 调用底层 I/O 多路复用，等待事件
         *
         * aeApiPoll() 的实现因平台而异：
         * - Linux：epoll_wait(epfd, events, nevents, timeout)
         * - BSD/Mac：kevent(kqfd, NULL, 0, events, nevents, &timeout)
         * - 其他：select(maxfd+1, &readfds, &writefds, NULL, &timeout)
         *
         * 返回值：就绪的事件数量
         *
         * Call the multiplexing API, will return only on timeout or when
         * some event fires.
         */
        numevents = aeApiPoll(eventLoop, tvp);

        /*
         * 4. 根据 flags 决定是否处理文件事件
         *
         * Don't process file events if not requested.
         */
        if (!(flags & AE_FILE_EVENTS)) {
            numevents = 0;
        }

        /*
         * 5. 调用 aftersleep 回调
         *
         * After sleep callback.
         */
        if (eventLoop->aftersleep != NULL && flags & AE_CALL_AFTER_SLEEP)
            eventLoop->aftersleep(eventLoop);

        /*
         * 6. 处理触发的文件事件
         *
         * 遍历 fired[] 数组，调用对应的回调函数
         */
        for (j = 0; j < numevents; j++) {
            int fd = eventLoop->fired[j].fd;
            aeFileEvent *fe = &eventLoop->events[fd];
            int mask = eventLoop->fired[j].mask;
            int fired = 0; /* Number of events fired for current fd. */

            /*
             * 【重要】处理顺序：先读后写
             *
             * 为什么先读后写？
             * 例子：客户端发送命令 "GET key"
             * 1. 先读取命令（可读事件）
             * 2. 执行命令
             * 3. 立即发送响应（可写事件）
             *
             * 这样可以减少延迟！
             *
             * Normally we execute the readable event first, and the writable
             * event later. This is useful as sometimes we may be able
             * to serve the reply of a query immediately after processing the
             * query.
             */

            /*
             * 【AE_BARRIER】反转读写顺序
             *
             * 如果设置了 AE_BARRIER，先执行写事件，再执行读事件
             *
             * 使用场景：
             * 在 beforeSleep 中执行 fsync（持久化），然后发送响应
             * 必须先发送响应（写事件），再读取新命令（读事件）
             *
             * However if AE_BARRIER is set in the mask, our application is
             * asking us to do the reverse: never fire the writable event
             * after the readable. In such a case, we invert the calls.
             * This is useful when, for instance, we want to do things
             * in the beforeSleep() hook, like fsyncing a file to disk,
             * before replying to a client.
             */
            int invert = fe->mask & AE_BARRIER;

            /*
             * 注意：fe->mask & mask & ...
             *
             * 为什么需要这个检查？
             * 因为前面执行的回调可能删除了事件！
             *
             * 例子：
             * 1. 执行可读回调，发现客户端关闭连接
             * 2. 回调中删除了 fd 的所有事件（mask=AE_NONE）
             * 3. 不能再执行可写回调（会崩溃）
             *
             * Note the "fe->mask & mask & ..." code: maybe an already
             * processed event removed an element that fired and we still
             * didn't processed, so we check if the event is still valid.
             */

            /* 7a. 执行可读事件（如果没有反转） */
            if (!invert && fe->mask & mask & AE_READABLE) {
                fe->rfileProc(eventLoop,fd,fe->clientData,mask);
                fired++;
                fe = &eventLoop->events[fd]; /* Refresh in case of resize. */
            }

            /* 7b. 执行可写事件 */
            /* Fire the writable event. */
            if (fe->mask & mask & AE_WRITABLE) {
                /*
                 * 避免重复调用
                 *
                 * 如果读写回调是同一个函数，且已经调用过（fired>0），
                 * 就不再调用
                 */
                if (!fired || fe->wfileProc != fe->rfileProc) {
                    fe->wfileProc(eventLoop,fd,fe->clientData,mask);
                    fired++;
                }
            }

            /* 7c. 执行可读事件（如果反转了） */
            /* If we have to invert the call, fire the readable event now
             * after the writable one. */
            if (invert) {
                fe = &eventLoop->events[fd]; /* Refresh in case of resize. */
                if ((fe->mask & mask & AE_READABLE) &&
                    (!fired || fe->wfileProc != fe->rfileProc))
                {
                    fe->rfileProc(eventLoop,fd,fe->clientData,mask);
                    fired++;
                }
            }

            processed++;
        }
    }

    /*
     * 8. 处理定时事件
     *
     * Check time events
     */
    if (flags & AE_TIME_EVENTS)
        processed += processTimeEvents(eventLoop);

    return processed; /* return the number of processed file/time events */
}
```

---

### aeWait：等待 fd 事件

```c
// 文件位置：ae.c:470-490

/*
 * aeWait：等待 fd 上的事件（阻塞式）
 *
 * 【作用】
 * 同步等待指定 fd 上的事件（可读/可写）
 *
 * 【参数】
 * - fd：文件描述符
 * - mask：要等待的事件（AE_READABLE、AE_WRITABLE）
 * - milliseconds：超时时间（毫秒）
 *
 * 【返回值】
 * - 0：超时
 * - >0：触发的事件掩码
 * - -1：错误
 *
 * 【实现】
 * 使用 poll() 系统调用（不是 epoll）
 *
 * 【使用场景】
 * 主从复制中，等待连接建立：
 * aeWait(fd, AE_WRITABLE, 1000);  // 等待 socket 可写（连接成功）
 *
 * Wait for milliseconds until the given file descriptor becomes
 * writable/readable/exception
 */
int aeWait(int fd, int mask, long long milliseconds) {
    struct pollfd pfd;
    int retmask = 0, retval;

    /* 初始化 pollfd 结构 */
    memset(&pfd, 0, sizeof(pfd));
    pfd.fd = fd;
    if (mask & AE_READABLE) pfd.events |= POLLIN;   /* 等待可读 */
    if (mask & AE_WRITABLE) pfd.events |= POLLOUT;  /* 等待可写 */

    /* 调用 poll() 等待 */
    if ((retval = poll(&pfd, 1, milliseconds))== 1) {
        /* 有事件触发，转换为 AE_* 格式 */
        if (pfd.revents & POLLIN) retmask |= AE_READABLE;
        if (pfd.revents & POLLOUT) retmask |= AE_WRITABLE;
        if (pfd.revents & POLLERR) retmask |= AE_WRITABLE;  /* 错误也返回可写 */
        if (pfd.revents & POLLHUP) retmask |= AE_WRITABLE;  /* 挂断也返回可写 */
        return retmask;
    } else {
        /* 超时或错误 */
        return retval;
    }
}
```

---

### aeMain：主事件循环

```c
// 文件位置：ae.c:492-499

/*
 * aeMain：主事件循环（Redis 的心脏！）
 *
 * 【作用】
 * 无限循环处理事件，直到 stop 标志被设置
 *
 * 【实现】
 * while (!eventLoop->stop) {
 *     aeProcessEvents(eventLoop, AE_ALL_EVENTS | AE_CALL_BEFORE_SLEEP | AE_CALL_AFTER_SLEEP);
 * }
 *
 * 【使用场景】
 * Redis 启动的最后一步：
 * aeMain(server.el);  // 进入事件循环，永不返回（除非收到 SHUTDOWN 命令）
 *
 * 【退出条件】
 * 1. 收到 SHUTDOWN 命令：aeStop(server.el)
 * 2. 收到 SIGTERM 信号：信号处理函数中调用 aeStop(server.el)
 */
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        aeProcessEvents(eventLoop, AE_ALL_EVENTS|
                                   AE_CALL_BEFORE_SLEEP|
                                   AE_CALL_AFTER_SLEEP);
    }
}
```

---

### 工具函数

```c
// 文件位置：ae.c:501-512

/*
 * aeGetApiName：获取底层 I/O 多路复用库的名称
 *
 * 【返回值】
 * - "epoll"（Linux）
 * - "kqueue"（BSD/Mac）
 * - "select"（其他）
 * - "evport"（Solaris）
 *
 * 【实现】
 * 调用 ae_*.c 中定义的 aeApiName()
 */
char *aeGetApiName(void) {
    return aeApiName();
}

/*
 * aeSetBeforeSleepProc：设置睡眠前回调
 *
 * Redis 在这里做很多重要工作：
 * 1. beforeSleep() 函数会被调用（在 server.c 中定义）
 * 2. 处理客户端输出缓冲区
 * 3. AOF 刷盘
 * 4. 处理异步操作
 */
void aeSetBeforeSleepProc(aeEventLoop *eventLoop, aeBeforeSleepProc *beforesleep) {
    eventLoop->beforesleep = beforesleep;
}

/*
 * aeSetAfterSleepProc：设置睡眠后回调
 *
 * Redis 很少使用
 */
void aeSetAfterSleepProc(aeEventLoop *eventLoop, aeBeforeSleepProc *aftersleep) {
    eventLoop->aftersleep = aftersleep;
}
```

---

## 内部实现细节

### 动态扩容机制

**events 和 fired 数组的动态扩容**：

1. **初始大小**：`min(setsize, 1024)`
2. **扩容时机**：当 `fd >= nevents` 时
3. **扩容策略**：
   - 方案 1：扩容到 `nevents * 2`（如果够用）
   - 方案 2：扩容到 `fd + 1`（如果 `nevents * 2` 不够）
   - 限制：不能超过 `setsize`

**为什么要动态扩容？**
- 节省内存：大多数情况下不会用满所有 fd
- 例如：`setsize=10000`，初始只分配 1024 个槽位

### 定时事件链表

**为什么不排序？**
- 插入更快（O(1) vs O(n)）
- Redis 定时事件很少（通常只有 serverCron）
- 查找最早的定时器时再遍历（usUntilEarliestTimer）

**可能的优化**（Redis 没有使用）：
1. 按时间排序插入：查找最早的变成 O(1)，但插入变成 O(n)
2. 使用跳表（skiplist）：插入和查找都是 O(log n)

### AE_BARRIER 标志

**作用**：强制先执行写事件，再执行读事件（正常是先读后写）

**使用场景**：
```c
// 1. 客户端发送命令 "SET key value"
// 2. Redis 执行命令，准备响应
// 3. beforesleep 中执行 fsync（持久化到磁盘）
// 4. 发送 +OK 给客户端（如果先发送响应，客户端以为成功了，但数据还没持久化）
// 5. 读取下一条命令

// 使用 AE_BARRIER 确保顺序：fsync → 发送响应 → 读取命令
aeCreateFileEvent(server.el, c->fd, AE_WRITABLE | AE_BARRIER, sendReplyToClient, c);
```

---

## 典型执行流程

### Redis 启动流程

```c
int main(int argc, char **argv) {
    // 1. 初始化配置
    initServerConfig();

    // 2. 创建事件循环
    server.el = aeCreateEventLoop(server.maxclients + CONFIG_FDSET_INCR);

    // 3. 打开监听 socket
    listenToPort(server.port, &server.ipfd_count, server.ipfd);

    // 4. 注册监听事件
    for (j = 0; j < server.ipfd_count; j++) {
        aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE,
            acceptTcpHandler, NULL);
    }

    // 5. 创建 serverCron 定时器
    aeCreateTimeEvent(server.el, 1, serverCron, NULL, NULL);

    // 6. 设置 beforesleep
    aeSetBeforeSleepProc(server.el, beforeSleep);

    // 7. 进入事件循环
    aeMain(server.el);

    // 8. 清理
    aeDeleteEventLoop(server.el);
    return 0;
}
```

### 一次完整的事件循环

```
1. aeMain() 调用 aeProcessEvents()
   ↓
2. 调用 beforesleep()
   - 处理客户端输出缓冲区
   - AOF 刷盘
   - 处理异步操作
   ↓
3. 计算超时时间
   - 查找最早的定时器
   - timeout = min(到最早定时器的时间, 无限)
   ↓
4. 调用 aeApiPoll(eventLoop, timeout)
   - Linux: epoll_wait(epfd, events, nevents, timeout)
   - 阻塞等待事件（或超时）
   ↓
5. 调用 aftersleep()
   ↓
6. 处理触发的文件事件
   - 遍历 fired[] 数组
   - 调用回调函数（readQueryFromClient, sendReplyToClient）
   ↓
7. 处理到期的定时事件
   - 遍历定时事件链表
   - 调用回调函数（serverCron）
   ↓
8. 返回 aeMain()，继续下次循环
```

### 处理客户端命令流程

```
1. 客户端连接：acceptTcpHandler
   - accept() 接受连接
   - createClient() 创建客户端结构
   - aeCreateFileEvent(fd, AE_READABLE, readQueryFromClient)
   ↓
2. 客户端发送命令：readQueryFromClient
   - read() 读取命令
   - processInputBuffer() 解析命令
   - call() 执行命令
   - prepareClientToWrite() 准备响应
   - aeCreateFileEvent(fd, AE_WRITABLE, sendReplyToClient)
   ↓
3. 发送响应：sendReplyToClient
   - write() 发送响应
   - aeDeleteFileEvent(fd, AE_WRITABLE) 删除可写事件
   ↓
4. 等待下一条命令（回到步骤 2）
```

---

## 总结

**ae.c 实现了 Redis 事件循环的核心逻辑**，包括：

1. **事件循环管理**：创建、销毁、运行
2. **文件事件**：注册、删除、触发（I/O 多路复用）
3. **定时事件**：创建、删除、执行（周期任务）
4. **事件处理**：aeProcessEvents（核心函数）

**关键优化**：
1. **动态扩容**：节省内存，按需分配
2. **先读后写**：减少延迟，快速响应
3. **beforesleep**：批量处理输出缓冲区
4. **单调时钟**：防止系统时间修改影响定时器

**为什么 Redis 这么快？**
1. **单线程 + I/O 多路复用**：避免线程切换，充分利用 CPU
2. **事件驱动**：不阻塞等待，高效处理并发
3. **内存操作为主**：减少磁盘 I/O
4. **精心设计的事件循环**：最小化延迟

**下一步阅读建议**：
- `ae_epoll.c`：Linux epoll 实现
- `networking.c`：网络 I/O 处理
- `server.c`：beforeSleep, serverCron 实现
- `anet.c`：网络工具函数

---

> 📝 **注意**：这份注释基于 Redis 8.4.0 源码，不同版本可能有细微差异。
