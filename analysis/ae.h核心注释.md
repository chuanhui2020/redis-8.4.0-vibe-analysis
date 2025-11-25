# Redis ae.h 事件循环核心详解（中文注释版）

> 📘 **说明**：ae.h/ae.c 是 Redis 的事件循环库，负责处理所有的 I/O 事件（网络读写）和定时事件（周期任务）。这是 Redis 高性能的基础！

---

## 📚 目录

1. [文件概述](#文件概述)
2. [核心概念](#核心概念)
3. [常量定义](#常量定义)
4. [回调函数类型](#回调函数类型)
5. [核心结构体](#核心结构体)
6. [API 函数说明](#api-函数说明)
7. [典型使用流程](#典型使用流程)

---

## 文件概述

`ae.h` 是 Redis 事件驱动库的头文件，定义了事件循环的核心接口。

**文件位置**：`src/ae.h`（119行）

**核心思想**：
- Redis 是**单线程事件驱动**模型
- 使用 **I/O 多路复用**（epoll/kqueue/select）监听网络事件
- 使用**定时事件**处理周期性任务（如：持久化、过期键清理）
- 所有操作都在**一个线程的事件循环**中完成

**AE 是什么？**
- AE = **A**ntirez **E**vent library（作者 Antirez 写的事件库）
- 最早是为 Jim（Tcl 解释器）写的，后来移植到 Redis

---

## 核心概念

### 什么是事件循环？

事件循环就是一个**无限循环**，不断做这三件事：

```c
// 伪代码展示事件循环的本质
while (!stop) {
    // 1. 等待事件发生（网络数据到达、定时器到期）
    events = wait_for_events();

    // 2. 处理网络 I/O 事件（读取客户端命令、发送响应）
    for (event in events) {
        if (event.readable) read_from_client(event.fd);
        if (event.writable) write_to_client(event.fd);
    }

    // 3. 处理定时事件（后台任务，如持久化、过期键清理）
    run_expired_timers();
}
```

**Redis 中的事件循环**：
1. **启动时**：调用 `aeCreateEventLoop()` 创建事件循环
2. **注册事件**：
   - 调用 `aeCreateFileEvent()` 监听客户端连接、命令读写
   - 调用 `aeCreateTimeEvent()` 注册周期任务（如 serverCron）
3. **运行**：调用 `aeMain()` 进入无限循环
4. **退出**：调用 `aeStop()` 停止循环

---

## 常量定义

```c
// 文件位置：ae.h:18-19
#define AE_OK 0        /* 操作成功 */
#define AE_ERR -1      /* 操作失败 */
```

### 文件事件掩码

```c
// 文件位置：ae.h:21-28

/*
 * 文件事件类型（用位掩码表示，可以组合）
 * 例如：mask = AE_READABLE | AE_WRITABLE 表示同时监听读和写
 */
#define AE_NONE 0       /* 无事件：这个 fd 没有注册任何事件 */
#define AE_READABLE 1   /* 可读事件：当 socket 有数据到达时触发（客户端发来命令） */
#define AE_WRITABLE 2   /* 可写事件：当 socket 缓冲区可写时触发（可以发送响应给客户端） */

/*
 * AE_BARRIER：屏障标志（与 AE_WRITABLE 一起使用）
 * 作用：强制先执行写事件，再执行读事件（正常是先读后写）
 *
 * 使用场景：
 * 当你想在 beforesleep 中做 fsync（把数据刷到磁盘），
 * 然后再发送响应给客户端时使用。
 *
 * 例子：
 * 1. 客户端发送 SET key value
 * 2. Redis 处理命令，准备响应
 * 3. beforesleep 中执行 fsync（持久化）
 * 4. 然后发送 +OK 给客户端（如果先发送响应，客户端以为成功了，但数据还没持久化）
 */
#define AE_BARRIER 4
```

### 事件处理标志

```c
// 文件位置：ae.h:30-35

/*
 * aeProcessEvents() 的参数标志
 * 用于控制事件循环的行为
 */
#define AE_FILE_EVENTS (1<<0)   /* 处理文件事件（I/O 事件） */
#define AE_TIME_EVENTS (1<<1)   /* 处理定时事件 */
#define AE_ALL_EVENTS (AE_FILE_EVENTS|AE_TIME_EVENTS)  /* 处理所有事件 */
#define AE_DONT_WAIT (1<<2)     /* 不阻塞等待，处理完当前就绪事件后立即返回 */
#define AE_CALL_BEFORE_SLEEP (1<<3)  /* 在等待事件前调用 beforesleep 回调 */
#define AE_CALL_AFTER_SLEEP (1<<4)   /* 在等待事件后调用 aftersleep 回调 */
```

**使用场景**：
```c
// 例子 1：只处理 I/O 事件，不处理定时器
aeProcessEvents(eventLoop, AE_FILE_EVENTS | AE_DONT_WAIT);

// 例子 2：处理所有事件，调用 beforesleep/aftersleep（正常的事件循环）
aeProcessEvents(eventLoop, AE_ALL_EVENTS | AE_CALL_BEFORE_SLEEP | AE_CALL_AFTER_SLEEP);
```

### 特殊返回值

```c
// 文件位置：ae.h:37-38

/*
 * AE_NOMORE：定时事件返回这个值表示"不再重复执行"
 *
 * 定时事件的回调函数返回值含义：
 * - 返回 AE_NOMORE (-1)：删除这个定时器，不再执行
 * - 返回正整数 N：N 毫秒后再次执行
 *
 * 例子：
 * int myTimer(aeEventLoop *el, long long id, void *data) {
 *     printf("定时器触发\n");
 *     return 1000;  // 1000毫秒后再次执行（周期任务）
 * }
 *
 * int oneTimeTimer(aeEventLoop *el, long long id, void *data) {
 *     printf("只执行一次\n");
 *     return AE_NOMORE;  // 不再执行（一次性任务）
 * }
 */
#define AE_NOMORE -1

/*
 * AE_DELETED_EVENT_ID：标记定时事件已被删除
 *
 * 定时事件不会立即删除，而是先标记 id = AE_DELETED_EVENT_ID，
 * 然后在下次处理定时事件时真正释放内存。
 *
 * 为什么不立即删除？
 * 因为可能正在执行这个定时事件的回调，立即删除会导致崩溃。
 */
#define AE_DELETED_EVENT_ID -1
```

### 工具宏

```c
// 文件位置：ae.h:41
#define AE_NOTUSED(V) ((void) V)  /* 标记未使用的参数，避免编译器警告 */
```

---

## 回调函数类型

```c
// 文件位置：ae.h:43-49

struct aeEventLoop;  /* 前向声明，避免循环依赖 */

/*
 * aeFileProc：文件事件回调函数
 *
 * 参数：
 * - eventLoop：事件循环对象
 * - fd：触发事件的文件描述符（socket）
 * - clientData：注册事件时传入的自定义数据（通常是 client 结构）
 * - mask：触发的事件类型（AE_READABLE 或 AE_WRITABLE）
 *
 * 使用场景：
 * - 客户端连接到达：acceptTcpHandler()
 * - 客户端发送命令：readQueryFromClient()
 * - 发送响应给客户端：sendReplyToClient()
 */
typedef void aeFileProc(struct aeEventLoop *eventLoop, int fd, void *clientData, int mask);

/*
 * aeTimeProc：定时事件回调函数
 *
 * 参数：
 * - eventLoop：事件循环对象
 * - id：定时事件的唯一 ID
 * - clientData：注册事件时传入的自定义数据
 *
 * 返回值：
 * - AE_NOMORE (-1)：删除这个定时器
 * - N (正整数)：N 毫秒后再次执行
 *
 * 使用场景：
 * - serverCron()：Redis 的主定时任务（默认每 100ms 执行一次）
 *   负责过期键删除、rehash、统计信息更新等
 */
typedef int aeTimeProc(struct aeEventLoop *eventLoop, long long id, void *clientData);

/*
 * aeEventFinalizerProc：定时事件释放回调
 *
 * 在定时事件被删除时调用，用于清理 clientData
 *
 * 参数：
 * - eventLoop：事件循环对象
 * - clientData：注册事件时传入的自定义数据
 *
 * 使用场景：
 * 如果 clientData 是动态分配的内存，需要在这里释放
 */
typedef void aeEventFinalizerProc(struct aeEventLoop *eventLoop, void *clientData);

/*
 * aeBeforeSleepProc：睡眠前/后回调
 *
 * - beforesleep：在等待事件前调用（epoll_wait 之前）
 * - aftersleep：在等待事件后调用（epoll_wait 之后）
 *
 * 参数：
 * - eventLoop：事件循环对象
 *
 * 使用场景：
 * Redis 在 beforesleep 中做很多重要工作：
 * 1. 处理客户端输出缓冲区（把响应发送出去）
 * 2. 执行 AOF 刷盘（fsync）
 * 3. 处理异步 I/O
 * 4. 处理懒删除（lazy free）
 */
typedef void aeBeforeSleepProc(struct aeEventLoop *eventLoop);
```

---

## 核心结构体

### aeFileEvent：文件事件

```c
// 文件位置：ae.h:52-57

/*
 * aeFileEvent：文件事件结构
 *
 * 【作用】
 * 表示一个 socket（文件描述符）上注册的事件
 *
 * 【存储位置】
 * 存储在 eventLoop->events[] 数组中，数组下标就是 fd
 * 例如：eventLoop->events[5] 表示 fd=5 的事件
 *
 * 【使用场景】
 * - 监听 socket：等待客户端连接
 * - 客户端 socket：等待命令到达、等待缓冲区可写
 */
typedef struct aeFileEvent {
    /*
     * mask：事件掩码
     *
     * 取值：
     * - AE_NONE (0)：没有注册事件
     * - AE_READABLE (1)：监听可读事件
     * - AE_WRITABLE (2)：监听可写事件
     * - AE_READABLE | AE_WRITABLE：同时监听读写
     * - AE_WRITABLE | AE_BARRIER：先写后读
     */
    int mask;

    /*
     * rfileProc：可读事件回调函数
     * 当 socket 可读时（有数据到达），调用这个函数
     *
     * 常见回调：
     * - acceptTcpHandler：处理新连接
     * - readQueryFromClient：读取客户端命令
     */
    aeFileProc *rfileProc;

    /*
     * wfileProc：可写事件回调函数
     * 当 socket 可写时（缓冲区有空间），调用这个函数
     *
     * 常见回调：
     * - sendReplyToClient：发送响应给客户端
     */
    aeFileProc *wfileProc;

    /*
     * clientData：自定义数据
     * 通常指向一个 client 结构（表示客户端连接）
     */
    void *clientData;
} aeFileEvent;
```

### aeTimeEvent：定时事件

```c
// 文件位置：ae.h:60-70

/*
 * aeTimeEvent：定时事件结构
 *
 * 【作用】
 * 表示一个定时任务（在指定时间后执行）
 *
 * 【存储位置】
 * 使用双向链表存储，头指针在 eventLoop->timeEventHead
 *
 * 【使用场景】
 * - serverCron()：Redis 主定时任务（每 100ms 执行一次）
 * - 客户端超时检测
 * - AOF 重写、RDB 保存
 */
typedef struct aeTimeEvent {
    /*
     * id：定时事件的唯一标识符
     * 从 0 开始递增，删除时设置为 AE_DELETED_EVENT_ID
     */
    long long id;

    /*
     * when：下次执行的时间（微秒级时间戳）
     * 使用单调时钟（monotonic clock），不受系统时间修改影响
     */
    monotime when;

    /*
     * timeProc：定时事件回调函数
     * 返回值：
     * - AE_NOMORE：删除定时器
     * - N (正整数)：N 毫秒后再次执行
     */
    aeTimeProc *timeProc;

    /*
     * finalizerProc：清理回调
     * 在定时事件被删除时调用，用于释放 clientData
     */
    aeEventFinalizerProc *finalizerProc;

    /*
     * clientData：自定义数据
     */
    void *clientData;

    /*
     * prev/next：双向链表指针
     * 所有定时事件串成一个链表
     */
    struct aeTimeEvent *prev;
    struct aeTimeEvent *next;

    /*
     * refcount：引用计数
     *
     * 作用：防止在执行定时事件回调时，回调内部删除了这个定时事件，
     * 导致崩溃。
     *
     * 原理：
     * 1. 执行回调前 refcount++
     * 2. 执行回调
     * 3. 执行回调后 refcount--
     * 4. 如果 refcount > 0，延迟删除
     */
    int refcount;
} aeTimeEvent;
```

### aeFiredEvent：触发的事件

```c
// 文件位置：ae.h:73-76

/*
 * aeFiredEvent：已触发的事件
 *
 * 【作用】
 * aeApiPoll() 返回的就绪事件列表
 *
 * 【存储位置】
 * 存储在 eventLoop->fired[] 数组中
 *
 * 【生命周期】
 * 1. aeApiPoll() 从 epoll/kqueue 获取就绪事件，填充到 fired[] 数组
 * 2. aeProcessEvents() 遍历 fired[] 数组，调用对应的回调函数
 * 3. 下次 aeApiPoll() 会覆盖 fired[] 数组
 */
typedef struct aeFiredEvent {
    /* fd：触发事件的文件描述符 */
    int fd;

    /*
     * mask：触发的事件类型
     * - AE_READABLE：有数据可读
     * - AE_WRITABLE：缓冲区可写
     * - AE_READABLE | AE_WRITABLE：同时可读可写
     */
    int mask;
} aeFiredEvent;
```

### aeEventLoop：事件循环

```c
// 文件位置：ae.h:79-93

/*
 * aeEventLoop：事件循环核心结构
 *
 * 【作用】
 * 这是整个事件循环的中枢！管理所有文件事件和定时事件。
 *
 * 【使用场景】
 * Redis 启动时创建一个全局的 aeEventLoop，存储在 server.el 中
 *
 * 【生命周期】
 * 1. 启动：aeCreateEventLoop(setsize)
 * 2. 注册事件：aeCreateFileEvent(), aeCreateTimeEvent()
 * 3. 运行：aeMain(eventLoop)
 * 4. 停止：aeStop(eventLoop)
 * 5. 销毁：aeDeleteEventLoop(eventLoop)
 */
typedef struct aeEventLoop {
    /*
     * maxfd：当前注册的最大文件描述符
     *
     * 作用：优化事件遍历（只遍历 0 到 maxfd）
     *
     * 例子：
     * 如果只注册了 fd=3, fd=5, fd=10，那么 maxfd=10
     * 遍历时只需检查 events[0] 到 events[10]
     */
    int maxfd;

    /*
     * setsize：events 数组的最大容量（最多能监听多少个 fd）
     *
     * 通常设置为：
     * - 10000+2（小型服务器）
     * - 50000+2（大型服务器）
     *
     * 为什么 +2？
     * 留给监听 socket 和 pipe（用于信号处理）
     */
    int setsize;

    /*
     * timeEventNextId：下一个定时事件的 ID
     * 从 0 开始递增
     */
    long long timeEventNextId;

    /*
     * nevents：events 和 fired 数组的当前大小
     *
     * 动态扩容：
     * 初始值为 min(setsize, 1024)
     * 当 fd >= nevents 时，扩容到 fd+1 或 nevents*2（取较大值）
     */
    int nevents;

    /*
     * events：已注册的文件事件数组
     *
     * 数组下标就是 fd：events[fd] 表示 fd 上注册的事件
     *
     * 例子：
     * events[5].mask = AE_READABLE：fd=5 监听可读事件
     * events[5].rfileProc = readQueryFromClient：可读时调用这个函数
     */
    aeFileEvent *events;

    /*
     * fired：已触发的事件数组
     *
     * aeApiPoll() 把就绪的事件填充到这里
     * aeProcessEvents() 遍历这个数组，调用回调函数
     */
    aeFiredEvent *fired;

    /*
     * timeEventHead：定时事件链表头
     *
     * 所有定时事件串成一个双向链表
     * 注意：链表是无序的（不按时间排序）
     */
    aeTimeEvent *timeEventHead;

    /*
     * stop：停止标志
     *
     * 0：继续运行事件循环
     * 1：停止事件循环，aeMain() 会退出
     */
    int stop;

    /*
     * apidata：底层 I/O 多路复用的私有数据
     *
     * 不同平台指向不同结构：
     * - Linux：指向 aeApiState（包含 epoll fd）
     * - BSD/Mac：指向 aeApiState（包含 kqueue fd）
     * - 其他：指向 aeApiState（包含 fd_set）
     */
    void *apidata;

    /*
     * beforesleep：在等待事件前调用（epoll_wait 之前）
     *
     * Redis 在这里做：
     * 1. 发送响应给客户端
     * 2. AOF 刷盘
     * 3. 处理异步操作
     */
    aeBeforeSleepProc *beforesleep;

    /*
     * aftersleep：在等待事件后调用（epoll_wait 之后）
     *
     * Redis 很少使用这个回调
     */
    aeBeforeSleepProc *aftersleep;

    /*
     * flags：事件循环标志
     *
     * - AE_DONT_WAIT：不阻塞等待，立即返回
     */
    int flags;

    /*
     * privdata：私有数据（最多 2 个指针）
     * Redis 用来存储模块相关数据
     */
    void *privdata[2];
} aeEventLoop;
```

---

## API 函数说明

### 生命周期管理

```c
// 文件位置：ae.h:96-98

/*
 * aeCreateEventLoop：创建事件循环
 *
 * 参数：
 * - setsize：最多能监听多少个文件描述符
 *
 * 返回值：
 * - 成功：返回 aeEventLoop 指针
 * - 失败：返回 NULL
 *
 * 使用场景：
 * Redis 启动时调用：server.el = aeCreateEventLoop(server.maxclients + CONFIG_FDSET_INCR)
 */
aeEventLoop *aeCreateEventLoop(int setsize);

/*
 * aeDeleteEventLoop：销毁事件循环
 *
 * 会释放所有资源：
 * 1. 关闭 epoll/kqueue fd
 * 2. 释放 events 和 fired 数组
 * 3. 释放所有定时事件（调用 finalizerProc）
 * 4. 释放 eventLoop 本身
 */
void aeDeleteEventLoop(aeEventLoop *eventLoop);

/*
 * aeStop：停止事件循环
 *
 * 设置 eventLoop->stop = 1
 * aeMain() 会在下次循环时退出
 */
void aeStop(aeEventLoop *eventLoop);
```

### 文件事件操作

```c
// 文件位置：ae.h:99-103

/*
 * aeCreateFileEvent：注册文件事件
 *
 * 参数：
 * - eventLoop：事件循环对象
 * - fd：要监听的文件描述符（socket）
 * - mask：事件类型（AE_READABLE、AE_WRITABLE）
 * - proc：回调函数
 * - clientData：自定义数据（通常是 client 结构）
 *
 * 返回值：
 * - AE_OK (0)：成功
 * - AE_ERR (-1)：失败
 *
 * 使用场景：
 * // 监听客户端连接
 * aeCreateFileEvent(server.el, server.ipfd[0], AE_READABLE, acceptTcpHandler, NULL);
 *
 * // 读取客户端命令
 * aeCreateFileEvent(server.el, c->fd, AE_READABLE, readQueryFromClient, c);
 *
 * // 发送响应给客户端
 * aeCreateFileEvent(server.el, c->fd, AE_WRITABLE, sendReplyToClient, c);
 */
int aeCreateFileEvent(aeEventLoop *eventLoop, int fd, int mask,
        aeFileProc *proc, void *clientData);

/*
 * aeDeleteFileEvent：删除文件事件
 *
 * 参数：
 * - eventLoop：事件循环对象
 * - fd：文件描述符
 * - mask：要删除的事件类型
 *
 * 使用场景：
 * // 发送完响应后，删除可写事件（避免不必要的触发）
 * aeDeleteFileEvent(server.el, c->fd, AE_WRITABLE);
 */
void aeDeleteFileEvent(aeEventLoop *eventLoop, int fd, int mask);

/*
 * aeGetFileEvents：获取 fd 上注册的事件
 *
 * 返回值：事件掩码（AE_READABLE、AE_WRITABLE 或组合）
 */
int aeGetFileEvents(aeEventLoop *eventLoop, int fd);

/*
 * aeGetFileClientData：获取 fd 的自定义数据
 *
 * 返回值：注册事件时传入的 clientData
 */
void *aeGetFileClientData(aeEventLoop *eventLoop, int fd);
```

### 定时事件操作

```c
// 文件位置：ae.h:104-107

/*
 * aeCreateTimeEvent：创建定时事件
 *
 * 参数：
 * - eventLoop：事件循环对象
 * - milliseconds：多少毫秒后执行
 * - proc：回调函数
 * - clientData：自定义数据
 * - finalizerProc：清理回调（可以是 NULL）
 *
 * 返回值：
 * - 成功：返回定时事件的 ID（>= 0）
 * - 失败：返回 AE_ERR (-1)
 *
 * 使用场景：
 * // 创建 serverCron 定时任务（每 100ms 执行一次）
 * aeCreateTimeEvent(server.el, 1, serverCron, NULL, NULL);
 */
long long aeCreateTimeEvent(aeEventLoop *eventLoop, long long milliseconds,
        aeTimeProc *proc, void *clientData,
        aeEventFinalizerProc *finalizerProc);

/*
 * aeDeleteTimeEvent：删除定时事件
 *
 * 参数：
 * - eventLoop：事件循环对象
 * - id：定时事件的 ID
 *
 * 返回值：
 * - AE_OK (0)：成功
 * - AE_ERR (-1)：失败（找不到这个 ID）
 *
 * 注意：
 * 不会立即删除，只是标记 id = AE_DELETED_EVENT_ID
 * 下次处理定时事件时才真正释放内存
 */
int aeDeleteTimeEvent(aeEventLoop *eventLoop, long long id);
```

### 事件处理

```c
// 文件位置：ae.h:108-110

/*
 * aeProcessEvents：处理事件（核心函数！）
 *
 * 参数：
 * - eventLoop：事件循环对象
 * - flags：处理标志（见常量定义部分）
 *
 * 返回值：
 * 处理的事件数量
 *
 * 处理流程：
 * 1. 如果有文件事件或定时事件：
 *    a. 调用 beforesleep（如果 flags 包含 AE_CALL_BEFORE_SLEEP）
 *    b. 调用 aeApiPoll() 等待事件（epoll_wait）
 *    c. 调用 aftersleep（如果 flags 包含 AE_CALL_AFTER_SLEEP）
 *    d. 处理触发的文件事件（调用回调函数）
 * 2. 处理到期的定时事件
 * 3. 返回处理的事件数量
 *
 * 使用场景：
 * // 正常的事件循环（aeMain 内部调用）
 * aeProcessEvents(eventLoop, AE_ALL_EVENTS | AE_CALL_BEFORE_SLEEP | AE_CALL_AFTER_SLEEP);
 *
 * // 只处理 I/O 事件，不阻塞
 * aeProcessEvents(eventLoop, AE_FILE_EVENTS | AE_DONT_WAIT);
 */
int aeProcessEvents(aeEventLoop *eventLoop, int flags);

/*
 * aeWait：等待 fd 上的事件（阻塞式）
 *
 * 参数：
 * - fd：文件描述符
 * - mask：要等待的事件（AE_READABLE、AE_WRITABLE）
 * - milliseconds：超时时间（毫秒）
 *
 * 返回值：
 * - 0：超时
 * - >0：触发的事件掩码
 * - -1：错误
 *
 * 使用场景：
 * // 同步等待连接建立（用于主从复制）
 * aeWait(fd, AE_WRITABLE, 1000);
 */
int aeWait(int fd, int mask, long long milliseconds);

/*
 * aeMain：主事件循环（Redis 的心脏！）
 *
 * 这是一个无限循环：
 * while (!eventLoop->stop) {
 *     aeProcessEvents(eventLoop, AE_ALL_EVENTS | AE_CALL_BEFORE_SLEEP | AE_CALL_AFTER_SLEEP);
 * }
 *
 * 使用场景：
 * Redis 启动的最后一步：
 * aeMain(server.el);  // 进入事件循环，永不返回（除非收到 SHUTDOWN 命令）
 */
void aeMain(aeEventLoop *eventLoop);
```

### 配置和查询

```c
// 文件位置：ae.h:111-116

/*
 * aeGetApiName：获取底层 I/O 多路复用库的名称
 *
 * 返回值：
 * - "epoll"（Linux）
 * - "kqueue"（BSD/Mac）
 * - "select"（其他）
 *
 * 使用场景：
 * Redis 启动时打印：
 * serverLog(LL_NOTICE, "Server initialized");
 * serverLog(LL_NOTICE, "The server is now ready to accept connections on port %d", server.port);
 * serverLog(LL_NOTICE, "Running in %s mode", server.cluster_enabled ? "cluster" : "standalone");
 * serverLog(LL_NOTICE, "oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo");
 * serverLog(LL_NOTICE, "Redis version=%s, bits=%d, pid=%d, just started",
 *     REDIS_VERSION, sizeof(long) == 4 ? 32 : 64, getpid());
 * serverLog(LL_NOTICE, "Configuration loaded");
 * serverLog(LL_NOTICE, "Server initialized with %s multiplexing API", aeGetApiName());
 */
char *aeGetApiName(void);

/*
 * aeSetBeforeSleepProc：设置睡眠前回调
 *
 * Redis 在这里做很多重要工作：
 * 1. beforeSleep() 函数会被调用（在 server.c 中定义）
 * 2. 处理客户端输出缓冲区
 * 3. AOF 刷盘
 * 4. 处理异步操作
 */
void aeSetBeforeSleepProc(aeEventLoop *eventLoop, aeBeforeSleepProc *beforesleep);

/*
 * aeSetAfterSleepProc：设置睡眠后回调
 *
 * Redis 很少使用
 */
void aeSetAfterSleepProc(aeEventLoop *eventLoop, aeBeforeSleepProc *aftersleep);

/*
 * aeGetSetSize：获取 setsize（最大 fd 数）
 */
int aeGetSetSize(aeEventLoop *eventLoop);

/*
 * aeResizeSetSize：调整 setsize
 *
 * 参数：
 * - eventLoop：事件循环对象
 * - setsize：新的 setsize
 *
 * 返回值：
 * - AE_OK (0)：成功
 * - AE_ERR (-1)：失败（如果 maxfd >= setsize，不能缩小）
 *
 * 使用场景：
 * 动态调整最大客户端连接数：
 * CONFIG SET maxclients 20000
 */
int aeResizeSetSize(aeEventLoop *eventLoop, int setsize);

/*
 * aeSetDontWait：设置不等待标志
 *
 * 参数：
 * - eventLoop：事件循环对象
 * - noWait：1=不等待，0=正常等待
 *
 * 作用：
 * 设置后，aeProcessEvents() 不会阻塞在 epoll_wait，
 * 而是立即返回（timeout=0）
 *
 * 使用场景：
 * 当有紧急任务需要处理时（如：SHUTDOWN 命令）
 */
void aeSetDontWait(aeEventLoop *eventLoop, int noWait);
```

---

## 典型使用流程

### Redis 启动流程（简化版）

```c
int main(int argc, char **argv) {
    // 1. 初始化服务器配置
    initServerConfig();

    // 2. 创建事件循环
    server.el = aeCreateEventLoop(server.maxclients + CONFIG_FDSET_INCR);

    // 3. 打开监听 socket
    listenToPort(server.port, &server.ipfd_count, server.ipfd);

    // 4. 注册文件事件：监听客户端连接
    for (j = 0; j < server.ipfd_count; j++) {
        aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE,
            acceptTcpHandler, NULL);
    }

    // 5. 创建定时事件：serverCron（每 100ms 执行一次）
    aeCreateTimeEvent(server.el, 1, serverCron, NULL, NULL);

    // 6. 设置睡眠前回调
    aeSetBeforeSleepProc(server.el, beforeSleep);

    // 7. 进入事件循环（永不返回）
    aeMain(server.el);

    // 8. 退出时清理
    aeDeleteEventLoop(server.el);
    return 0;
}
```

### 处理客户端连接流程

```c
// 1. 客户端连接到达，触发 acceptTcpHandler
void acceptTcpHandler(aeEventLoop *el, int fd, void *privdata, int mask) {
    int cfd = accept(fd, ...);  // 接受连接
    client *c = createClient(cfd);  // 创建客户端结构

    // 2. 注册可读事件：等待客户端发送命令
    aeCreateFileEvent(server.el, cfd, AE_READABLE, readQueryFromClient, c);
}

// 3. 客户端发送命令，触发 readQueryFromClient
void readQueryFromClient(aeEventLoop *el, int fd, void *privdata, int mask) {
    client *c = (client*) privdata;
    read(fd, c->querybuf, ...);  // 读取命令
    processInputBuffer(c);  // 解析并执行命令

    // 4. 如果有响应要发送，注册可写事件
    if (c->bufpos > 0 || listLength(c->reply) > 0) {
        aeCreateFileEvent(server.el, c->fd, AE_WRITABLE, sendReplyToClient, c);
    }
}

// 5. 可以发送响应了，触发 sendReplyToClient
void sendReplyToClient(aeEventLoop *el, int fd, void *privdata, int mask) {
    client *c = (client*) privdata;
    write(fd, c->buf, c->bufpos);  // 发送响应

    // 6. 发送完毕，删除可写事件
    aeDeleteFileEvent(server.el, c->fd, AE_WRITABLE);
}
```

### serverCron 定时任务流程

```c
// serverCron：Redis 的主定时任务（每 100ms 执行一次）
int serverCron(struct aeEventLoop *eventLoop, long long id, void *clientData) {
    // 1. 更新统计信息
    run_with_period(100) {
        trackInstantaneousMetric();
    }

    // 2. 清理过期键（每秒 10 次）
    databasesCron();

    // 3. rehash（渐进式）
    incrementallyRehash();

    // 4. AOF 重写（如果需要）
    if (server.aof_rewrite_scheduled) {
        rewriteAppendOnlyFileBackground();
    }

    // 5. RDB 保存（如果需要）
    if (server.rdb_child_pid == -1 && server.aof_child_pid == -1) {
        for (j = 0; j < server.saveparamslen; j++) {
            if (server.dirty >= sp->changes && server.unixtime-server.lastsave > sp->seconds) {
                rdbSaveBackground();
            }
        }
    }

    // 6. 关闭超时的客户端
    clientsCron();

    // 7. 返回 100，表示 100ms 后再次执行
    server.cronloops++;
    return 1000/server.hz;  // 默认 hz=10，所以返回 100ms
}
```

---

## 总结

**ae.h 定义了 Redis 事件循环的核心接口**，包括：

1. **文件事件**：监听 socket 的可读/可写事件（I/O 多路复用）
2. **定时事件**：周期性任务（如 serverCron）
3. **事件循环**：aeEventLoop 结构管理所有事件

**核心流程**：
```
启动 → 创建事件循环 → 注册事件 → 进入 aeMain 无限循环 → 处理事件 → 退出
```

**为什么 Redis 这么快？**
1. **单线程 + I/O 多路复用**：避免线程切换开销
2. **事件驱动**：不阻塞等待，充分利用 CPU
3. **beforesleep 优化**：批量处理输出缓冲区
4. **内存操作为主**：减少磁盘 I/O

**下一步阅读建议**：
- `ae.c`：事件循环的实现
- `ae_epoll.c`：Linux 下的 epoll 实现
- `networking.c`：网络 I/O 处理（readQueryFromClient、sendReplyToClient）
- `server.c`：serverCron 定时任务

---

> 📝 **注意**：这份注释基于 Redis 8.4.0 源码，不同版本可能有细微差异。
