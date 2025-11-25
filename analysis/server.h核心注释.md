# Redis server.h 核心结构详解（中文注释版）

> 📘 **说明**：server.h 文件太大（4366行），完整注释不太实用。这份文档重点讲解最核心的部分，帮助你快速理解 Redis 内部结构。

---

## 📚 目录

1. [文件概述](#文件概述)
2. [基础类型定义](#基础类型定义)
3. [核心对象：redisObject](#核心对象redisobject)
4. [特殊对象：kvobj](#特殊对象kvobj)
5. [客户端结构：client](#客户端结构client)
6. [全局服务器状态：redisServer](#全局服务器状态redisserver)
7. [数据库结构：redisDb](#数据库结构redisdb)
8. [命令结构：redisCommand](#命令结构rediscommand)
9. [重要宏定义](#重要宏定义)
10. [如何阅读完整源码](#如何阅读完整源码)

---

## 文件概述

`server.h` 是 Redis 最核心的头文件，包含了：

- **3个超级核心结构**：`redisObject`, `client`, `redisServer`
- **100+ 个其他结构体**
- **500+ 个宏定义和常量**
- **1000+ 个函数声明**

**文件位置**：`src/server.h`（4366行）

**核心思想**：
- Redis 是一个**单线程事件驱动**服务器
- 所有数据都包装成 `redisObject`
- 每个连接都是一个 `client` 结构
- 整个服务器状态存在一个全局变量 `server` 中

---

## 基础类型定义

```c
// 文件位置：server.h:45-46

/* Redis 内部使用的时间类型 */
typedef long long mstime_t;  /* 毫秒时间类型 (millisecond time) */
typedef long long ustime_t;  /* 微秒时间类型 (microsecond time) */
```

**为什么需要这两个类型？**
- Redis 需要高精度计时（LRU 淘汰、超时检测、性能统计）
- 用 `long long` 保证在 32位/64位系统上都够用
- 毫秒：用于超时、过期时间（粒度足够且节省空间）
- 微秒：用于性能统计（更精确）

---

## 核心对象：redisObject

```c
// 文件位置：server.h:1057-1068

/*
 * redisObject - Redis 最核心的结构之一！
 *
 * 【作用】
 * 所有存储在 Redis 中的"值"都会被包装成这个结构
 * 比如执行 SET mykey "hello"，"hello" 会被包装成一个 redisObject
 *
 * 【为什么要包装？】
 * 1. 统一处理：字符串、列表、哈希等都是 robj，代码更简洁
 * 2. 引用计数：多个键可以共享同一个对象（比如整数 0-9999）
 * 3. 携带元信息：类型、编码方式、LRU 时间等
 *
 * 【内存布局】
 * 总共 16 字节（64位系统）：
 * - 4位 type（类型）
 * - 4位 encoding（编码）
 * - 24位 lru（LRU 时间或 LFU 频率）
 * - 1位 iskvobj（是否是 kvobj）
 * - 1位 expirable（是否有过期时间）
 * - 30位 refcount（引用计数）
 * - 8字节 ptr（指向实际数据）
 */
struct redisObject {
    unsigned type:4;        /* 数据类型：OBJ_STRING(0), OBJ_LIST(1), OBJ_SET(2), OBJ_ZSET(3), OBJ_HASH(4) */
    unsigned encoding:4;    /* 编码方式：OBJ_ENCODING_RAW, OBJ_ENCODING_INT, OBJ_ENCODING_HT 等 */
    unsigned lru:LRU_BITS;  /* LRU 时间（相对于全局 lru_clock）或 LFU 数据（8位频率 + 16位访问时间）*/
    unsigned iskvobj : 1;   /* 1 表示这是一个 kvobj（嵌入了键名的对象）*/
    unsigned expirable : 1; /* 1 表示有过期时间（此时这个对象必须是 kvobj）*/
    unsigned refcount : OBJ_REFCOUNT_BITS; /* 引用计数：多少个地方在用这个对象 */
    void *ptr;             /* 指向实际数据：可能是 sds 字符串、list、dict、zset 等 */
};
```

###  type 字段（对象类型）

```c
// 文件位置：server.h:815-835

#define OBJ_STRING 0    /* 字符串：SET key "value" */
#define OBJ_LIST 1      /* 列表：LPUSH mylist "item" */
#define OBJ_SET 2       /* 集合：SADD myset "member" */
#define OBJ_ZSET 3      /* 有序集合：ZADD myzset 1.0 "member" */
#define OBJ_HASH 4      /* 哈希：HSET myhash field "value" */
#define OBJ_MODULE 5    /* 模块类型：自定义类型 */
#define OBJ_STREAM 6    /* 流：XADD mystream * field value */
```

### encoding 字段（编码方式）

```c
// 文件位置：server.h:1034-1046

/* 同一种类型可以用不同的底层结构存储，以优化内存和性能 */

#define OBJ_ENCODING_RAW 0       /* 原始：sds 字符串 */
#define OBJ_ENCODING_INT 1       /* 整数：long 类型 */
#define OBJ_ENCODING_HT 2        /* 哈希表：dict */
#define OBJ_ENCODING_ZIPMAP 3    /* 已废弃 */
#define OBJ_ENCODING_LINKEDLIST 4 /* 已废弃 */
#define OBJ_ENCODING_ZIPLIST 5   /* 已废弃 */
#define OBJ_ENCODING_INTSET 6    /* 整数集合：intset */
#define OBJ_ENCODING_SKIPLIST 7  /* 跳表：zset */
#define OBJ_ENCODING_EMBSTR 8    /* 嵌入式字符串：短字符串优化 */
#define OBJ_ENCODING_QUICKLIST 9 /* 快速列表：listpack 链表 */
#define OBJ_ENCODING_STREAM 10   /* 流：radix tree */
#define OBJ_ENCODING_LISTPACK 11 /* listpack：紧凑编码 */
#define OBJ_ENCODING_LISTPACK_EX 12 /* 扩展 listpack：带元数据 */
```

**编码示例**：

| 命令 | type | encoding | 说明 |
|------|------|----------|------|
| `SET key 123` | OBJ_STRING | OBJ_ENCODING_INT | 整数编码 |
| `SET key "hello"` | OBJ_STRING | OBJ_ENCODING_EMBSTR | 短字符串 |
| `SET key "very long string..."` | OBJ_STRING | OBJ_ENCODING_RAW | 长字符串 |
| `LPUSH list a b c` | OBJ_LIST | OBJ_ENCODING_QUICKLIST | 列表 |
| `SADD set 1 2 3` | OBJ_SET | OBJ_ENCODING_INTSET | 整数集合 |
| `ZADD zset 1.0 a 2.0 b` | OBJ_ZSET | OBJ_ENCODING_SKIPLIST | 跳表 |

### refcount 字段（引用计数）

```c
#define OBJ_SHARED_REFCOUNT ((1 << OBJ_REFCOUNT_BITS) - 1)  /* 共享对象：永不释放 */
#define OBJ_STATIC_REFCOUNT ((1 << OBJ_REFCOUNT_BITS) - 2)  /* 栈对象：不需要释放 */
```

**引用计数的作用**：
1. **内存管理**：refcount 减到 0 时自动释放对象
2. **对象共享**：Redis 会共享整数 0-9999，节省内存
   ```c
   SET key1 100
   SET key2 100
   // key1 和 key2 共享同一个 robj，refcount=2
   ```
3. **操作示例**：
   - `incrRefCount(obj)` - 增加引用
   - `decrRefCount(obj)` - 减少引用，可能释放

---

## 特殊对象：kvobj

```c
// 文件位置：server.h:72-92

/*
 * kvobj - 嵌入键名的特殊 robj
 *
 * 【为什么需要 kvobj？】
 * 传统的 Redis：键和值分开存储
 *   键："mykey" (sds)
 *   值：robj --> "myvalue"
 *
 * kvobj：键和值在同一块内存
 *   +-------robj-------+---expiry---+---key---+---value---+
 *   | 16 bytes header | 8 bytes    | "mykey" | "myvalue" |
 *   +------------------+------------+---------+-----------+
 *
 * 【好处】
 * 1. 减少内存分配次数（一次分配完成）
 * 2. 提高缓存局部性（键值在相邻内存）
 * 3. 方便携带过期时间
 *
 * 【使用场景】
 * - 小键小值：键+值 < 某个阈值
 * - 带过期时间的键
 */
typedef struct redisObject kvobj;

/* kvobj 内存布局示例 */

// 示例 1：带过期时间的键 "mykey"
// +--------------+--------------+--------------+--------------------+
// | serverObject | Expiry Time  | key-hdr-size | sdshdr5 "mykey" \0 |
// | 16 bytes     | 8 byte       | 1 byte       | 1      +   5   + 1 |
// +--------------+--------------+--------------+--------------------+

// 示例 2：键 "mykey" + 值 "myvalue"
// +--------------+--------------+--------------------+----------------------+
// | serverObject | key-hdr-size | sdshdr5 "mykey" \0 | sdshdr8 "myvalue" \0 |
// | 16 bytes     | 1 byte       | 1      +   5   + 1 | 3    +      7    + 1 |
// +--------------+--------------+--------------------+----------------------+
```

---

## 客户端结构：client

```c
// 文件位置：server.h:1364-1514（150行！）

/*
 * client - 代表一个连接到 Redis 的客户端
 *
 * 【重要性】
 * 这是理解 Redis 网络层和命令执行的关键！
 *
 * 【什么被抽象成 client？】
 * 1. 普通 TCP 连接（用户连接）
 * 2. 从库连接（replication client）
 * 3. AOF 文件（fake client，用于重放命令）
 * 4. Lua 脚本执行上下文（fake client）
 * 5. 模块执行上下文（fake client）
 *
 * 【生命周期】
 * 1. 客户端连接    --> acceptTcpHandler() 创建 client
 * 2. 接收数据      --> readQueryFromClient() 写入 querybuf
 * 3. 解析命令      --> processInputBuffer() 解析出 argc/argv
 * 4. 执行命令      --> processCommand() 执行并写入 reply
 * 5. 发送响应      --> sendReplyToClient() 发送数据
 * 6. 客户端断开    --> freeClient() 释放 client
 */
typedef struct client {
    /* ============ 基本信息 ============ */
    uint64_t id;                /* 客户端唯一 ID，自增 */
    uint64_t flags;             /* 客户端标志：CLIENT_SLAVE, CLIENT_BLOCKED 等 */
    connection *conn;           /* 连接对象（TCP socket 或 TLS）*/
    uint8_t tid;                /* 所属的 IO 线程 ID */
    int resp;                   /* RESP 协议版本：2 或 3 */
    redisDb *db;                /* 当前选中的数据库（SELECT 命令切换）*/
    robj *name;                 /* 客户端名称（CLIENT SETNAME 设置）*/

    /* ============ 输入缓冲区（接收命令）============ */
    sds querybuf;               /* 输入缓冲区：存储客户端发来的原始数据
                                 * 比如："*3\r\n$3\r\nSET\r\n$3\r\nkey\r\n$5\r\nvalue\r\n"
                                 */
    size_t qb_pos;              /* 已读取位置 */
    size_t querybuf_peak;       /* 最近的峰值大小（用于监控）*/

    /* ============ 命令解析结果 ============ */
    int argc;                   /* 当前命令的参数个数 */
    robj **argv;                /* 参数数组：argv[0]="SET", argv[1]="key", argv[2]="value" */
    int argv_len;               /* argv 数组大小（可能大于 argc）*/
    struct redisCommand *cmd;   /* 当前要执行的命令结构 */
    struct redisCommand *lastcmd; /* 上一个执行的命令 */

    /* ============ 输出缓冲区（发送响应）============ */
    list *reply;                /* 回复对象链表：存储要发给客户端的响应 */
    unsigned long long reply_bytes; /* 回复缓冲区总字节数 */
    size_t sentlen;             /* 当前对象已发送的字节数 */
    char *buf;                  /* 静态缓冲区：小响应直接用这个（16KB）*/
    int bufpos;                 /* buf 中已使用的字节数 */

    /* ============ 时间和状态 ============ */
    time_t ctime;               /* 客户端创建时间 */
    time_t lastinteraction;     /* 最后一次交互时间（用于超时检测）*/
    long duration;              /* 当前命令执行耗时（毫秒）*/
    int authenticated;          /* 是否已认证（需要密码时）*/

    /* ============ 复制相关（如果是从库连接）============ */
    int replstate;              /* 复制状态：SLAVE_STATE_ONLINE 等 */
    int repldbfd;               /* RDB 文件描述符 */
    off_t repldboff;            /* RDB 文件偏移量 */
    off_t repldbsize;           /* RDB 文件大小 */
    long long reploff;          /* 复制偏移量 */
    long long repl_ack_off;     /* 从库确认的偏移量 */
    long long repl_ack_time;    /* 从库最后确认时间 */
    char replid[CONFIG_RUN_ID_SIZE+1]; /* 主库的 replication ID */
    int slave_listening_port;   /* 从库监听端口 */
    int slave_capa;             /* 从库能力：SLAVE_CAPA_PSYNC2 等 */

    /* ============ 事务相关 ============ */
    multiState mstate;          /* MULTI/EXEC 事务状态：缓存的命令队列 */

    /* ============ 阻塞相关 ============ */
    blockingState bstate;       /* 阻塞状态：BLPOP, WAIT 等命令用 */

    /* ============ Pub/Sub 订阅 ============ */
    dict *pubsub_channels;      /* 订阅的频道：SUBSCRIBE channel */
    dict *pubsub_patterns;      /* 订阅的模式：PSUBSCRIBE pattern */

    /* ============ 监视键（WATCH）============ */
    list *watched_keys;         /* WATCH 命令监视的键（用于事务 CAS）*/

    /* ============ 统计信息 ============ */
    unsigned long long net_input_bytes;   /* 从这个客户端读取的总字节数 */
    unsigned long long net_output_bytes;  /* 发送给这个客户端的总字节数 */
    unsigned long long commands_processed; /* 执行的命令总数 */

    /* ... 还有几十个其他字段（跟踪、模块、内存统计等）... */
} client;
```

### client 的重要标志位

```c
// 文件位置：server.h:366-439

/* 客户端标志（flags 字段）*/
#define CLIENT_SLAVE (1<<0)        /* 这是一个从库连接 */
#define CLIENT_MASTER (1<<1)       /* 这是主库连接（当前服务器是从库）*/
#define CLIENT_MONITOR (1<<2)      /* MONITOR 命令：监视所有命令 */
#define CLIENT_MULTI (1<<3)        /* 正在执行 MULTI 事务 */
#define CLIENT_BLOCKED (1<<4)      /* 被阻塞：等待 BLPOP, WAIT 等 */
#define CLIENT_DIRTY_CAS (1<<5)    /* WATCH 的键被修改，EXEC 会失败 */
#define CLIENT_CLOSE_AFTER_REPLY (1<<6) /* 发送完响应后关闭连接 */
#define CLIENT_SCRIPT (1<<8)       /* 这是 Lua 脚本执行的伪客户端 */
#define CLIENT_READONLY (1<<17)    /* 集群只读模式 */
#define CLIENT_PUBSUB (1<<18)      /* Pub/Sub 模式：只能执行订阅命令 */
```

### 客户端的输入输出流程

```
[客户端] ---发送---> [TCP Socket]
                         |
                         v
                  readQueryFromClient()
                         |
                         v
                  +-----------------+
                  |   querybuf      |  <-- 原始数据："*3\r\n$3\r\nSET..."
                  +-----------------+
                         |
                         v
                 processInputBuffer()
                         |
                         v
                  +-----------------+
                  | argc=3          |
                  | argv[]=["SET",  |  <-- 解析后的命令
                  |        "key",   |
                  |        "value"] |
                  +-----------------+
                         |
                         v
                  processCommand()
                         |
                         v
                  执行命令 + 生成响应
                         |
                         v
                  +-----------------+
                  |     reply       |  <-- 响应数据："+OK\r\n"
                  |      buf        |
                  +-----------------+
                         |
                         v
                  sendReplyToClient()
                         |
                         v
                  [TCP Socket] ---发送---> [客户端]
```

---

## 全局服务器状态：redisServer

```c
// 文件位置：server.h:1812-2385（573行！！！）

/*
 * redisServer - 全局服务器状态（超级核心！）
 *
 * 【重要性】
 * 这是 Redis 最重要的结构！全局只有一个实例，叫做 `server`
 *
 * 【包含什么？】
 * 整个 Redis 服务器的所有状态：
 * - 配置选项（端口、密码、内存限制等）
 * - 运行时状态（客户端列表、数据库、事件循环）
 * - 统计信息（命令数、连接数、内存使用）
 * - 持久化状态（RDB、AOF）
 * - 复制状态（主从复制）
 * - 集群状态（Redis Cluster）
 *
 * 【访问方式】
 * 在代码中直接用全局变量 server：
 *   server.port          // 监听端口
 *   server.db[0]         // 数据库0
 *   server.clients       // 所有客户端
 *   server.maxmemory     // 最大内存
 *
 * 【字段太多！】
 * 这个结构体有 200+ 个字段，下面列出最重要的
 */
struct redisServer {
    /* ============ 通用字段 ============ */
    pid_t pid;                  /* 主进程 PID */
    pthread_t main_thread_id;   /* 主线程 ID */
    char *configfile;           /* 配置文件路径：redis.conf */
    int hz;                     /* serverCron() 执行频率（默认10）*/
    redisDb *db;                /* 数据库数组：默认 16 个（db[0] - db[15]）*/
    dict *commands;             /* 命令表：所有 Redis 命令（GET, SET, LPUSH...）*/
    aeEventLoop *el;            /* 事件循环：Redis 的核心，处理 I/O 和定时任务 */

    /* ============ 网络配置 ============ */
    int port;                   /* TCP 监听端口：默认 6379 */
    int tls_port;               /* TLS 端口：默认 0（不启用）*/
    char *bindaddr[CONFIG_BINDADDR_MAX]; /* 绑定地址：默认 0.0.0.0 */
    int bindaddr_count;         /* 绑定地址数量 */
    list *clients;              /* 所有客户端连接的链表 */
    list *slaves;               /* 所有从库连接的链表 */
    client *current_client;     /* 当前正在执行命令的客户端 */
    uint64_t next_client_id;    /* 下一个客户端 ID（自增）*/
    int maxclients;             /* 最大客户端连接数：默认 10000 */

    /* ============ 加载状态 ============ */
    volatile sig_atomic_t loading; /* 是否正在加载 RDB/AOF：1=加载中 */
    off_t loading_total_bytes;  /* 要加载的总字节数 */
    off_t loading_loaded_bytes; /* 已加载的字节数 */
    time_t loading_start_time;  /* 加载开始时间 */

    /* ============ 统计信息 ============ */
    time_t stat_starttime;           /* 服务器启动时间 */
    long long stat_numcommands;      /* 已处理的命令总数 */
    long long stat_numconnections;   /* 已接收的连接总数 */
    long long stat_expiredkeys;      /* 已过期的键数量 */
    long long stat_evictedkeys;      /* 已淘汰的键数量（maxmemory）*/
    long long stat_keyspace_hits;    /* 键空间命中次数（GET 找到键）*/
    long long stat_keyspace_misses;  /* 键空间未命中次数（GET 没找到）*/
    size_t stat_peak_memory;         /* 内存使用峰值 */
    long long stat_fork_time;        /* 最近一次 fork() 耗时（微秒）*/
    long long stat_rejected_conn;    /* 因 maxclients 拒绝的连接数 */

    /* ============ 配置选项 ============ */
    int verbosity;              /* 日志级别：LL_DEBUG, LL_NOTICE, LL_WARNING */
    int maxidletime;            /* 客户端超时时间（秒）：0=不超时 */
    int active_expire_enabled;  /* 是否启用主动过期：1=是 */
    size_t client_max_querybuf_len; /* 客户端输入缓冲区最大值 */
    int dbnum;                  /* 数据库数量：默认 16 */
    int daemonize;              /* 是否后台运行：1=是 */
    char *pidfile;              /* PID 文件路径 */

    /* ============ AOF 持久化 ============ */
    int aof_enabled;            /* AOF 是否启用：0=关闭, 1=开启 */
    int aof_state;              /* AOF 状态：AOF_ON, AOF_OFF, AOF_WAIT_REWRITE */
    int aof_fsync;              /* fsync 策略：
                                 *   AOF_FSYNC_NO=不调用
                                 *   AOF_FSYNC_ALWAYS=每条命令
                                 *   AOF_FSYNC_EVERYSEC=每秒一次（默认）*/
    int aof_fd;                 /* AOF 文件描述符 */
    sds aof_buf;                /* AOF 缓冲区：待写入的数据 */
    off_t aof_current_size;     /* 当前 AOF 文件大小 */
    off_t aof_rewrite_base_size; /* 上次重写后的 AOF 大小 */
    int aof_rewrite_scheduled;  /* 是否计划执行 AOF 重写 */
    pid_t aof_child_pid;        /* AOF 重写子进程 PID：-1=没有子进程 */

    /* ============ RDB 持久化 ============ */
    long long dirty;            /* 自上次保存以来修改的键数量 */
    long long dirty_before_bgsave; /* BGSAVE 前的 dirty 值 */
    struct saveparam *saveparams; /* SAVE 触发条件：比如 "900秒内改了1个键" */
    int saveparamslen;          /* saveparams 数组长度 */
    char *rdb_filename;         /* RDB 文件名：默认 "dump.rdb" */
    int rdb_compression;        /* 是否压缩 RDB：1=是 */
    int rdb_checksum;           /* 是否校验 RDB：1=是 */
    time_t lastsave;            /* 上次保存时间（UNIX 时间戳）*/
    pid_t rdb_child_pid;        /* RDB 保存子进程 PID：-1=没有子进程 */

    /* ============ 复制配置（作为主库）============ */
    char replid[CONFIG_RUN_ID_SIZE+1]; /* 主库 replication ID（40字符）*/
    char replid2[CONFIG_RUN_ID_SIZE+1]; /* 第二个 replication ID（故障转移用）*/
    long long master_repl_offset;   /* 主库的复制偏移量 */
    int repl_backlog_size;          /* 复制积压缓冲区大小：默认 1MB */
    replBacklog *repl_backlog;      /* 复制积压缓冲区：用于部分重同步 */
    list *slaves;                   /* 从库列表 */

    /* ============ 复制配置（作为从库）============ */
    char *masterhost;           /* 主库主机名：NULL=不是从库 */
    int masterport;             /* 主库端口 */
    client *master;             /* 主库客户端对象 */
    repl_state repl_state;      /* 复制状态：REPL_STATE_CONNECT, REPL_STATE_CONNECTED 等 */
    int repl_serve_stale_data;  /* 断线时是否继续服务：1=是 */
    int repl_slave_ro;          /* 从库是否只读：1=是 */
    int repl_diskless_sync;     /* 无盘复制：1=直接发送 RDB 到 socket */

    /* ============ 内存管理 ============ */
    size_t maxmemory;           /* 最大内存限制（字节）：0=不限制 */
    int maxmemory_policy;       /* 淘汰策略：
                                 *   MAXMEMORY_VOLATILE_LRU=LRU 淘汰有过期时间的键
                                 *   MAXMEMORY_ALLKEYS_LRU=LRU 淘汰所有键
                                 *   MAXMEMORY_VOLATILE_LFU=LFU 淘汰有过期时间的键
                                 *   MAXMEMORY_ALLKEYS_LFU=LFU 淘汰所有键
                                 *   MAXMEMORY_VOLATILE_RANDOM=随机淘汰有过期时间的键
                                 *   MAXMEMORY_ALLKEYS_RANDOM=随机淘汰所有键
                                 *   MAXMEMORY_VOLATILE_TTL=淘汰 TTL 最小的键
                                 *   MAXMEMORY_NO_EVICTION=不淘汰，返回错误 */
    int maxmemory_samples;      /* LRU/LFU 采样数量：默认 5 */

    /* ============ 集群配置 ============ */
    int cluster_enabled;        /* 是否启用集群：1=是 */
    struct clusterState *cluster; /* 集群状态 */

    /* ============ Lua 脚本 ============ */
    lua_State *lua;             /* Lua 虚拟机 */
    client *lua_client;         /* Lua 脚本执行的伪客户端 */
    long long lua_time_limit;   /* 脚本最大执行时间（毫秒）*/

    /* ============ 模块系统 ============ */
    dict *moduleapi;            /* 模块 API 字典 */
    list *loadmodule_queue;     /* 启动时要加载的模块列表 */

    /* ============ 慢查询日志 ============ */
    list *slowlog;              /* 慢查询日志链表 */
    long long slowlog_entry_id; /* 慢查询日志 ID（自增）*/
    long long slowlog_log_slower_than; /* 慢查询阈值（微秒）*/
    unsigned long slowlog_max_len;     /* 慢查询日志最大长度 */

    /* ============ Pub/Sub ============ */
    dict *pubsub_channels;      /* 频道订阅：key=频道名, value=订阅客户端列表 */
    list *pubsub_patterns;      /* 模式订阅：链表项=订阅模式 */

    /* ... 还有 100+ 个其他字段 ... */
};

/* 全局服务器实例（在 server.c 中定义）*/
extern struct redisServer server;
```

### 重要子结构

#### saveparam（RDB 保存条件）

```c
// 文件位置：server.h:1548-1551

struct saveparam {
    time_t seconds;  /* 时间间隔（秒）*/
    int changes;     /* 修改的键数量 */
};

/* 配置示例：redis.conf
 * save 900 1      # 900秒内至少改了1个键，就保存
 * save 300 10     # 300秒内至少改了10个键，就保存
 * save 60 10000   # 60秒内至少改了10000个键，就保存
 */
```

---

## 数据库结构：redisDb

```c
// 文件位置：server.h:1127-1141

/*
 * redisDb - Redis 数据库结构
 *
 * 【说明】
 * Redis 默认有 16 个数据库（db[0] - db[15]），通过 SELECT 命令切换
 * 每个数据库有独立的键空间、过期字典、阻塞键等
 *
 * 【核心概念】
 * - keys: 主键空间，存储所有键值对
 * - expires: 过期字典，只存储有过期时间的键
 */
typedef struct redisDb {
    kvstore *keys;              /* 主键空间：所有键值对存在这里
                                 * key=键名(sds), value=robj
                                 * 这是数据库的核心！*/

    kvstore *expires;           /* 过期字典：存储有过期时间的键
                                 * key=键名(sds), value=过期时间(mstime_t)
                                 * 只有设置了 EXPIRE 的键才会在这里 */

    estore *subexpires;         /* 子键过期（哈希字段过期）
                                 * Redis 7.4+ 支持哈希字段级别的过期 */

    dict *blocking_keys;        /* 阻塞键字典：有客户端在等这些键
                                 * 比如 BLPOP mylist，mylist 就在这里 */

    dict *blocking_keys_unblock_on_nokey;
                                /* 键删除时取消阻塞的键
                                 * 比如 XREADGROUP，键被删除时要唤醒客户端 */

    dict *stream_claim_pending_keys;
                                /* 流的待认领条目 */

    dict *ready_keys;           /* 就绪键：有数据了，可以唤醒阻塞的客户端 */

    dict *watched_keys;         /* 监视键：WATCH 命令用于事务 CAS */

    int id;                     /* 数据库 ID：0-15 */
    long long avg_ttl;          /* 平均 TTL（毫秒）：用于统计 */
    unsigned long expires_cursor; /* 过期扫描游标：主动过期用 */
} redisDb;
```

### 键空间和过期字典的关系

```
假设执行以下命令：
  SET key1 "value1"                 # 永久键
  SET key2 "value2" EX 10           # 10秒后过期
  SET key3 "value3" EX 20           # 20秒后过期

数据库状态：

keys 字典（主键空间）：
  +-----------------+
  | "key1" => robj1 |  ---> "value1"
  | "key2" => robj2 |  ---> "value2"
  | "key3" => robj3 |  ---> "value3"
  +-----------------+

expires 字典（过期字典）：
  +--------------------------+
  | "key2" => 1704067200000  |  (当前时间 + 10秒)
  | "key3" => 1704067210000  |  (当前时间 + 20秒)
  +--------------------------+

注意：
- key1 是永久键，只在 keys 中，不在 expires 中
- key2 和 key3 同时存在于 keys 和 expires 中
- 当键过期时，从 keys 和 expires 中都删除
```

---

## 命令结构：redisCommand

```c
// 文件位置：server.h:2676-2727

/*
 * redisCommand - Redis 命令定义
 *
 * 【说明】
 * 每个 Redis 命令（GET, SET, LPUSH 等）都有一个 redisCommand 结构
 * 所有命令存储在 server.commands 字典中
 *
 * 【注册流程】
 * 1. 命令定义在 commands/*.json 文件中
 * 2. 编译时生成 commands.c
 * 3. 服务器启动时注册到 server.commands
 */
struct redisCommand {
    /* ============ 声明性数据（命令的静态属性）============ */
    char *name;                 /* 命令名称：大写，比如 "GET", "SET" */
    redisCommandProc *proc;     /* 命令处理函数：比如 getCommand(), setCommand() */
    int arity;                  /* 参数数量：
                                 *   > 0: 固定数量（GET=2: GET key）
                                 *   < 0: 至少 |arity|-1 个（SET=-3: SET key value [EX ...]）*/
    uint64_t flags;             /* 命令标志：CMD_WRITE, CMD_READONLY 等 */
    uint64_t acl_categories;    /* ACL 类别：ACL_CATEGORY_READ, ACL_CATEGORY_WRITE 等 */

    /* ============ Key 规格（告诉 Redis 哪些参数是键）============ */
    int firstkey;               /* 第一个键的位置：GET 是 1（argv[1]）*/
    int lastkey;                /* 最后一个键的位置：GET 是 1，MGET 是 -1（最后一个参数）*/
    int keystep;                /* 键的步长：MGET 是 1（连续），MSET 是 2（key value key value）*/

    /* ============ 运行时数据（命令执行统计）============ */
    long long microseconds;     /* 执行此命令的总耗时（微秒）*/
    long long calls;            /* 此命令被调用的次数 */
    long long rejected_calls;   /* 被拒绝的调用次数 */
    long long failed_calls;     /* 失败的调用次数 */
    struct hdr_histogram *latency_histogram; /* 延迟直方图 */
};

/* 命令处理函数的原型 */
typedef void redisCommandProc(client *c);
```

### 命令示例

```c
/* GET 命令的定义（简化版）*/
{
    .name = "get",
    .proc = getCommand,         // 处理函数
    .arity = 2,                 // 固定2个参数：GET key
    .flags = CMD_READONLY | CMD_FAST,
    .acl_categories = ACL_CATEGORY_READ | ACL_CATEGORY_STRING,
    .firstkey = 1,              // 第一个键在 argv[1]
    .lastkey = 1,               // 最后一个键也在 argv[1]
    .keystep = 1
}

/* SET 命令的定义（简化版）*/
{
    .name = "set",
    .proc = setCommand,
    .arity = -3,                // 至少3个参数：SET key value [NX|XX] [EX seconds] ...
    .flags = CMD_WRITE | CMD_DENYOOM,
    .acl_categories = ACL_CATEGORY_WRITE | ACL_CATEGORY_STRING,
    .firstkey = 1,
    .lastkey = 1,
    .keystep = 1
}

/* MGET 命令的定义（简化版）*/
{
    .name = "mget",
    .proc = mgetCommand,
    .arity = -2,                // 至少2个参数：MGET key [key ...]
    .flags = CMD_READONLY | CMD_FAST,
    .acl_categories = ACL_CATEGORY_READ | ACL_CATEGORY_STRING,
    .firstkey = 1,
    .lastkey = -1,              // 最后一个键在最后一个参数
    .keystep = 1                // 每个参数都是键
}

/* MSET 命令的定义（简化版）*/
{
    .name = "mset",
    .proc = msetCommand,
    .arity = -3,                // 至少3个参数：MSET key value [key value ...]
    .flags = CMD_WRITE | CMD_DENYOOM,
    .acl_categories = ACL_CATEGORY_WRITE | ACL_CATEGORY_STRING,
    .firstkey = 1,
    .lastkey = -1,
    .keystep = 2                // 步长为2：key value key value
}
```

---

## 重要宏定义

### 配置常量

```c
// 文件位置：server.h:124-153

#define CONFIG_DEFAULT_HZ        10    /* serverCron() 每秒执行 10 次 */
#define CONFIG_MIN_HZ            1
#define CONFIG_MAX_HZ            500
#define MAX_CLIENTS_PER_CLOCK_TICK 200 /* 每个时钟周期最多处理 200 个客户端 */
#define CRON_DBS_PER_CALL 16           /* 每次 cron 最多扫描 16 个数据库 */
#define NET_MAX_WRITES_PER_EVENT (1024*64) /* 每个事件最多写 64KB */
#define OBJ_SHARED_INTEGERS 10000      /* 共享整数 0-9999 */
#define LOG_MAX_LEN    1024            /* 日志最大长度 */
#define CONFIG_RUN_ID_SIZE 40          /* Replication ID 长度 */
#define CONFIG_BINDADDR_MAX 16         /* 最多绑定 16 个地址 */
```

### 命令标志

```c
// 文件位置：server.h:233-263

#define CMD_WRITE (1ULL<<0)      /* 会修改数据：SET, DEL, LPUSH */
#define CMD_READONLY (1ULL<<1)   /* 只读命令：GET, LRANGE */
#define CMD_DENYOOM (1ULL<<2)    /* OOM 时拒绝：SET, LPUSH */
#define CMD_ADMIN (1ULL<<4)      /* 管理命令：CONFIG, SHUTDOWN */
#define CMD_PUBSUB (1ULL<<5)     /* Pub/Sub 命令：PUBLISH, SUBSCRIBE */
#define CMD_BLOCKING (1ULL<<8)   /* 可能阻塞：BLPOP, WAIT */
#define CMD_LOADING (1ULL<<9)    /* 加载时可执行：PING, INFO */
#define CMD_FAST (1ULL<<14)      /* O(1) 时间复杂度：GET, SET */
```

### 客户端标志

```c
// 文件位置：server.h:367-439

#define CLIENT_SLAVE (1<<0)      /* 从库连接 */
#define CLIENT_MASTER (1<<1)     /* 主库连接 */
#define CLIENT_MONITOR (1<<2)    /* MONITOR 客户端 */
#define CLIENT_MULTI (1<<3)      /* 在 MULTI 事务中 */
#define CLIENT_BLOCKED (1<<4)    /* 被阻塞 */
#define CLIENT_DIRTY_CAS (1<<5)  /* WATCH 的键被修改 */
#define CLIENT_PUBSUB (1<<18)    /* Pub/Sub 模式 */
#define CLIENT_READONLY (1<<17)  /* 集群只读模式 */
```

### 对象类型

```c
// 文件位置：server.h:815-835

#define OBJ_STRING 0    /* 字符串 */
#define OBJ_LIST 1      /* 列表 */
#define OBJ_SET 2       /* 集合 */
#define OBJ_ZSET 3      /* 有序集合 */
#define OBJ_HASH 4      /* 哈希 */
#define OBJ_MODULE 5    /* 模块类型 */
#define OBJ_STREAM 6    /* 流 */
```

### 对象编码

```c
// 文件位置：server.h:1034-1046

#define OBJ_ENCODING_RAW 0       /* sds 字符串 */
#define OBJ_ENCODING_INT 1       /* long 整数 */
#define OBJ_ENCODING_HT 2        /* dict 哈希表 */
#define OBJ_ENCODING_INTSET 6    /* intset 整数集合 */
#define OBJ_ENCODING_SKIPLIST 7  /* zset 跳表 */
#define OBJ_ENCODING_EMBSTR 8    /* 嵌入式字符串 */
#define OBJ_ENCODING_QUICKLIST 9 /* quicklist 快速列表 */
#define OBJ_ENCODING_STREAM 10   /* radix tree 流 */
#define OBJ_ENCODING_LISTPACK 11 /* listpack 紧凑编码 */
```

### 错误码

```c
// 文件位置：server.h:119-121

#define C_OK     0    /* 成功 */
#define C_ERR   -1    /* 通用错误 */
#define C_RETRY -2    /* 需要重试 */
```

---

## 如何阅读完整源码

### 推荐阅读顺序

1. **基础类型和常量**（1-800行）
   - 时间类型：mstime_t, ustime_t
   - 配置常量：CONFIG_DEFAULT_HZ 等
   - 命令标志：CMD_WRITE, CMD_READONLY 等

2. **核心对象：redisObject**（1057-1068行）
   - 理解 type, encoding, lru, refcount, ptr 字段
   - 查看对象类型宏（OBJ_STRING 等）
   - 查看对象编码宏（OBJ_ENCODING_RAW 等）

3. **数据库结构：redisDb**（1127-1141行）
   - keys 和 expires 的关系
   - 阻塞键和监视键

4. **客户端结构：client**（1364-1514行）
   - 输入缓冲区：querybuf
   - 命令解析：argc, argv
   - 输出缓冲区：buf, reply
   - 复制字段：replstate, reploff
   - 事务字段：mstate
   - 阻塞字段：bstate

5. **全局服务器：redisServer**（1812-2385行）
   - 通用字段：pid, configfile, hz, db, commands
   - 网络字段：port, clients, slaves
   - 统计字段：stat_numcommands, stat_expiredkeys
   - AOF 字段：aof_state, aof_buf
   - RDB 字段：rdb_child_pid, dirty
   - 复制字段：master, repl_state

6. **命令结构：redisCommand**（2676-2727行）
   - 命令属性：name, proc, arity, flags
   - 键规格：firstkey, lastkey, keystep

7. **函数声明**（2833-4365行）
   - 浏览一遍，了解 Redis 提供的 API
   - 重点看网络 I/O、客户端管理、对象操作

### 配合其他文件阅读

```
server.h          - 结构定义和宏
  ├─ server.c       - 服务器主逻辑（main 函数、初始化、事件循环）
  ├─ networking.c   - 网络 I/O（acceptTcpHandler, readQueryFromClient）
  ├─ db.c           - 数据库操作（lookupKey, dbAdd, dbDelete）
  ├─ object.c       - 对象操作（createObject, decrRefCount）
  ├─ t_string.c     - 字符串命令（setCommand, getCommand）
  ├─ t_list.c       - 列表命令（lpushCommand, lpopCommand）
  ├─ t_hash.c       - 哈希命令（hsetCommand, hgetCommand）
  ├─ t_set.c        - 集合命令（saddCommand, smembersCommand）
  ├─ t_zset.c       - 有序集合命令（zaddCommand, zrangeCommand）
  ├─ aof.c          - AOF 持久化
  ├─ rdb.c          - RDB 持久化
  ├─ replication.c  - 主从复制
  ├─ cluster.c      - Redis Cluster
  └─ sentinel.c     - Redis Sentinel
```

### 使用 VSCode 阅读技巧

1. **安装 C/C++ 扩展**
2. **使用"转到定义"功能**（F12）
   - 点击 `client` -> 跳转到定义
   - 点击 `redisServer` -> 跳转到定义
3. **使用"查找所有引用"**（Shift+F12）
   - 查看 `createClient` 在哪里被调用
4. **使用大纲视图**（Ctrl+Shift+O）
   - 快速跳转到结构体定义
5. **使用全局搜索**（Ctrl+Shift+F）
   - 搜索 `CLIENT_MULTI` 找到所有使用的地方

---

## 总结

### 核心数据结构

| 结构体 | 作用 | 重要程度 |
|--------|------|----------|
| `redisObject` | Redis 值的通用包装器 | ⭐⭐⭐⭐⭐ |
| `client` | 客户端连接状态 | ⭐⭐⭐⭐⭐ |
| `redisServer` | 全局服务器状态 | ⭐⭐⭐⭐⭐ |
| `redisDb` | 数据库结构 | ⭐⭐⭐⭐ |
| `redisCommand` | 命令定义 | ⭐⭐⭐⭐ |
| `zskiplist` | 跳表（有序集合）| ⭐⭐⭐ |
| `dict` | 哈希表 | ⭐⭐⭐ |
| `sds` | 动态字符串 | ⭐⭐⭐ |

### Redis 的设计精髓

1. **单线程事件驱动**
   - 主线程负责所有命令执行
   - I/O 线程只负责读写网络（Redis 6.0+）
   - 避免锁竞争，简化实现

2. **统一的对象系统**
   - 所有值都是 `redisObject`
   - 引用计数自动管理内存
   - 共享对象节省内存

3. **多种编码优化**
   - 小对象用紧凑编码（listpack）
   - 大对象用复杂结构（dict, skiplist）
   - 自动转换，用户无感知

4. **渐进式操作**
   - rehash 分多次完成（避免阻塞）
   - 过期键分批删除
   - 大键异步释放

5. **事件驱动架构**
   - 文件事件：网络 I/O
   - 时间事件：serverCron 定时任务
   - 事件循环：aeEventLoop

---

## 完整源码位置

- **完整 server.h**：`src/server.h`（4366行）
- **在线查看**：https://github.com/redis/redis/blob/unstable/src/server.h

---

## 下一步学习建议

1. **阅读 server.c**
   - 重点：`main()` 函数、`initServer()`、`aeMain()`

2. **阅读 networking.c**
   - 重点：`readQueryFromClient()`、`processInputBuffer()`、`sendReplyToClient()`

3. **阅读 db.c**
   - 重点：`lookupKey()`、`dbAdd()`、`dbDelete()`、`setExpire()`

4. **阅读一个命令实现**
   - 推荐从 `t_string.c` 的 `setCommand()` 开始

5. **调试 Redis**
   - 编译调试版本：`make noopt`
   - 用 GDB/LLDB 单步执行

---

**祝你学习愉快！有问题随时问我 😊**
