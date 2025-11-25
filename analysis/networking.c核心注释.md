# Redis networking.c 核心详解（中文注释版）

> 📘 **说明**：networking.c 是 Redis 网络层的核心文件（5208行），负责客户端连接管理、命令接收解析、响应发送等所有网络相关操作。本文档重点讲解最关键的部分，帮助你快速理解 Redis 网络层的工作原理。

---

## 📚 目录

1. [文件概述](#文件概述)
2. [核心数据结构](#核心数据结构)
3. [客户端生命周期](#客户端生命周期)
4. [输入处理流程](#输入处理流程)
5. [输出处理流程](#输出处理流程)
6. [缓冲区管理](#缓冲区管理)
7. [完整数据流转图](#完整数据流转图)
8. [重要函数速查表](#重要函数速查表)

---

## 文件概述

`networking.c` 是 Redis 的"网络引擎"，包含：

- **客户端管理**：创建、销毁、连接处理
- **输入处理**：从网络读取命令、解析协议
- **输出处理**：构造响应、发送到网络
- **缓冲区管理**：输入缓冲区（querybuf）、输出缓冲区（buf + reply list）
- **流量控制**：限流、暂停、超时处理
- **协议支持**：RESP2、RESP3 协议

**文件位置**：`src/networking.c`（5208行，185个函数）

**核心思想**：
- Redis 使用**事件驱动**模型处理网络 I/O
- 每个客户端连接对应一个 `client` 结构体
- 读写操作通过回调函数处理（`readQueryFromClient`, `sendReplyToClient`）
- 使用缓冲区优化性能，减少系统调用

---

## 核心数据结构

### 客户端结构 client（重点字段）

```c
// 完整定义在 server.h 中，这里只列出与网络相关的核心字段

typedef struct client {
    /* ===== 网络连接 ===== */
    connection *conn;              // 连接对象（封装了 TCP socket）
    uint64_t id;                   // 客户端唯一 ID
    int tid;                       // 所属 I/O 线程 ID
    int running_tid;               // 当前正在运行的线程 ID

    /* ===== 输入缓冲区（接收客户端命令）===== */
    sds querybuf;                  // 查询缓冲区，存储从网络读取的原始数据
    size_t qb_pos;                 // 当前解析位置
    size_t querybuf_peak;          // 缓冲区峰值大小（用于统计）
    int reqtype;                   // 请求类型：PROTO_REQ_INLINE 或 PROTO_REQ_MULTIBULK
    int multibulklen;              // Multi-bulk 协议：还需要读取多少个参数
    long bulklen;                  // 当前 bulk 参数的长度

    /* ===== 命令参数 ===== */
    int argc;                      // 命令参数数量
    robj **argv;                   // 命令参数数组

    /* ===== 输出缓冲区（发送响应给客户端）===== */
    char *buf;                     // 固定大小的静态缓冲区（16KB）
    size_t buf_usable_size;        // buf 实际可用大小
    int bufpos;                    // buf 当前使用位置
    list *reply;                   // 动态缓冲区链表（当静态缓冲区装不下时使用）
    unsigned long long reply_bytes;// 动态缓冲区总大小
    size_t sentlen;                // 已发送的字节数（部分发送时使用）

    /* ===== 状态标志 ===== */
    uint64_t flags;                // 客户端标志（CLIENT_MASTER, CLIENT_SLAVE 等）
    int io_flags;                  // I/O 标志（读写使能状态）

    /* ===== 时间统计 ===== */
    time_t ctime;                  // 客户端创建时间
    time_t lastinteraction;        // 最后一次交互时间
    time_t obuf_soft_limit_reached_time; // 输出缓冲区达到软限制的时间

    /* ===== 流量统计 ===== */
    long long net_input_bytes;     // 接收的总字节数
    long long net_output_bytes;    // 发送的总字节数

    /* ===== 其他 ===== */
    listNode *client_list_node;    // 在 server.clients 链表中的节点
    listNode *io_thread_client_list_node; // 在 IO 线程链表中的节点
    ClientReplyBlock_listNode clients_pending_write_node; // 待写入链表节点

} client;
```

### 输出缓冲区块结构

```c
// 文件位置：server.h

typedef struct clientReplyBlock {
    size_t size;    // 缓冲区总大小
    size_t used;    // 已使用大小
    char buf[];     // 柔性数组，实际数据存储在这里
} clientReplyBlock;
```

**为什么要同时用静态缓冲区 + 动态链表？**
1. **静态缓冲区（buf）**：
   - 大小固定（16KB）
   - 分配在 client 结构体中，无需额外 malloc
   - 适合小响应（GET 单个键、SET 成功等）

2. **动态链表（reply）**：
   - 用于大响应（LRANGE、KEYS * 等）
   - 每个节点大小至少 16KB
   - 可以扩展到任意大小

这种设计在性能和灵活性之间取得平衡。

---

## 客户端生命周期

###  创建客户端：createClient()

```c
// 文件位置：networking.c:128-248

/*
 * createClient() - 创建一个新的客户端结构
 *
 * 【调用时机】
 * - TCP 连接建立后（由 acceptTcpHandler 调用）
 * - 创建伪客户端（Lua 脚本、AOF 加载等，conn 为 NULL）
 *
 * 【参数】
 * conn - 网络连接对象，可以为 NULL（伪客户端）
 *
 * 【主要步骤】
 * 1. 分配 client 结构体内存
 * 2. 设置 TCP 连接参数（NoDelay, KeepAlive）
 * 3. 注册读事件处理器：connSetReadHandler(conn, readQueryFromClient)
 * 4. 初始化所有字段（缓冲区、命令参数、状态标志等）
 * 5. 分配静态输出缓冲区（16KB）
 * 6. 设置默认数据库（DB 0）
 * 7. 分配全局唯一 ID
 * 8. 初始化认证状态
 * 9. 如果有连接，调用 linkClient() 将客户端加入全局列表
 *
 * 【返回】
 * 新创建的 client 指针
 */
client *createClient(connection *conn) {
    client *c = zmalloc(sizeof(client));

    // 如果有网络连接，设置 TCP 参数和读回调
    if (conn) {
        connEnableTcpNoDelay(conn);               // 禁用 Nagle 算法，减少延迟
        if (server.tcpkeepalive)
            connKeepAlive(conn,server.tcpkeepalive); // 启用 TCP keepalive
        connSetReadHandler(conn, readQueryFromClient); // ⭐ 注册读事件回调
        connSetPrivateData(conn, c);              // 连接对象绑定客户端
    }

    // 分配静态输出缓冲区（16KB）
    c->buf = zmalloc_usable(PROTO_REPLY_CHUNK_BYTES, &c->buf_usable_size);

    // 设置默认数据库
    selectDb(c, 0);

    // 分配全局唯一 ID（原子递增）
    uint64_t client_id;
    atomicGetIncr(server.next_client_id, client_id, 1);
    c->id = client_id;

    // 初始化线程 ID
    c->tid = IOTHREAD_MAIN_THREAD_ID;        // 默认在主线程
    c->running_tid = IOTHREAD_MAIN_THREAD_ID;

    // 初始化输入缓冲区（延迟分配，首次使用时创建）
    c->querybuf = NULL;
    c->qb_pos = 0;
    c->querybuf_peak = 0;

    // 初始化输出缓冲区
    c->bufpos = 0;
    c->reply = listCreate();                 // 动态缓冲区链表
    listSetFreeMethod(c->reply, freeClientReplyValue);
    listSetDupMethod(c->reply, dupClientReplyValue);

    // 初始化命令参数
    c->argc = 0;
    c->argv = NULL;
    c->cmd = c->lastcmd = c->realcmd = NULL;

    // 初始化时间戳
    c->ctime = c->lastinteraction = server.unixtime;

    // 初始化认证状态
    clientSetDefaultAuth(c);

    // 其他字段初始化...（标志、复制状态、PubSub、统计信息等）

    // 加入全局客户端列表
    if (conn) linkClient(c);

    return c;
}
```

### 加入全局列表：linkClient()

```c
// 文件位置：networking.c:97-105

/*
 * linkClient() - 将客户端加入全局链表和索引
 *
 * 【作用】
 * 1. 加入 server.clients 双向链表（用于遍历所有客户端）
 * 2. 加入 server.clients_index rax 树（用于快速根据 ID 查找）
 *
 * 【为什么需要两个数据结构？】
 * - 链表：方便遍历所有客户端（INFO clients, CLIENT LIST 等）
 * - Rax 树：根据 ID 快速查找（CLIENT KILL ID xxx）
 */
void linkClient(client *c) {
    // 加入链表尾部
    listAddNodeTail(server.clients, c);

    // 记住链表节点位置，删除时无需遍历（O(1) 删除）
    c->client_list_node = listLast(server.clients);

    // 加入 rax 索引树（键是 ID，值是 client 指针）
    uint64_t id = htonu64(c->id);  // 转换为网络字节序
    raxInsert(server.clients_index, (unsigned char*)&id, sizeof(id), c, NULL);
}
```

### 销毁客户端：freeClient()

```c
// 文件位置：networking.c:1771-1958

/*
 * freeClient() - 销毁客户端并释放所有资源
 *
 * 【调用时机】
 * - 客户端主动断开连接
 * - 服务器检测到连接异常
 * - 输出缓冲区超限
 * - 超时或违反协议规则
 *
 * 【主要步骤】
 * 1. 检查是否受保护（CLIENT_PROTECTED），如果是则异步释放
 * 2. 如果在 I/O 线程中运行，先从线程中取回
 * 3. 触发模块钩子（REDISMODULE_EVENT_CLIENT_CHANGE）
 * 4. 特殊处理 Master 客户端（缓存状态以便部分重同步）
 * 5. 特殊处理 Slave 客户端（记录日志）
 * 6. 释放输入缓冲区
 * 7. 解除阻塞状态（如果客户端正在 BLPOP 等）
 * 8. 取消所有 WATCH 监视
 * 9. 取消所有 PubSub 订阅
 * 10. 释放输出缓冲区
 * 11. 调用 unlinkClient() 从全局列表中移除
 * 12. 释放 MULTI/EXEC 状态
 * 13. 关闭网络连接
 * 14. 释放 client 结构体
 *
 * 【注意】
 * - 如果是 Master 且连接正常，会缓存状态而不是真正释放（replicationCacheMaster）
 * - 这样可以在重连后进行部分重同步（PSYNC）
 */
void freeClient(client *c) {
    listNode *ln;

    // 如果客户端受保护，使用异步释放
    if (c->flags & CLIENT_PROTECTED) {
        freeClientAsync(c);
        return;
    }

    // 如果客户端在 I/O 线程中，先取回主线程
    if (c->running_tid != IOTHREAD_MAIN_THREAD_ID) {
        fetchClientFromIOThread(c);
    }

    // 从 I/O 线程的事件循环中解绑
    if (c->tid != IOTHREAD_MAIN_THREAD_ID) {
        unbindClientFromIOThreadEventLoop(c);
    }

    // 更新 I/O 线程客户端计数
    if (c->conn) server.io_threads_clients_num[c->tid]--;

    // 触发模块断开事件
    if (c->conn) {
        moduleFireServerEvent(REDISMODULE_EVENT_CLIENT_CHANGE,
                              REDISMODULE_SUBEVENT_CLIENT_CHANGE_DISCONNECTED,
                              c);
    }

    // 从异步释放队列中移除（如果在队列中）
    if (c->flags & CLIENT_CLOSE_ASAP) {
        ln = listSearchKey(server.clients_to_close, c);
        serverAssert(ln != NULL);
        listDelNode(server.clients_to_close, ln);
    }

    // ⭐ 特殊处理：Master 客户端断开连接
    if (server.master && c->flags & CLIENT_MASTER) {
        serverLog(LL_WARNING, "Connection with master lost.");
        // 如果不是协议错误或阻塞状态，缓存 Master 状态
        if (!(c->flags & (CLIENT_PROTOCOL_ERROR|CLIENT_BLOCKED))) {
            c->flags &= ~(CLIENT_CLOSE_ASAP|CLIENT_CLOSE_AFTER_REPLY);
            replicationCacheMaster(c);  // ⭐ 缓存而不是释放！
            return;
        }
    }

    // 记录 Slave 断开日志
    if (clientTypeIsSlave(c)) {
        serverLog(LL_NOTICE, "Connection with replica %s lost.",
            replicationGetSlaveName(c));
    }

    // 释放输入缓冲区
    if (c->io_flags & CLIENT_IO_REUSABLE_QUERYBUFFER)
        resetReusableQueryBuf(c);
    sdsfree(c->querybuf);
    c->querybuf = NULL;

    // 解除阻塞
    if (c->flags & CLIENT_BLOCKED) unblockClient(c, 1);
    dictRelease(c->bstate.keys);

    // 取消 WATCH
    unwatchAllKeys(c);
    listRelease(c->watched_keys);

    // 取消 PubSub 订阅
    pubsubUnsubscribeAllChannels(c, 0);
    pubsubUnsubscribeShardAllChannels(c, 0);
    pubsubUnsubscribeAllPatterns(c, 0);
    dictRelease(c->pubsub_channels);
    dictRelease(c->pubsub_patterns);
    dictRelease(c->pubsubshard_channels);

    // 释放输出缓冲区
    listRelease(c->reply);
    zfree(c->buf);

    // 从全局列表中移除（同时关闭连接）
    unlinkClient(c);

    // 释放 MULTI/EXEC 状态
    freeClientMultiState(c);

    // 其他清理...

    // 最后释放 client 结构体
    zfree(c);
}
```

### 异步释放：freeClientAsync()

```c
// 文件位置：networking.c:1960-1975

/*
 * freeClientAsync() - 异步释放客户端
 *
 * 【为什么需要异步释放？】
 * 有些情况下不能立即释放客户端：
 * 1. 正在事件循环的回调中（不能在回调中删除自己）
 * 2. 正在迭代客户端列表（不能边遍历边删除）
 * 3. 客户端标记为 CLIENT_PROTECTED
 *
 * 【实现方式】
 * 1. 设置 CLIENT_CLOSE_ASAP 标志
 * 2. 加入 server.clients_to_close 队列
 * 3. 在下一次事件循环前（beforeSleep）批量释放
 */
void freeClientAsync(client *c) {
    // 避免重复加入队列
    if (c->flags & CLIENT_CLOSE_ASAP || c->flags & CLIENT_SCRIPT)
        return;

    c->flags |= CLIENT_CLOSE_ASAP;

    // 加入异步释放队列
    listAddNodeTail(server.clients_to_close, c);

    // 如果客户端在 I/O 线程中，需要先取回
    fetchClientFromIOThread(c);
}
```

---

## 输入处理流程

输入处理是 Redis 网络层最复杂的部分之一，主要分为三个步骤：
1. **读取数据**：从 socket 读取到 querybuf
2. **协议解析**：根据 RESP 协议解析出命令和参数
3. **命令执行**：调用 processCommand() 执行命令

### 读取数据：readQueryFromClient()

```c
// 文件位置：networking.c:3177-3312

/*
 * readQueryFromClient() - 从客户端读取查询数据
 *
 * 【调用时机】
 * 当客户端 socket 可读时，事件循环会调用这个函数（通过 connSetReadHandler 注册）
 *
 * 【主要步骤】
 * 1. 检查是否允许读取（CLIENT_IO_READ_ENABLED）
 * 2. 决定读取长度（普通请求 16KB，大参数特殊处理）
 * 3. 准备或复用 querybuf
 * 4. 从 socket 读取数据到 querybuf
 * 5. 更新统计信息（流量、时间戳）
 * 6. 检查缓冲区是否超限
 * 7. 调用 processInputBuffer() 解析命令
 *
 * 【优化技巧】
 * - 使用线程局部变量复用 querybuf（减少内存分配）
 * - 对大参数（>= 32KB）使用精确分配（避免浪费）
 * - 对普通请求使用贪婪分配（减少 read 系统调用次数）
 */
void readQueryFromClient(connection *conn) {
    client *c = connGetPrivateData(conn);
    int nread, big_arg = 0;
    size_t qblen, readlen;

    // 检查是否允许读取
    if (!(c->io_flags & CLIENT_IO_READ_ENABLED)) return;

    c->read_error = 0;

    // 更新 I/O 统计
    atomicIncr(server.stat_io_reads_processed[c->running_tid], 1);

    readlen = PROTO_IOBUF_LEN;  // 默认 16KB

    /* ===== 步骤1：决定读取长度 ===== */

    // 特殊处理：如果正在读取大参数（>= 32KB）
    if (c->reqtype == PROTO_REQ_MULTIBULK && c->multibulklen && c->bulklen != -1
        && c->bulklen >= PROTO_MBULK_BIG_ARG)
    {
        // 为大参数分配独立的 querybuf
        if (!c->querybuf) c->querybuf = sdsempty();

        // 精确计算还需要读取多少字节
        ssize_t remaining = (size_t)(c->bulklen+2)-(sdslen(c->querybuf)-c->qb_pos);
        big_arg = 1;

        if (remaining > 0) readlen = remaining;

        // Master 客户端需要更大的读取缓冲区
        if (c->flags & CLIENT_MASTER && readlen < PROTO_IOBUF_LEN)
            readlen = PROTO_IOBUF_LEN;
    }
    // 普通情况：复用线程局部 querybuf
    else if (c->querybuf == NULL) {
        if (unlikely(thread_reusable_qb_used)) {
            // 复用缓冲区已被占用（嵌套命令执行），分配新的
            c->querybuf = sdsnewlen(NULL, PROTO_IOBUF_LEN);
            sdsclear(c->querybuf);
        } else {
            // 首次使用：创建或分配线程局部复用缓冲区
            if (!thread_reusable_qb) {
                thread_reusable_qb = sdsnewlen(NULL, PROTO_IOBUF_LEN);
                sdsclear(thread_reusable_qb);
            }

            // ⭐ 优化：复用缓冲区，减少内存分配
            c->querybuf = thread_reusable_qb;
            c->io_flags |= CLIENT_IO_REUSABLE_QUERYBUFFER;
            thread_reusable_qb_used = 1;
        }
    }

    /* ===== 步骤2：扩展 querybuf 容量 ===== */

    qblen = sdslen(c->querybuf);

    if (!(c->flags & CLIENT_MASTER) &&
        (big_arg || sdsalloc(c->querybuf) < PROTO_IOBUF_LEN)) {
        // 非贪婪增长（大参数或首次分配）
        c->querybuf = sdsMakeRoomForNonGreedy(c->querybuf, readlen);
        if (c->querybuf_peak < qblen + readlen)
            c->querybuf_peak = qblen + readlen;
    } else {
        // 贪婪增长（尽可能多分配，减少后续扩展）
        c->querybuf = sdsMakeRoomFor(c->querybuf, readlen);
        readlen = sdsavail(c->querybuf);  // 利用所有可用空间
    }

    /* ===== 步骤3：从 socket 读取数据 ===== */

    nread = connRead(c->conn, c->querybuf+qblen, readlen);

    // 处理读取错误
    if (nread == -1) {
        if (connGetState(conn) == CONN_STATE_CONNECTED) {
            goto done;  // 暂时没有数据，稍后再读
        } else {
            c->read_error = CLIENT_READ_CONN_DISCONNECTED;
            freeClientAsync(c);
            goto done;
        }
    } else if (nread == 0) {
        // 客户端关闭连接
        c->read_error = CLIENT_READ_CONN_CLOSED;
        freeClientAsync(c);
        goto done;
    }

    // 更新 SDS 长度
    sdsIncrLen(c->querybuf, nread);
    qblen = sdslen(c->querybuf);
    if (c->querybuf_peak < qblen) c->querybuf_peak = qblen;

    /* ===== 步骤4：更新统计信息 ===== */

    c->lastinteraction = server.unixtime;

    if (c->flags & CLIENT_MASTER) {
        c->read_reploff += nread;  // Master 读取的数据计入复制偏移量
        atomicIncr(server.stat_net_repl_input_bytes, nread);
    } else {
        atomicIncr(server.stat_net_input_bytes, nread);
    }
    c->net_input_bytes += nread;

    /* ===== 步骤5：检查缓冲区是否超限 ===== */

    if (!(c->flags & CLIENT_MASTER) &&
        (c->mstate.argv_len_sums + sdslen(c->querybuf) > server.client_max_querybuf_len ||
         (c->mstate.argv_len_sums + sdslen(c->querybuf) > 1024*1024 && authRequired(c))))
    {
        // 缓冲区超限，断开客户端
        c->read_error = CLIENT_READ_REACHED_MAX_QUERYBUF;
        freeClientAsync(c);
        atomicIncr(server.stat_client_qbuf_limit_disconnections, 1);
        goto done;
    }

    /* ===== 步骤6：解析并执行命令 ===== */

    // ⭐ 关键调用：解析 querybuf 中的命令
    if (processInputBuffer(c) == C_ERR)
        c = NULL;  // 客户端可能已被释放

done:
    // 处理致命读取错误
    if (c && isClientReadErrorFatal(c)) {
        if (c->running_tid == IOTHREAD_MAIN_THREAD_ID) {
            handleClientReadError(c);
        }
    }

    // 重置复用缓冲区
    if (c && (c->io_flags & CLIENT_IO_REUSABLE_QUERYBUFFER)) {
        resetReusableQueryBuf(c);
    }

    beforeNextClient(c);
}
```

### 协议解析：processInputBuffer()

```c
// 文件位置：networking.c:2995-3103

/*
 * processInputBuffer() - 解析输入缓冲区中的命令
 *
 * 【协议类型】
 * Redis 支持两种请求协议：
 * 1. RESP (REdis Serialization Protocol)：二进制安全，用于客户端库
 *    格式：*3\r\n$3\r\nSET\r\n$3\r\nkey\r\n$5\r\nvalue\r\n
 * 2. Inline：简单文本协议，用于 redis-cli 交互
 *    格式：SET key value\r\n
 *
 * 【主要步骤】
 * 1. 检测协议类型（首次读取时）
 * 2. 循环解析所有完整命令
 * 3. 对每个命令调用 processCommandAndResetClient()
 * 4. 如果客户端被释放或暂停，提前退出
 *
 * 【返回】
 * C_OK - 成功
 * C_ERR - 客户端已被释放
 */
int processInputBuffer(client *c) {
    /* 当缓冲区中还有数据时，持续解析 */
    while(c->qb_pos < sdslen(c->querybuf)) {
        /* ===== 步骤1：检测协议类型（首次） ===== */

        if (!c->reqtype) {
            // 根据第一个字符判断协议类型
            if (c->querybuf[c->qb_pos] == '*') {
                c->reqtype = PROTO_REQ_MULTIBULK;  // RESP 协议
            } else {
                c->reqtype = PROTO_REQ_INLINE;     // Inline 协议
            }
        }

        /* ===== 步骤2：根据协议类型解析命令 ===== */

        pendingCommand *pcmd = NULL;

        if (c->reqtype == PROTO_REQ_INLINE) {
            // 解析 Inline 协议（简单的空格分隔）
            if (processInlineBuffer(c, pcmd) != C_OK) break;
        } else if (c->reqtype == PROTO_REQ_MULTIBULK) {
            // 解析 RESP 协议（复杂的二进制协议）
            if (processMultibulkBuffer(c, pcmd) != C_OK) break;
        } else {
            serverPanic("Unknown request type");
        }

        /* ===== 步骤3：执行命令 ===== */

        // 如果解析出完整命令（argc > 0），执行它
        if (c->argc) {
            // ⭐ 执行命令并重置客户端状态
            if (processCommandAndResetClient(c) == C_ERR) {
                // 客户端已被释放
                return C_ERR;
            }
        }

        /* ===== 步骤4：检查是否需要暂停 ===== */

        // 如果客户端被标记为暂停，停止解析
        if (c->flags & CLIENT_BLOCKED) break;

        // 如果客户端被标记为关闭，停止解析
        if (c->flags & CLIENT_CLOSE_ASAP) break;
    }

    /* ===== 步骤5：清理已解析的数据 ===== */

    // 如果使用复用缓冲区，必须清空已解析部分
    if (c->io_flags & CLIENT_IO_REUSABLE_QUERYBUFFER) {
        serverAssert(c->qb_pos == 0);  // 必须完全解析
    }
    // 否则，可以保留未解析部分
    else if (c->qb_pos) {
        sdsrange(c->querybuf, c->qb_pos, -1);
        c->qb_pos = 0;
    }

    return C_OK;
}
```

### RESP 协议解析（简化说明）

RESP 协议是 Redis 的标准协议，格式如下：

```
命令示例：SET mykey myvalue

编码后：
*3\r\n          // * 表示数组，3 表示有 3 个元素
$3\r\n          // $ 表示字符串，3 表示长度为 3
SET\r\n         // 命令名
$5\r\n          // $ 表示字符串，5 表示长度为 5
mykey\r\n       // 键名
$7\r\n          // $ 表示字符串，7 表示长度为 7
myvalue\r\n     // 键值
```

解析过程：
1. 读取 `*3\r\n`，知道有 3 个参数
2. 读取 `$3\r\n`，知道第一个参数长度为 3
3. 读取 `SET\r\n`，第一个参数是 "SET"
4. 依次读取剩余参数...
5. 解析完成后，`argc=3`, `argv=["SET", "mykey", "myvalue"]`

---

## 输出处理流程

输出处理分为两个阶段：
1. **构造响应**：将响应数据写入输出缓冲区
2. **发送响应**：将缓冲区数据发送到网络

### 构造响应：addReply*() 系列函数

Redis 提供了丰富的 API 来构造各种类型的响应：

```c
// 基础函数

void addReply(client *c, robj *obj);              // 添加 Redis 对象
void addReplyProto(client *c, const char *s, size_t len); // 添加原始协议数据
void addReplySds(client *c, sds s);               // 添加 SDS 字符串（会释放）

// 特定类型响应

void addReplyError(client *c, const char *err);   // 错误：-ERR message\r\n
void addReplyStatus(client *c, const char *status); // 状态：+OK\r\n
void addReplyLongLong(client *c, long long ll);   // 整数：:123\r\n
void addReplyNull(client *c);                     // 空值：$-1\r\n (RESP2) 或 _\r\n (RESP3)
void addReplyBool(client *c, int b);              // 布尔：#t\r\n 或 #f\r\n (RESP3)
void addReplyDouble(client *c, double d);         // 浮点：,3.14\r\n (RESP3)

// Bulk 字符串

void addReplyBulk(client *c, robj *obj);          // $<len>\r\n<data>\r\n
void addReplyBulkCBuffer(client *c, const void *p, size_t len);
void addReplyBulkCString(client *c, const char *s);

// 数组/集合/哈希

void addReplyArrayLen(client *c, long length);    // *<len>\r\n
void addReplyMapLen(client *c, long length);      // %<len>\r\n (RESP3)
void addReplySetLen(client *c, long length);      // ~<len>\r\n (RESP3)

// 延迟长度（用于不知道长度的情况）

void *addReplyDeferredLen(client *c);             // 返回占位符
void setDeferredArrayLen(client *c, void *node, long length); // 填充真实长度
```

**使用示例**：

```c
// 示例1：简单响应
addReplyStatus(c, "OK");  // 输出：+OK\r\n

// 示例2：整数响应
addReplyLongLong(c, 123);  // 输出：:123\r\n

// 示例3：字符串响应
addReplyBulkCString(c, "Hello");  // 输出：$5\r\nHello\r\n

// 示例4：数组响应
addReplyArrayLen(c, 2);           // 输出：*2\r\n
addReplyBulkCString(c, "foo");    // 输出：$3\r\nfoo\r\n
addReplyBulkCString(c, "bar");    // 输出：$3\r\nbar\r\n

// 示例5：延迟长度（用于 LRANGE 等不知道返回多少元素的命令）
void *replylen = addReplyDeferredLen(c);  // 先占位
long count = 0;
// ... 循环添加元素 ...
addReplyBulkCString(c, "item1"); count++;
addReplyBulkCString(c, "item2"); count++;
setDeferredArrayLen(c, replylen, count);  // 最后填充真实长度
```

### 底层实现：_addReplyToBufferOrList()

```c
// 文件位置：networking.c:406-453

/*
 * _addReplyToBufferOrList() - 将数据添加到输出缓冲区
 *
 * 【策略】
 * 1. 优先使用静态缓冲区（c->buf，16KB）
 * 2. 如果静态缓冲区满了，使用动态链表（c->reply）
 *
 * 【为什么这样设计？】
 * - 静态缓冲区：快速、无内存分配，适合小响应（大部分情况）
 * - 动态链表：灵活、无限容量，适合大响应（LRANGE、KEYS * 等）
 */
void _addReplyToBufferOrList(client *c, const char *s, size_t len) {
    // 如果客户端即将关闭，不添加数据
    if (c->flags & CLIENT_CLOSE_AFTER_REPLY) return;

    // Replica 不应该产生响应（如果有，说明出错了）
    if (unlikely(clientTypeIsSlave(c))) {
        logInvalidUseAndFreeClientAsync(c, "Replica generated a reply");
        return;
    }

    // 更新流量统计
    c->net_output_bytes_curr_cmd += len;

    /* ===== 特殊处理：PUSH 消息 ===== */

    // 如果是 PUSH 消息（PubSub 通知），可能需要延迟发送
    if ((c->flags & CLIENT_PUSHING) && c == server.current_client &&
        server.executing_client && !cmdHasPushAsReply(server.executing_client->cmd))
    {
        // 将 PUSH 消息暂存到 pending_push_messages，命令执行完后再发送
        _addReplyProtoToList(c, server.pending_push_messages, s, len);
        return;
    }

    /* ===== 步骤1：尝试使用静态缓冲区 ===== */

    const size_t available = c->buf_usable_size - c->bufpos;

    size_t reply_len = 0;

    // 只有当动态链表为空时，才能使用静态缓冲区
    if (listLength(c->reply) < 1) {
        reply_len = len > available ? available : len;
        memcpy(c->buf + c->bufpos, s, reply_len);
        c->bufpos += reply_len;

        // 更新峰值
        c->buf_peak = max(c->buf_peak, (size_t)c->bufpos);
    }

    /* ===== 步骤2：剩余数据使用动态链表 ===== */

    if (len > reply_len) {
        _addReplyProtoToList(c, c->reply, s + reply_len, len - reply_len);
    }
}
```

### 发送响应：writeToClient()

```c
// 文件位置：networking.c:2130-2286

/*
 * writeToClient() - 将输出缓冲区数据发送到客户端
 *
 * 【调用时机】
 * 1. beforeSleep() 中调用 handleClientsWithPendingWrites()
 * 2. socket 可写时，事件循环调用 sendReplyToClient()
 *
 * 【参数】
 * c - 客户端
 * handler_installed - 是否已安装写事件处理器
 *   0: 在 beforeSleep 中同步调用，如果写不完需要安装处理器
 *   1: 在事件循环中异步调用，已有处理器
 *
 * 【主要步骤】
 * 1. 检查是否有数据要发送
 * 2. 循环发送：先发静态缓冲区，再发动态链表
 * 3. 使用 writev() 批量发送（减少系统调用）
 * 4. 更新统计信息
 * 5. 检查是否发送完成，决定是否保留写事件处理器
 *
 * 【返回】
 * C_OK - 成功（可能还有未发送数据）
 * C_ERR - 连接断开，客户端已释放
 */
int writeToClient(client *c, int handler_installed) {
    ssize_t nwritten = 0, totwritten = 0;

    /* ===== 步骤1：检查是否有数据 ===== */

    if (!clientHasPendingReplies(c)) {
        // 没有数据要发送
        if (!handler_installed) return C_OK;

        // 已安装写处理器但没数据，移除处理器
        connSetWriteHandler(c->conn, NULL);
        return C_OK;
    }

    /* ===== 步骤2：发送数据 ===== */

    while(clientHasPendingReplies(c)) {
        // 根据客户端类型选择不同的写函数
        if (clientTypeIsSlave(c)) {
            if (_writeToClientSlave(c, &nwritten) == C_ERR) return C_ERR;
        } else {
            if (_writeToClientNonSlave(c, &nwritten) == C_ERR) return C_ERR;
        }

        if (nwritten == 0) break;  // socket 缓冲区满，稍后再发
        totwritten += nwritten;

        /* 限制单次发送量，避免阻塞太久 */
        if (totwritten > NET_MAX_WRITES_PER_EVENT &&
            (server.maxmemory == 0 ||
             zmalloc_used_memory() < server.maxmemory) &&
            !(c->flags & CLIENT_SLAVE)) break;
    }

    /* ===== 步骤3：更新统计 ===== */

    atomicIncr(server.stat_net_output_bytes, totwritten);
    c->net_output_bytes += totwritten;

    /* ===== 步骤4：检查是否发送完成 ===== */

    if (!clientHasPendingReplies(c)) {
        // 发送完成
        c->sentlen = 0;

        // 移除写事件处理器（如果已安装）
        if (handler_installed) connSetWriteHandler(c->conn, NULL);

        // 如果标记为 CLIENT_CLOSE_AFTER_REPLY，关闭客户端
        if (c->flags & CLIENT_CLOSE_AFTER_REPLY) {
            freeClientAsync(c);
            return C_ERR;
        }
    }

    return C_OK;
}
```

### 实际发送：_writeToClientNonSlave()

```c
// 文件位置：networking.c:2042-2126（简化版）

/*
 * _writeToClientNonSlave() - 实际发送数据（非 Slave 客户端）
 *
 * 【优化】
 * 使用 writev() 批量发送多个缓冲区，减少系统调用次数
 *
 * 【发送顺序】
 * 1. 静态缓冲区（c->buf）
 * 2. 动态链表的第一个节点
 * 3. 动态链表的第二个节点
 * ... 最多一次发送 IOV_MAX 个缓冲区
 */
static inline int _writeToClientNonSlave(client *c, ssize_t *nwritten) {
    *nwritten = 0;

    /* ===== 步骤1：准备 iovec 数组（用于 writev）===== */

    struct iovec iov[IOV_MAX];
    int iovcnt = 0;
    size_t iov_bytes_len = 0;

    // 添加静态缓冲区
    if (c->bufpos > 0) {
        iov[iovcnt].iov_base = c->buf + c->sentlen;
        iov[iovcnt].iov_len = c->bufpos - c->sentlen;
        iov_bytes_len += iov[iovcnt].iov_len;
        iovcnt++;
    }

    // 添加动态链表节点
    listIter li;
    listNode *ln;
    listRewind(c->reply, &li);

    while((ln = listNext(&li)) && iovcnt < IOV_MAX) {
        clientReplyBlock *block = listNodeValue(ln);

        // 跳过已发送的部分
        if (block->used == 0) continue;

        size_t send_len = block->used;
        if (ln == listFirst(c->reply)) {
            send_len -= c->sentlen;  // 首个节点可能已部分发送
        }

        iov[iovcnt].iov_base = block->buf + (block->used - send_len);
        iov[iovcnt].iov_len = send_len;
        iov_bytes_len += send_len;
        iovcnt++;
    }

    /* ===== 步骤2：发送数据 ===== */

    if (iovcnt == 0) return C_OK;  // 没有数据

    *nwritten = connWritev(c->conn, iov, iovcnt);

    if (*nwritten <= 0) {
        // 发送失败
        if (*nwritten == -1 && connGetState(c->conn) != CONN_STATE_CONNECTED) {
            freeClientAsync(c);
            return C_ERR;
        }
        return C_OK;  // 暂时不可写，稍后再试
    }

    /* ===== 步骤3：更新发送进度 ===== */

    ssize_t remaining = *nwritten;

    // 更新静态缓冲区
    if (c->bufpos > 0) {
        if (remaining >= (ssize_t)(c->bufpos - c->sentlen)) {
            // 静态缓冲区全部发送完成
            remaining -= (c->bufpos - c->sentlen);
            c->bufpos = 0;
            c->sentlen = 0;
        } else {
            // 部分发送
            c->sentlen += remaining;
            remaining = 0;
        }
    }

    // 更新动态链表
    while (remaining > 0 && listLength(c->reply) > 0) {
        listNode *ln = listFirst(c->reply);
        clientReplyBlock *block = listNodeValue(ln);

        size_t send_len = block->used - c->sentlen;

        if (remaining >= (ssize_t)send_len) {
            // 当前节点全部发送完成，删除节点
            remaining -= send_len;
            listDelNode(c->reply, ln);
            c->reply_bytes -= block->size;
            c->sentlen = 0;
        } else {
            // 部分发送
            c->sentlen += remaining;
            remaining = 0;
        }
    }

    return C_OK;
}
```

---

## 缓冲区管理

### 输入缓冲区（querybuf）

**作用**：暂存从网络读取但尚未解析的数据

**优化策略**：
1. **复用线程局部缓冲区**：
   - 每个 I/O 线程维护一个复用缓冲区（`thread_reusable_qb`）
   - 普通请求优先使用复用缓冲区，避免频繁分配/释放
   - 解析完成后立即清空，供下一个客户端使用

2. **大参数独立分配**：
   - 如果参数 >= 32KB，使用独立的 querybuf
   - 精确分配所需大小，避免浪费

3. **贪婪 vs 非贪婪增长**：
   - 普通情况：贪婪增长（尽可能多分配，减少后续扩展）
   - 大参数：非贪婪增长（精确分配，避免浪费）

4. **限制最大大小**：
   - 默认限制 1GB（`server.client_max_querybuf_len`）
   - 超限后断开客户端，防止内存耗尽攻击

**示例**：

```c
// 普通请求（使用复用缓冲区）
客户端1: SET key1 value1
  └─> 使用 thread_reusable_qb (16KB)
  └─> 解析完成后清空
客户端2: GET key2
  └─> 复用同一个 thread_reusable_qb
  └─> 解析完成后清空

// 大参数（独立分配）
客户端3: SET bigkey <32KB 的值>
  └─> 分配独立的 querybuf（精确 32KB + 头部）
  └─> 解析完成后释放
```

### 输出缓冲区（buf + reply）

**作用**：暂存要发送给客户端的响应数据

**两级结构**：

1. **静态缓冲区（buf）**：
   - 大小：16KB
   - 位置：在 client 结构体中
   - 优点：快速、无内存分配
   - 适用：小响应（大部分情况）

2. **动态链表（reply）**：
   - 节点大小：至少 16KB
   - 位置：独立分配
   - 优点：无限容量
   - 适用：大响应（LRANGE、KEYS * 等）

**使用策略**：

```c
// 小响应：只用静态缓冲区
GET mykey
  └─> 响应：$5\r\nvalue\r\n (11 字节)
  └─> 写入 c->buf
  └─> c->bufpos = 11

// 中等响应：静态缓冲区 + 少量动态节点
LRANGE mylist 0 100
  └─> 前 16KB 写入 c->buf
  └─> 剩余部分写入 c->reply 链表

// 大响应：主要用动态链表
KEYS *（返回 100 万个键）
  └─> 第一个 16KB 可能写入 c->buf
  └─> 后续所有数据写入 c->reply 链表
```

**限制机制**：

```c
// 输出缓冲区限制（防止慢客户端耗尽内存）

typedef struct clientBufferLimitsConfig {
    unsigned long long hard_limit_bytes;  // 硬限制（立即断开）
    unsigned long long soft_limit_bytes;  // 软限制
    time_t soft_limit_seconds;            // 软限制持续时间
} clientBufferLimitsConfig;

// 默认配置
client-output-buffer-limit normal 0 0 0        // 普通客户端：无限制
client-output-buffer-limit replica 256mb 64mb 60  // Replica：硬限256MB，软限64MB持续60秒
client-output-buffer-limit pubsub 32mb 8mb 60     // PubSub：硬限32MB，软限8MB持续60秒
```

---

## 完整数据流转图

```
┌─────────────────────────────────────────────────────────────────────┐
│                       Redis 网络层数据流转                           │
└─────────────────────────────────────────────────────────────────────┘

【输入流程】客户端 → Redis

1. TCP 连接建立
   ↓
   acceptTcpHandler()              // ae.c 事件循环检测到新连接
   ↓
   acceptCommonHandler()           // networking.c 接受连接
   ↓
   createClient(conn)              // 创建 client 结构
   ↓
   connSetReadHandler(conn, readQueryFromClient)  // 注册读事件

2. 客户端发送命令
   ↓
   [事件循环检测到 socket 可读]
   ↓
   readQueryFromClient(conn)       // ⭐ 读取数据
   │
   ├─> connRead(conn, querybuf, readlen)  // 从 socket 读取到 querybuf
   │
   └─> processInputBuffer(c)       // ⭐ 解析命令
       │
       ├─> [检测协议类型] RESP or Inline
       │
       ├─> processMultibulkBuffer(c)  // 解析 RESP 协议
       │   └─> 解析出：argc, argv[]
       │
       └─> processCommandAndResetClient(c)  // ⭐ 执行命令
           └─> processCommand(c)        // server.c
               └─> call(c, CMD_CALL_FULL)  // server.c
                   └─> c->cmd->proc(c)     // 实际的命令函数（如 setCommand）

【输出流程】Redis → 客户端

1. 命令执行中构造响应
   ↓
   setCommand(c) / getCommand(c) / ... // 各种命令实现
   ↓
   addReply*(c, ...)                   // ⭐ 构造响应
   │
   ├─> prepareClientToWrite(c)        // 检查是否可以写
   │   └─> putClientInPendingWriteQueue(c)  // 加入待写队列
   │
   └─> _addReplyToBufferOrList(c, s, len)   // ⭐ 添加数据到缓冲区
       │
       ├─> [优先] 写入静态缓冲区 c->buf (16KB)
       │
       └─> [满了] 写入动态链表 c->reply

2. 事件循环前发送响应（优化：避免安装写事件）
   ↓
   beforeSleep()                       // server.c，每次事件循环前调用
   ↓
   handleClientsWithPendingWrites()   // ⭐ 处理待写客户端
   │
   └─> writeToClient(c, 0)            // ⭐ 同步发送
       │
       ├─> [如果发送完成] 完成，无需写事件
       │
       └─> [如果未发送完] installClientWriteHandler(c)
           └─> connSetWriteHandler(conn, sendReplyToClient)

3. socket 可写时继续发送（如果第2步未发送完）
   ↓
   [事件循环检测到 socket 可写]
   ↓
   sendReplyToClient(conn)            // 写事件回调
   ↓
   writeToClient(c, 1)                // ⭐ 异步发送
   │
   ├─> connWritev(conn, iov[], iovcnt)  // 使用 writev 批量发送
   │
   ├─> [更新发送进度] c->sentlen, c->bufpos
   │
   └─> [如果发送完成] connSetWriteHandler(conn, NULL)  // 移除写事件
```

### 关键路径总结

**正常请求处理路径**：
```
客户端发送命令
  → readQueryFromClient()        // 读取数据
  → processInputBuffer()         // 解析协议
  → processCommand()             // 执行命令（server.c）
  → addReply*()                  // 构造响应
  → beforeSleep()                // 事件循环前
  → handleClientsWithPendingWrites()  // 发送响应
  → writeToClient()              // 写入 socket
  → 完成
```

**大响应处理路径**（单次发送不完）：
```
命令执行
  → addReply*()                  // 构造大量数据
  → beforeSleep()
  → writeToClient() 部分发送    // 发送部分数据
  → installClientWriteHandler()  // 安装写事件
  → [事件循环] socket 可写
  → sendReplyToClient()          // 继续发送
  → writeToClient()              // 再次发送
  → [可能多次] 直到发送完成
  → 移除写事件
```

---

## 重要函数速查表

### 客户端生命周期

| 函数 | 行号 | 作用 | 调用时机 |
|------|------|------|----------|
| `createClient()` | 128 | 创建客户端结构 | TCP 连接建立、创建伪客户端 |
| `linkClient()` | 97 | 加入全局链表和索引 | createClient() 内部调用 |
| `freeClient()` | 1771 | 销毁客户端 | 连接断开、超时、错误 |
| `freeClientAsync()` | 1960 | 异步销毁客户端 | 在回调中释放客户端 |
| `unlinkClient()` | 1584 | 从全局链表中移除 | freeClient() 内部调用 |

### 输入处理

| 函数 | 行号 | 作用 | 调用时机 |
|------|------|------|----------|
| `readQueryFromClient()` | 3177 | 从 socket 读取数据 | socket 可读时（事件回调） |
| `processInputBuffer()` | 2995 | 解析输入缓冲区 | readQueryFromClient() 内部 |
| `processMultibulkBuffer()` | 2588 | 解析 RESP 协议 | processInputBuffer() 内部 |
| `processInlineBuffer()` | 2448 | 解析 Inline 协议 | processInputBuffer() 内部 |
| `processCommandAndResetClient()` | 2860 | 执行命令并重置 | processInputBuffer() 内部 |

### 输出处理

| 函数 | 行号 | 作用 | 调用时机 |
|------|------|------|----------|
| `prepareClientToWrite()` | 350 | 准备写入（加入待写队列） | 所有 addReply*() 内部调用 |
| `_addReplyToBufferOrList()` | 406 | 添加数据到输出缓冲区 | 所有 addReply*() 内部调用 |
| `addReply()` | 461 | 添加 robj 对象 | 命令实现中 |
| `addReplyError()` | 644 | 添加错误响应 | 命令实现中 |
| `addReplyStatus()` | 720 | 添加状态响应 | 命令实现中 |
| `addReplyLongLong()` | 1030 | 添加整数响应 | 命令实现中 |
| `addReplyBulk()` | 1184 | 添加字符串响应 | 命令实现中 |
| `addReplyArrayLen()` | 1053 | 添加数组头 | 命令实现中 |
| `handleClientsWithPendingWrites()` | 2297 | 批量发送待写客户端 | beforeSleep() 中 |
| `writeToClient()` | 2130 | 发送输出缓冲区 | 同步或异步调用 |
| `sendReplyToClient()` | 2288 | 写事件回调 | socket 可写时 |

### 缓冲区管理

| 函数 | 行号 | 作用 | 调用时机 |
|------|------|------|----------|
| `resetReusableQueryBuf()` | 2954 | 重置复用缓冲区 | 命令解析完成后 |
| `_addReplyProtoToList()` | 360 | 添加数据到动态链表 | 静态缓冲区满时 |
| `setDeferredArrayLen()` | 883 | 设置延迟数组长度 | 不知道长度时先占位 |
| `trimReplyUnusedTailSpace()` | 736 | 裁剪未使用空间 | 减少内存碎片 |

### 连接管理

| 函数 | 行号 | 作用 | 调用时机 |
|------|------|------|----------|
| `acceptTcpHandler()` | 4221 | 接受 TCP 连接 | 监听 socket 可读 |
| `acceptCommonHandler()` | 4144 | 通用连接处理 | acceptTcpHandler() 内部 |
| `disconnectSlaves()` | 1543 | 断开所有从服务器 | 主从切换、关机 |
| `pauseClients()` | 4652 | 暂停所有客户端 | CLIENT PAUSE 命令 |
| `unpauseClients()` | 4707 | 恢复所有客户端 | 暂停超时 |

### 协议处理

| 函数 | 行号 | 作用 | 调用时机 |
|------|------|------|----------|
| `setProtocolError()` | 2408 | 设置协议错误 | 解析失败时 |
| `resetClientQbufState()` | 2337 | 重置解析状态 | 命令执行完成 |

### I/O 线程相关

| 函数 | 行号 | 作用 | 调用时机 |
|------|------|------|----------|
| `assignClientToIOThread()` | N/A | 分配客户端到 I/O 线程 | handleClientsWithPendingWrites() |
| `fetchClientFromIOThread()` | N/A | 从 I/O 线程取回客户端 | 需要在主线程处理时 |

---

## 附录：RESP 协议快速参考

### RESP2 数据类型

| 类型 | 首字符 | 格式 | 示例 |
|------|--------|------|------|
| 简单字符串 | `+` | `+OK\r\n` | `+OK\r\n` |
| 错误 | `-` | `-ERR message\r\n` | `-ERR unknown command\r\n` |
| 整数 | `:` | `:123\r\n` | `:1000\r\n` |
| 批量字符串 | `$` | `$<len>\r\n<data>\r\n` | `$5\r\nhello\r\n` |
| 数组 | `*` | `*<count>\r\n<elements>` | `*2\r\n$3\r\nfoo\r\n$3\r\nbar\r\n` |
| 空值 | `$` | `$-1\r\n` | `$-1\r\n` |

### RESP3 新增类型

| 类型 | 首字符 | 格式 | 示例 |
|------|--------|------|------|
| 空值 | `_` | `_\r\n` | `_\r\n` |
| 布尔 | `#` | `#t\r\n` 或 `#f\r\n` | `#t\r\n` |
| 浮点 | `,` | `,3.14\r\n` | `,3.14159\r\n` |
| 大数 | `(` | `(123456789012345\r\n` | `(123456789012345\r\n` |
| 批量错误 | `!` | `!<len>\r\n<error>\r\n` | `!21\r\nSYNTAX error\r\n` |
| 字典 | `%` | `%<count>\r\n<key-val>` | `%2\r\n$3\r\nfoo\r\n:1\r\n$3\r\nbar\r\n:2\r\n` |
| 集合 | `~` | `~<count>\r\n<elements>` | `~3\r\n:1\r\n:2\r\n:3\r\n` |
| 属性 | `\|` | `\|<count>\r\n<key-val>` | `\|1\r\n$9\r\ndata-type\r\n$6\r\nstring\r\n` |
| 推送 | `>` | `><count>\r\n<elements>` | `>3\r\n$7\r\nmessage\r\n...\r\n` |

---

## 总结

`networking.c` 是 Redis 网络层的核心，实现了：

1. **客户端管理**：创建、销毁、链接管理
2. **输入处理**：读取 → 解析 → 执行
3. **输出处理**：构造 → 缓冲 → 发送
4. **缓冲区优化**：两级缓冲、复用机制、批量发送
5. **协议支持**：RESP2、RESP3、Inline

**关键设计思想**：
- 事件驱动：高并发、低延迟
- 缓冲优化：减少系统调用、内存分配
- 分离读写：异步处理、流水线
- 限制保护：防止慢客户端、内存攻击

**性能优化点**：
- 复用输入缓冲区（避免频繁分配）
- 两级输出缓冲区（静态 + 动态）
- writev 批量发送（减少系统调用）
- beforeSleep 批处理（避免安装写事件）
- I/O 线程（多线程并发处理）

**学习建议**：
1. 先理解客户端生命周期（创建 → 读 → 解析 → 写 → 销毁）
2. 跟踪一个简单命令的完整流程（如 `SET key value`）
3. 研究缓冲区管理策略（何时用静态、何时用动态）
4. 对比 RESP2 和 RESP3 协议差异
5. 结合 server.c 理解命令执行流程

---

📘 **相关文件**：
- `src/server.h` - 客户端结构定义
- `src/server.c` - 命令执行、事件循环
- `src/ae.c` - 事件驱动引擎
- `src/connection.c` - 连接抽象层
- `src/sds.c` - 动态字符串实现

---

*文档生成时间：2025-11-24*
*基于 Redis 源码版本：unstable (commit b0694f1)*
*作者：Claude Code*
