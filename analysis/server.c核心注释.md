# Redis server.c 核心函数详解（中文注释版）

> 📘 **说明**：server.c 是 Redis 服务器的主控文件（7801行），包含了从启动、初始化到事件循环的所有核心逻辑。本文档重点讲解最关键的函数和流程。

---

## 📚 目录

1. [文件概述](#文件概述)
2. [main() 主函数](#main-主函数)
3. [初始化流程](#初始化流程)
4. [事件循环](#事件循环)
5. [定时任务 serverCron](#定时任务-servercron)
6. [命令处理流程](#命令处理流程)
7. [关机流程](#关机流程)
8. [完整启动流程图](#完整启动流程图)

---

## 文件概述

`server.c` 是 Redis 的"大脑"，包含：

- **main() 函数**：程序入口
- **初始化系统**：`initServerConfig()`, `initServer()`, `initListeners()`
- **事件循环钩子**：`beforeSleep()`, `afterSleep()`
- **定时任务**：`serverCron()`（每秒 10 次）
- **命令处理**：`processCommand()`, `call()`
- **关机系统**：`prepareForShutdown()`
- **INFO 命令**：服务器信息统计

**文件位置**：`src/server.c`（7801行）

**核心流程**：
```
main()
  ├─> initServerConfig()      # 初始化配置
  ├─> initServer()            # 初始化核心组件
  ├─> initListeners()         # 初始化网络监听
  ├─> loadDataFromDisk()      # 加载 RDB/AOF
  └─> aeMain()                # 进入事件循环
       ├─> beforeSleep()      # 每次睡眠前
       ├─> [处理网络事件]
       └─> afterSleep()       # 每次唤醒后
```

---

## main() 主函数

```c
// 文件位置：server.c:7455-7799

/*
 * main() - Redis 服务器的程序入口
 *
 * 【功能】
 * 负责整个 Redis 服务器的启动流程：
 * 1. 初始化基础设施（内存分配器、随机数、时钟）
 * 2. 加载配置（配置文件 + 命令行参数）
 * 3. 初始化服务器核心组件
 * 4. 加载持久化数据（RDB 或 AOF）
 * 5. 启动网络监听
 * 6. 进入事件循环（永不返回）
 *
 * 【执行流程】
 * main()
 *   ├─ 第1步：基础初始化（行 7503-7536）
 *   ├─ 第2步：配置加载（行 7560-7678）
 *   ├─ 第3步：系统检查（行 7681-7705）
 *   ├─ 第4步：守护进程化（行 7707-7710）
 *   ├─ 第5步：核心初始化（行 7727-7746）
 *   ├─ 第6步：加载数据（行 7748-7762）
 *   ├─ 第7步：启动监听（行 7764-7786）
 *   └─ 第8步：事件循环（行 7796）
 */
int main(int argc, char **argv) {
    struct timeval tv;
    int j;

    /* ============ 第1步：基础初始化 ============ */

    // 7503-7510: 测试模式处理
    // 如果命令行有 --test-memory，只运行内存测试后退出

    // 7512-7515: 初始化随机数种子
    gettimeofday(&tv, NULL);
    char hashseed[16];
    getRandomHexChars(hashseed, sizeof(hashseed));
    dictSetHashFunctionSeed((uint8_t*)hashseed);

    // 7517: 记录启动时间
    server.sentinel_mode = checkForSentinelMode(argc, argv);

    // 7519-7525: 初始化配置
    initServerConfig();  // 设置默认配置值

    // 7527-7536: 初始化 ACL、模块、共享对象
    ACLInit();
    moduleInitModulesSystem();
    connTypeInitialize();
    tlsInit();

    /* ============ 第2步：配置加载 ============ */

    // 7560-7630: 解析命令行参数
    // 处理 --help, --version, --test-memory 等选项

    // 7632-7678: 加载配置文件
    if (server.configfile) {
        if (!loadServerConfig(server.configfile, config_from_stdin, options)) {
            serverLog(LL_WARNING, "Fatal error, can't open config file");
            exit(1);
        }
    }

    /* ============ 第3步：系统检查 ============ */

    // 7681-7691: Linux 内存警告
    linuxMemoryWarnings();

    // 7693-7705: 内核 bug 检测
    checkForBuggyKernels();

    /* ============ 第4步：守护进程化 ============ */

    // 7707-7710: 如果配置了 daemonize，后台运行
    if (server.daemonize) daemonize();

    /* ============ 第5步：核心初始化 ============ */

    // 7727-7730: 初始化服务器核心组件
    initServer();  // ⭐ 最重要！创建事件循环、数据库、命令表等

    // 7732-7738: 初始化集群（如果启用）
    if (server.cluster_enabled) {
        clusterInit();
    }

    // 7740-7742: 加载模块
    moduleLoadFromQueue();

    // 7744-7746: 初始化网络监听器
    initListeners();  // 绑定端口，准备接受连接

    // 7747: 最后的初始化步骤
    InitServerLast();  // 启动 IO 线程

    /* ============ 第6步：加载数据 ============ */

    // 7748-7762: 从磁盘加载数据
    if (!server.sentinel_mode) {
        loadDataFromDisk();  // ⭐ 加载 RDB 或 AOF 文件

        // 7756-7760: 打开 AOF 文件（如果启用）
        if (server.aof_state == AOF_ON) {
            if (openAofForAppend() == C_ERR) {
                serverLog(LL_WARNING, "Can't open the append-only file");
                exit(1);
            }
        }
    }

    /* ============ 第7步：启动监听 ============ */

    // 7764-7786: 准备接受客户端连接
    for (j = 0; j < CONN_TYPE_MAX; j++) {
        connListener *listener = &server.listeners[j];
        if (listener->ct == NULL) continue;

        // 为每个监听器创建文件事件处理器
        for (int i = 0; i < listener->count; i++) {
            if (aeCreateFileEvent(server.el, listener->fd[i], AE_READABLE,
                                  listener->accept_handler, listener) == AE_ERR) {
                serverPanic("Unrecoverable error creating file event.");
            }
        }
    }

    // 7788-7794: 打印启动信息
    serverLog(LL_WARNING, "Server initialized");
    serverLog(LL_WARNING, "Ready to accept connections");

    /* ============ 第8步：进入事件循环 ============ */

    // 7796: 永不返回！
    aeMain(server.el);  // ⭐ 事件循环，处理网络 I/O 和定时任务

    // 7798: 理论上不会到这里（除非事件循环退出）
    aeDeleteEventLoop(server.el);
    return 0;
}
```

### main() 流程图

```
┌─────────────────────────────────────┐
│       Redis 启动流程                │
└─────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────┐
│  1. 基础初始化                      │
│  - 随机数种子                       │
│  - initServerConfig()               │
│  - ACL, 模块, TLS 初始化            │
└─────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────┐
│  2. 配置加载                        │
│  - 解析命令行参数                   │
│  - 加载 redis.conf                  │
└─────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────┐
│  3. 系统检查                        │
│  - 内存警告                         │
│  - 内核 bug 检测                    │
└─────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────┐
│  4. 守护进程化（可选）              │
│  - daemonize()                      │
└─────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────┐
│  5. 核心初始化 ⭐                   │
│  - initServer()                     │
│  - clusterInit()（可选）            │
│  - moduleLoadFromQueue()            │
│  - initListeners()                  │
│  - InitServerLast()                 │
└─────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────┐
│  6. 加载数据 ⭐                     │
│  - loadDataFromDisk()               │
│    ├─ 加载 RDB 文件                 │
│    └─ 加载 AOF 文件                 │
└─────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────┐
│  7. 启动监听                        │
│  - 为每个监听端口创建文件事件       │
│  - 准备接受连接                     │
└─────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────┐
│  8. 进入事件循环 ⭐                 │
│  - aeMain(server.el)                │
│  - 永不返回                         │
└─────────────────────────────────────┘
```

---

## 初始化流程

### initServerConfig() - 初始化配置

```c
// 文件位置：server.c:2223-2356

/*
 * initServerConfig() - 初始化服务器配置
 *
 * 【调用时机】
 * main() 函数早期，在加载配置文件之前
 *
 * 【功能】
 * 设置所有配置项的默认值，这些值可能被配置文件覆盖
 *
 * 【主要工作】
 * 1. 生成 Run ID（40 字符随机字符串）
 * 2. 设置默认配置值：
 *    - 端口：6379
 *    - HZ：10（定时任务频率）
 *    - 数据库数量：16
 *    - 最大客户端：10000
 *    - AOF/RDB 配置
 *    - 复制配置
 * 3. 填充命令表：populateCommandTable()
 * 4. 初始化慢日志、Lua 脚本配置等
 */
void initServerConfig(void) {
    int j;

    // 2230: 生成 Run ID（每次启动都不同）
    getRandomHexChars(server.runid, CONFIG_RUN_ID_SIZE);

    // 2232-2240: 基本配置
    server.hz = CONFIG_DEFAULT_HZ;        // 定时任务频率：10次/秒
    server.arch_bits = (sizeof(long) == 8) ? 64 : 32;
    server.port = CONFIG_DEFAULT_SERVER_PORT;  // 6379
    server.tcp_backlog = CONFIG_DEFAULT_TCP_BACKLOG;  // 511
    server.bindaddr_count = 0;
    server.unixsocket = NULL;

    // 2242-2250: 数据库和客户端配置
    server.dbnum = CONFIG_DEFAULT_DBNUM;  // 16 个数据库
    server.maxclients = CONFIG_DEFAULT_MAX_CLIENTS;  // 10000
    server.maxidletime = CONFIG_DEFAULT_CLIENT_TIMEOUT;  // 0（不超时）

    // 2252-2265: 内存配置
    server.maxmemory = 0;  // 0 = 不限制
    server.maxmemory_policy = CONFIG_DEFAULT_MAXMEMORY_POLICY;  // noeviction
    server.maxmemory_samples = CONFIG_DEFAULT_MAXMEMORY_SAMPLES;  // 5

    // 2267-2285: RDB 配置
    server.saveparams = NULL;
    server.loading = 0;
    server.rdb_filename = zstrdup(CONFIG_DEFAULT_RDB_FILENAME);  // "dump.rdb"
    server.rdb_compression = CONFIG_DEFAULT_RDB_COMPRESSION;  // yes
    server.rdb_checksum = CONFIG_DEFAULT_RDB_CHECKSUM;  // yes

    // 2287-2305: AOF 配置
    server.aof_state = CONFIG_DEFAULT_AOF_ENABLED;  // AOF_OFF
    server.aof_filename = zstrdup(CONFIG_DEFAULT_AOF_FILENAME);  // "appendonly.aof"
    server.aof_fsync = CONFIG_DEFAULT_AOF_FSYNC;  // AOF_FSYNC_EVERYSEC
    server.aof_rewrite_base_size = 0;
    server.aof_current_size = 0;

    // 2307-2320: 复制配置
    server.masterhost = NULL;  // NULL = 不是从库
    server.masterport = 6379;
    server.master = NULL;
    server.repl_state = REPL_STATE_NONE;
    server.repl_serve_stale_data = CONFIG_DEFAULT_SLAVE_SERVE_STALE_DATA;
    server.repl_slave_ro = CONFIG_DEFAULT_SLAVE_READ_ONLY;

    // 2322-2330: 慢日志配置
    server.slowlog_log_slower_than = CONFIG_DEFAULT_SLOWLOG_LOG_SLOWER_THAN;  // 10000微秒
    server.slowlog_max_len = CONFIG_DEFAULT_SLOWLOG_MAX_LEN;  // 128条

    // 2340-2345: 填充命令表 ⭐
    populateCommandTable();  // 注册所有 Redis 命令

    // ... 其他配置项 ...
}
```

### initServer() - 初始化服务器

```c
// 文件位置：server.c:2794-3022

/*
 * initServer() - 初始化 Redis 服务器核心组件
 *
 * 【调用时机】
 * main() 函数中后期，配置加载完成后
 *
 * 【功能】
 * 创建和初始化服务器运行所需的所有核心数据结构：
 * - 事件循环
 * - 数据库数组
 * - 客户端列表
 * - 共享对象
 * - 复制积压缓冲区
 * - 慢日志
 * 等等...
 *
 * 【主要工作】（按顺序）
 */
void initServer(void) {
    int j;

    /* ============ 1. 信号处理 ============ */

    // 2800-2802: 安装信号处理器
    signal(SIGHUP, SIG_IGN);   // 忽略 SIGHUP
    signal(SIGPIPE, SIG_IGN);  // 忽略 SIGPIPE
    setupSignalHandlers();     // SIGTERM, SIGINT 等

    /* ============ 2. 时钟和统计初始化 ============ */

    // 2804-2810: 初始化时间缓存
    server.hz = server.config_hz;
    server.pid = getpid();
    server.main_thread_id = pthread_self();
    server.current_client = NULL;
    server.executing_client = NULL;

    // 2812-2820: 统计信息初始化
    server.stat_starttime = time(NULL);
    server.stat_numcommands = 0;
    server.stat_numconnections = 0;
    server.stat_expiredkeys = 0;
    server.stat_evictedkeys = 0;

    /* ============ 3. 创建共享对象 ============ */

    // 2822: 创建常用的共享对象（如 "+OK\r\n", 整数 0-9999）
    createSharedObjects();

    /* ============ 4. 初始化数据库 ============ */

    // 2830-2838: 创建数据库数组
    server.db = zmalloc(sizeof(redisDb) * server.dbnum);

    for (j = 0; j < server.dbnum; j++) {
        server.db[j].keys = kvstoreCreate(...);         // 主键空间
        server.db[j].expires = kvstoreCreate(...);      // 过期字典
        server.db[j].blocking_keys = dictCreate(...);   // 阻塞键
        server.db[j].watched_keys = dictCreate(...);    // 监视键
        server.db[j].id = j;
    }

    /* ============ 5. 初始化客户端相关 ============ */

    // 2845-2855: 客户端列表
    server.clients = listCreate();              // 所有客户端
    server.clients_to_close = listCreate();     // 待关闭客户端
    server.slaves = listCreate();               // 从库列表
    server.monitors = listCreate();             // MONITOR 客户端
    server.clients_pending_write = listCreate(); // 待写入客户端
    server.clients_pending_read = listCreate();  // 待读取客户端

    // 2857: 客户端超时表
    server.clients_timeout_table = raxNew();

    // 2859: 客户端索引
    server.clients_index = raxNew();

    /* ============ 6. 创建事件循环 ============ */

    // 2865-2870: 创建事件循环 ⭐
    server.el = aeCreateEventLoop(server.maxclients + CONFIG_FDSET_INCR);
    if (server.el == NULL) {
        serverLog(LL_WARNING, "Failed creating the event loop. Error message: '%s'", strerror(errno));
        exit(1);
    }

    // 2872-2875: 注册睡眠前/唤醒后钩子
    aeSetBeforeSleepProc(server.el, beforeSleep);  // 睡眠前处理
    aeSetAfterSleepProc(server.el, afterSleep);    // 唤醒后处理

    /* ============ 7. 创建定时事件 ============ */

    // 2880-2885: 创建 serverCron 定时任务
    if (aeCreateTimeEvent(server.el, 1, serverCron, NULL, NULL) == AE_ERR) {
        serverPanic("Can't create event loop timers.");
        exit(1);
    }

    /* ============ 8. 初始化 Pub/Sub ============ */

    // 2890-2895: Pub/Sub 字典
    server.pubsub_channels = dictCreate(&keylistDictType);
    server.pubsub_patterns = listCreate();
    server.pubsubshard_channels = dictCreate(&keylistDictType);

    /* ============ 9. 初始化复制相关 ============ */

    // 2900-2910: 复制配置
    server.master = NULL;
    server.cached_master = NULL;
    server.repl_backlog = NULL;
    server.repl_backlog_size = CONFIG_DEFAULT_REPL_BACKLOG_SIZE;

    /* ============ 10. 初始化 Lua 脚本 ============ */

    // 2915: 初始化 Lua 环境
    scriptingInit(1);

    /* ============ 11. 初始化慢日志 ============ */

    // 2920: 创建慢日志列表
    server.slowlog = listCreate();
    server.slowlog_entry_id = 0;

    /* ============ 12. 初始化 LRU 时钟 ============ */

    // 2925: 更新 LRU 时钟
    updateCachedTime(1);
    server.lruclock = getLRUClock();

    /* ============ 13. 初始化集群（如果启用）============ */

    // 2930-2935: 集群模式
    if (server.cluster_enabled) {
        // 集群初始化会在 main() 中单独调用 clusterInit()
    }

    /* ============ 14. 初始化模块系统 ============ */

    // 2940-2945: 模块字典
    server.moduleapi = dictCreate(&moduleAPIDictType);
    server.sharedapi = dictCreate(&moduleAPIDictType);

    /* ============ 15. 其他初始化 ============ */

    // 2950-3020:
    // - 初始化 ACL
    // - 初始化 TLS
    // - 初始化 IO 线程相关结构
    // - 初始化 latency monitor
    // - 初始化 replication buffer
    // - 打开 PID 文件
    // 等等...

    serverLog(LL_NOTICE, "Server initialized");
}
```

### initListeners() - 初始化网络监听器

```c
// 文件位置：server.c:3024-3094

/*
 * initListeners() - 初始化网络监听器
 *
 * 【调用时机】
 * main() 函数后期，initServer() 之后
 *
 * 【功能】
 * 绑定配置的端口，准备接受客户端连接
 *
 * 【支持的连接类型】
 * - TCP 连接：普通 TCP socket
 * - TLS 连接：加密的 TLS socket
 * - Unix socket：本地 Unix 域套接字
 */
void initListeners(void) {
    int j;

    /* ============ 1. 绑定 TCP 端口 ============ */

    // 3030-3040: 普通 TCP 端口（默认 6379）
    if (server.port != 0 &&
        listenToPort(server.port, &server.listeners[CONN_TYPE_SOCKET]) == C_ERR) {
        serverLog(LL_WARNING, "Failed listening on port %u (TCP), aborting.", server.port);
        exit(1);
    }

    /* ============ 2. 绑定 TLS 端口 ============ */

    // 3042-3052: TLS 加密端口
    if (server.tls_port != 0 &&
        listenToPort(server.tls_port, &server.listeners[CONN_TYPE_TLS]) == C_ERR) {
        serverLog(LL_WARNING, "Failed listening on port %u (TLS), aborting.", server.tls_port);
        exit(1);
    }

    /* ============ 3. 绑定 Unix Socket ============ */

    // 3054-3070: Unix 域套接字
    if (server.unixsocket != NULL) {
        unlink(server.unixsocket);  // 删除旧的 socket 文件
        if (server.listeners[CONN_TYPE_UNIX].count == 0 &&
            listenToPort(-1, &server.listeners[CONN_TYPE_UNIX]) == C_ERR) {
            serverLog(LL_WARNING, "Opening Unix socket: %s", server.neterr);
            exit(1);
        }
    }

    /* ============ 4. 验证至少有一个监听器 ============ */

    // 3072-3080: 检查是否成功绑定了至少一个端口
    int listen_count = 0;
    for (j = 0; j < CONN_TYPE_MAX; j++) {
        listen_count += server.listeners[j].count;
    }

    if (listen_count == 0) {
        serverLog(LL_WARNING, "Configured to not listen anywhere, exiting.");
        exit(1);
    }

    /* ============ 5. 打印监听信息 ============ */

    // 3082-3094: 日志输出监听的地址和端口
    for (j = 0; j < CONN_TYPE_MAX; j++) {
        connListener *listener = &server.listeners[j];
        if (listener->count == 0) continue;

        for (int i = 0; i < listener->count; i++) {
            serverLog(LL_NOTICE, "Ready to accept connections %s on %s:%d",
                     listener->ct->get_type(NULL),
                     listener->bindaddr[i] ? listener->bindaddr[i] : "*",
                     listener->port);
        }
    }
}
```

### loadDataFromDisk() - 加载数据

```c
// 文件位置：server.c:7173-7288

/*
 * loadDataFromDisk() - 从磁盘加载持久化数据
 *
 * 【调用时机】
 * main() 函数后期，网络监听启动前
 *
 * 【功能】
 * 按优先级加载持久化数据：
 * 1. 优先加载 AOF 文件（如果启用且存在）
 * 2. 其次加载 RDB 文件（如果 AOF 不存在）
 *
 * 【为什么 AOF 优先？】
 * AOF 记录了每个写操作，比 RDB 快照更完整，数据丢失更少
 *
 * 【加载过程】
 * 1. 设置 loading 标志（阻止客户端连接）
 * 2. 加载数据文件
 * 3. 清除 loading 标志
 * 4. 打印加载耗时
 */
void loadDataFromDisk(void) {
    long long start = ustime();

    /* ============ 1. 尝试加载 AOF ============ */

    // 7180-7220: 如果 AOF 启用，优先加载 AOF
    if (server.aof_state == AOF_ON) {
        serverLog(LL_NOTICE, "Loading AOF...");

        // 7185: 加载 AOF 文件
        int ret = loadAppendOnlyFiles(server.aof_manifest);

        if (ret == AOF_OK) {
            // 7190-7195: AOF 加载成功
            serverLog(LL_NOTICE, "DB loaded from append only file: %.3f seconds",
                     (float)(ustime() - start) / 1000000);
        } else if (ret == AOF_TRUNCATED) {
            // 7197-7202: AOF 被截断（比如磁盘满了）
            serverLog(LL_WARNING, "AOF file is not complete!");
        } else if (ret == AOF_NOT_EXIST) {
            // 7204-7210: AOF 文件不存在，尝试加载 RDB
            serverLog(LL_NOTICE, "AOF file not found, loading RDB file.");
            if (rdbLoad(server.rdb_filename, NULL, RDBFLAGS_NONE) == RDB_OK) {
                serverLog(LL_NOTICE, "DB loaded from disk: %.3f seconds",
                         (float)(ustime() - start) / 1000000);
            }
        } else {
            // 7212-7215: AOF 加载失败
            serverLog(LL_WARNING, "Fatal error loading the AOF file.");
            exit(1);
        }
    }

    /* ============ 2. 加载 RDB（如果没有 AOF）============ */

    // 7222-7240: 如果 AOF 未启用，加载 RDB
    else {
        serverLog(LL_NOTICE, "Loading RDB...");

        rdbSaveInfo rsi = RDB_SAVE_INFO_INIT;
        int ret = rdbLoad(server.rdb_filename, &rsi, RDBFLAGS_NONE);

        if (ret == RDB_OK) {
            // 7230-7235: RDB 加载成功
            serverLog(LL_NOTICE, "DB loaded from disk: %.3f seconds",
                     (float)(ustime() - start) / 1000000);

            // 7237-7240: 如果是从库，设置复制 ID 和偏移量
            if (server.masterhost && rsi.repl_id_is_set) {
                memcpy(server.replid2, rsi.repl_id, sizeof(server.replid2));
                server.second_replid_offset = rsi.repl_offset;
            }
        } else if (ret == RDB_NOT_EXIST) {
            // 7242-7245: RDB 不存在（全新的 Redis）
            serverLog(LL_NOTICE, "No RDB file found, starting empty.");
        } else {
            // 7247-7250: RDB 加载失败
            serverLog(LL_WARNING, "Fatal error loading the DB: %s. Exiting.", strerror(errno));
            exit(1);
        }
    }

    /* ============ 3. 加载完成后的处理 ============ */

    // 7252-7260: 打印数据库信息
    for (int j = 0; j < server.dbnum; j++) {
        long long keys = kvstoreSize(server.db[j].keys);
        long long expires = kvstoreSize(server.db[j].expires);
        if (keys || expires) {
            serverLog(LL_NOTICE, "DB %d: %lld keys (%lld volatile) in %lld slots HT.",
                     j, keys, expires, kvstoreBuckets(server.db[j].keys));
        }
    }

    // 7262-7270: 如果启用了集群，验证配置
    if (server.cluster_enabled) {
        if (verifyClusterConfigWithData() == C_ERR) {
            serverLog(LL_WARNING,
                     "You can't have keys in a DB different than DB 0 when in "
                     "Cluster mode. Exiting.");
            exit(1);
        }
    }

    // 7272-7280: 如果是 Sentinel 模式，加载 Sentinel 配置
    if (server.sentinel_mode) {
        sentinelLoadConfigFromQueue();
    }
}
```

---

## 事件循环

Redis 使用 **事件驱动模型**，事件循环是服务器的心脏。

### 事件循环流程

```
┌──────────────────────────────────┐
│     aeMain(server.el)            │  <-- 永不返回
│     (事件循环主函数)              │
└──────────────────────────────────┘
             │
             ▼
    ┌────────────────────┐
    │  beforeSleep()     │  <-- 睡眠前处理
    └────────────────────┘
             │
             ▼
    ┌────────────────────┐
    │  aeProcessEvents() │  <-- 等待并处理事件
    │  - 文件事件         │      (网络 I/O)
    │  - 时间事件         │      (定时任务)
    └────────────────────┘
             │
             ▼
    ┌────────────────────┐
    │  afterSleep()      │  <-- 唤醒后处理
    └────────────────────┘
             │
             └──────> 循环回到开始
```

### beforeSleep() - 睡眠前处理

```c
// 文件位置：server.c:1799-1986

/*
 * beforeSleep() - 事件循环每次睡眠前调用
 *
 * 【调用时机】
 * 每次事件循环进入 epoll_wait/select 前
 *
 * 【为什么需要这个函数？】
 * 在处理完所有事件后、睡眠等待新事件前，有些任务需要在这个时机完成：
 * - 将 AOF 缓冲区写入磁盘
 * - 将待发送的数据发给客户端
 * - 清理过期的键
 * - 处理阻塞的客户端
 *
 * 【主要工作】（按执行顺序）
 */
void beforeSleep(struct aeEventLoop *eventLoop) {
    UNUSED(eventLoop);

    /* ============ 1. 模块钩子 ============ */

    // 1805: 调用模块的 beforeSleep 钩子
    moduleFireServerEvent(REDISMODULE_EVENT_EVENTLOOP, REDISMODULE_SUBEVENT_EVENTLOOP_BEFORE_SLEEP, NULL);

    /* ============ 2. 处理 TLS 待处理数据 ============ */

    // 1808-1810: TLS 连接可能有缓冲的数据需要处理
    tlsProcessPendingData();

    /* ============ 3. 集群睡眠前处理 ============ */

    // 1813-1815: 集群模式的睡眠前任务
    if (server.cluster_enabled) clusterBeforeSleep();

    /* ============ 4. 处理阻塞客户端 ============ */

    // 1818-1825: 检查是否有键就绪，可以唤醒阻塞的客户端
    // 比如 BLPOP 等待的列表有数据了
    if (listLength(server.ready_keys) > 0) {
        handleClientsBlockedOnKeys();
    }

    /* ============ 5. 快速过期循环 ============ */

    // 1828-1835: 快速清理一些过期键（不能阻塞太久）
    if (server.active_expire_enabled && !server.masterhost) {
        activeExpireCycle(ACTIVE_EXPIRE_CYCLE_FAST);
    }

    /* ============ 6. 发送 ACK 到从库 ============ */

    // 1838-1845: 如果是主库，向从库发送复制确认
    if (server.repl_backlog && listLength(server.slaves) > 0) {
        replicationSendAckToReplicas();
    }

    /* ============ 7. 客户端侧缓存失效 ============ */

    // 1848-1855: 发送缓存失效消息给跟踪的客户端
    trackingProcessPendingKeyInvalidations();

    /* ============ 8. 写入 AOF 缓冲区 ============ */

    // 1858-1880: 将 AOF 缓冲区的数据写入磁盘 ⭐
    // 这是持久化的关键！
    if (server.aof_state == AOF_ON) {
        // 1862: 将缓冲区写入文件
        flushAppendOnlyFile(0);  // 0 = 不强制 fsync
    }

    /* ============ 9. 处理待写入客户端 ============ */

    // 1885-1895: 将数据发送给有待写入数据的客户端
    // 这个函数会遍历 server.clients_pending_write 列表
    handleClientsWithPendingWrites();

    /* ============ 10. 发送客户端到 IO 线程 ============ */

    // 1898-1910: 如果启用了 IO 线程，将客户端分配给 IO 线程处理
    if (server.io_threads_active) {
        IOThreadBeforeEventLoopRead();
    }

    /* ============ 11. 释放客户端 ============ */

    // 1913-1920: 异步释放待关闭的客户端
    // 避免在主逻辑中释放，可能阻塞
    freeClientsInAsyncFreeQueue();

    /* ============ 12. 裁剪复制积压缓冲区 ============ */

    // 1923-1930: 如果复制缓冲区太大，裁剪掉旧数据
    if (server.repl_backlog) {
        incrementalTrimReplicationBacklog(REPL_BACKLOG_TRIM_BLOCKS_PER_CALL);
    }

    /* ============ 13. 驱逐客户端 ============ */

    // 1933-1940: 如果内存不足，驱逐一些客户端
    evictClients();

    /* ============ 14. 更新缓存时间 ============ */

    // 1945: 更新时间缓存（避免频繁调用 gettimeofday）
    updateCachedTime(0);
}
```

### afterSleep() - 唤醒后处理

```c
// 文件位置：server.c:1991-2029

/*
 * afterSleep() - 事件循环每次唤醒后调用
 *
 * 【调用时机】
 * 每次事件循环从 epoll_wait/select 返回后
 *
 * 【主要工作】
 * 相比 beforeSleep，afterSleep 做的事情少得多：
 * - 更新时间缓存
 * - 获取模块 GIL 锁
 */
void afterSleep(struct aeEventLoop *eventLoop) {
    UNUSED(eventLoop);

    /* ============ 1. 更新缓存时间 ============ */

    // 1995-2000: 更新时间缓存
    // 因为从睡眠中醒来，时间可能过去了一段时间
    updateCachedTime(1);  // 1 = 强制更新

    /* ============ 2. 获取模块 GIL ============ */

    // 2003-2010: 如果有模块，获取全局解释器锁
    // 确保模块操作是线程安全的
    moduleAcquireGIL();

    /* ============ 3. 模块钩子 ============ */

    // 2013-2015: 调用模块的 afterSleep 钩子
    moduleFireServerEvent(REDISMODULE_EVENT_EVENTLOOP, REDISMODULE_SUBEVENT_EVENTLOOP_AFTER_SLEEP, NULL);
}
```

---

## 定时任务 serverCron

```c
// 文件位置：server.c:1442-1712

/*
 * serverCron() - Redis 的定时任务处理函数
 *
 * 【调用频率】
 * 默认每秒 10 次（server.hz = 10），可动态调整
 *
 * 【为什么需要定时任务？】
 * Redis 是单线程事件驱动，但有些任务需要定期执行：
 * - 清理过期键
 * - 检查客户端超时
 * - 触发 RDB/AOF 保存
 * - 主从复制心跳
 * - 更新统计信息
 *
 * 【主要工作】（按执行顺序）
 */
int serverCron(struct aeEventLoop *eventLoop, long long id, void *clientData) {
    int j;
    UNUSED(eventLoop);
    UNUSED(id);
    UNUSED(clientData);

    /* ============ 1. 软件看门狗 ============ */

    // 1450: 如果启用了看门狗，调度 SIGALRM 信号
    if (server.watchdog_period) watchdogScheduleSignal(server.watchdog_period);

    /* ============ 2. 动态调整 HZ ============ */

    // 1455-1465: 根据客户端数量动态调整定时任务频率
    // 客户端多时，Hz 提高；客户端少时，Hz 降低（节省 CPU）
    updateCachedTime(0);
    server.hz = server.config_hz;
    if (server.dynamic_hz) {
        int clients = listLength(server.clients);
        if (clients > MAX_CLIENTS_PER_CLOCK_TICK) {
            server.hz = server.config_hz * 2;
            if (server.hz > CONFIG_MAX_HZ) server.hz = CONFIG_MAX_HZ;
        }
    }

    /* ============ 3. 性能指标采样 ============ */

    // 1472-1494: 记录瞬时指标（命令数、网络 I/O）
    trackInstantaneousMetric(STATS_METRIC_COMMAND, server.stat_numcommands);
    trackInstantaneousMetric(STATS_METRIC_NET_INPUT, server.stat_net_input_bytes);
    trackInstantaneousMetric(STATS_METRIC_NET_OUTPUT, server.stat_net_output_bytes);

    /* ============ 4. LRU 时钟更新 ============ */

    // 1507: 更新 LRU 淘汰算法的时钟
    server.lruclock = getLRUClock();

    /* ============ 5. 更新内存统计 ============ */

    // 1509: 采样内存使用情况
    cronUpdateMemoryStats();

    /* ============ 6. 关机处理 ============ */

    // 1513-1528: 如果收到 SIGTERM/SIGINT，执行关机流程
    if (shouldShutdownAsap()) {
        // 1515-1520: 尝试优雅关机
        if (prepareForShutdown(server.shutdown_flags) == C_OK) {
            exit(0);
        }

        // 1522-1527: 如果优雅关机失败，强制退出
        serverLog(LL_WARNING, "Failed graceful shutdown. Exiting now.");
        exit(1);
    }

    /* ============ 7. 数据库信息显示 ============ */

    // 1531-1555: 每隔一段时间打印数据库状态（用于调试）
    if (server.verbosity <= LL_VERBOSE) {
        run_with_period(5000) {  // 每 5 秒执行一次
            for (j = 0; j < server.dbnum; j++) {
                long long keys = kvstoreSize(server.db[j].keys);
                long long expires = kvstoreSize(server.db[j].expires);
                if (keys || expires) {
                    serverLog(LL_VERBOSE, "DB %d: %lld keys (%lld volatile)",
                             j, keys, expires);
                }
            }
        }
    }

    /* ============ 8. 客户端处理 ============ */

    // 1558: 处理客户端超时、缓冲区调整等 ⭐
    clientsCron();

    /* ============ 9. 数据库维护 ============ */

    // 1561: 过期键清理、rehash、碎片整理 ⭐
    databasesCron();

    /* ============ 10. AOF 重写调度 ============ */

    // 1565-1570: 如果 AOF 文件太大，调度重写
    if (!hasActiveChildProcess() &&
        server.aof_rewrite_scheduled) {
        rewriteAppendOnlyFileBackground();
        server.aof_rewrite_scheduled = 0;
    }

    /* ============ 11. 检查子进程状态 ============ */

    // 1573-1616: 检查 RDB/AOF 子进程是否完成
    if (hasActiveChildProcess() || ldbPendingChildren()) {
        checkChildrenDone();
    } else {
        // 1580-1615: 如果没有子进程，检查是否需要触发保存
        for (j = 0; j < server.saveparamslen; j++) {
            struct saveparam *sp = server.saveparams + j;

            // 1585-1595: 检查 save 条件：时间 + 修改次数
            if (server.dirty >= sp->changes &&
                server.unixtime - server.lastsave > sp->seconds &&
                (server.unixtime - server.lastbgsave_try > CONFIG_BGSAVE_RETRY_DELAY ||
                 server.lastbgsave_status == C_OK)) {

                serverLog(LL_NOTICE, "%d changes in %d seconds. Saving...",
                         sp->changes, (int)sp->seconds);

                // 1600-1605: 触发后台保存
                rdbSaveInfo rsi, *rsiptr;
                rsiptr = rdbPopulateSaveInfo(&rsi);
                rdbSaveBackground(SLAVE_REQ_NONE, server.rdb_filename, rsiptr, RDBFLAGS_NONE);
                break;
            }
        }
    }

    /* ============ 12. AOF 刷盘处理 ============ */

    // 1623-1639: 处理延迟的 AOF fsync
    if (server.aof_state == AOF_ON) {
        flushAppendOnlyFile(0);
    }

    /* ============ 13. 暂停状态更新 ============ */

    // 1642: 更新客户端暂停状态（用于 CLIENT PAUSE）
    updateClientPauseExpirationTime();

    /* ============ 14. 复制 Cron ============ */

    // 1649-1653: 主从复制维护任务 ⭐
    // - 向主库发送 PING
    // - 重连断开的主库
    // - 向从库发送心跳
    run_with_period(1000) replicationCron();

    /* ============ 15. 集群 Cron ============ */

    // 1656-1658: Redis Cluster 维护任务
    if (server.cluster_enabled) run_with_period(100) clusterCron();

    /* ============ 16. Sentinel 定时器 ============ */

    // 1661: Sentinel 模式的定时任务
    if (server.sentinel_mode) sentinelTimer();

    /* ============ 17. MIGRATE 清理 ============ */

    // 1664-1666: 清理超时的 MIGRATE 连接
    run_with_period(1000) migrateCloseTimedoutSockets();

    /* ============ 18. 命令池维护 ============ */

    // 1669-1671: 维护待处理命令对象池
    run_with_period(1000) trimPendingCommandPool();

    /* ============ 19. 跟踪表调整 ============ */

    // 1677: 调整客户端跟踪表大小
    if (server.tracking_clients) trackingLimitUsedSlots();

    /* ============ 20. 定时 BGSAVE ============ */

    // 1686-1695: 执行定时的后台保存（如果配置了 save）
    // 这部分在第 11 步已经处理

    /* ============ 21. 模块 Cron ============ */

    // 1697-1699: 调用模块的定时任务
    modulesCron();

    /* ============ 22. 模块事件 ============ */

    // 1702-1705: 触发模块的 cron 事件
    moduleFireServerEvent(REDISMODULE_EVENT_CRON_LOOP, 0, NULL);

    /* ============ 23. 更新循环计数 ============ */

    // 1708: 增加循环计数器
    server.cronloops++;

    // 1710: 返回下次执行的时间间隔（毫秒）
    return 1000 / server.hz;  // 默认 100ms
}
```

### serverCron 的关键子函数

#### clientsCron() - 客户端维护

```c
// 文件位置：server.c:1161-1208

/*
 * clientsCron() - 客户端定时维护
 *
 * 【功能】
 * 定期检查所有客户端：
 * - 超时检测：关闭空闲太久的客户端
 * - 缓冲区调整：动态调整输入/输出缓冲区大小
 * - 内存跟踪：更新客户端内存使用统计
 */
void clientsCron(void) {
    // 1165-1170: 计算要检查的客户端数量
    // 不是每次都检查所有客户端，而是分批检查
    int numclients = listLength(server.clients);
    int iterations = numclients / server.hz;

    if (iterations < CLIENTS_CRON_MIN_ITERATIONS)
        iterations = (numclients < CLIENTS_CRON_MIN_ITERATIONS) ?
                     numclients : CLIENTS_CRON_MIN_ITERATIONS;

    // 1172-1205: 遍历客户端列表
    while (listLength(server.clients) && iterations--) {
        client *c;
        listNode *head;

        // 1177-1180: 轮转客户端列表（公平性）
        listRotateTailToHead(server.clients);
        head = listFirst(server.clients);
        c = listNodeValue(head);

        // 1182-1195: 处理单个客户端
        // - 检查超时
        // - 调整查询缓冲区
        // - 调整输出缓冲区
        // - 更新内存统计
        if (clientsCronHandleTimeout(c, server.unixtime)) continue;
        if (clientsCronResizeQueryBuffer(c)) continue;
        if (clientsCronResizeOutputBuffer(c)) continue;
        if (clientsCronTrackMemoryUsage(c)) continue;
    }
}
```

#### databasesCron() - 数据库维护

```c
// 文件位置：server.c:1213-1268

/*
 * databasesCron() - 数据库定时维护
 *
 * 【功能】
 * 定期维护数据库：
 * - 过期键清理：主动删除过期的键
 * - Rehash：渐进式 rehash，避免阻塞
 * - 碎片整理：减少内存碎片（如果启用）
 */
void databasesCron(void) {
    /* ============ 1. 过期键清理 ============ */

    // 1220-1230: 主动清理过期键 ⭐
    // 慢速循环：每次尝试清理一部分过期键
    if (server.active_expire_enabled) {
        if (iAmMaster()) {
            activeExpireCycle(ACTIVE_EXPIRE_CYCLE_SLOW);
        } else {
            expireReplicaKeys();
        }
    }

    /* ============ 2. Rehash ============ */

    // 1235-1250: 渐进式 rehash
    // 每次只 rehash 一小部分，避免阻塞
    if (!hasActiveChildProcess()) {
        // 1238-1245: 尝试 rehash 多个数据库
        int dbs_per_call = CRON_DBS_PER_CALL;  // 16

        // 1247-1250: 依次 rehash 每个数据库
        for (int j = 0; j < dbs_per_call; j++) {
            int work_done = incrementallyRehash(rehash_db);
            if (work_done) {
                // 1252: 这个数据库还需要继续 rehash
                break;
            } else {
                // 1254-1256: 这个数据库 rehash 完成，切换到下一个
                rehash_db++;
                rehash_db %= server.dbnum;
            }
        }
    }

    /* ============ 3. 碎片整理 ============ */

    // 1260-1268: 主动碎片整理（如果启用）
    if (server.active_defrag_enabled) {
        activeDefragCycle();
    }
}
```

---

## 命令处理流程

### 完整的命令处理链

```
[客户端] 发送命令 "SET key value"
             │
             ▼
    ┌────────────────────┐
    │ readQueryFromClient │  <-- networking.c
    │ (读取网络数据)       │
    └────────────────────┘
             │
             ▼
    ┌────────────────────┐
    │ processInputBuffer  │  <-- networking.c
    │ (解析协议)          │
    └────────────────────┘
             │
             ▼
    ┌────────────────────┐
    │ processCommand()    │  <-- server.c:4158 ⭐
    │ (命令预处理)        │
    └────────────────────┘
             │
             ▼
    ┌────────────────────┐
    │ call()              │  <-- server.c:3752 ⭐
    │ (执行命令)          │
    └────────────────────┘
             │
             ▼
    ┌────────────────────┐
    │ setCommand()        │  <-- t_string.c
    │ (SET 命令实现)      │
    └────────────────────┘
             │
             ▼
    ┌────────────────────┐
    │ addReply()          │  <-- networking.c
    │ (生成响应)          │
    └────────────────────┘
             │
             ▼
    ┌────────────────────┐
    │ sendReplyToClient   │  <-- networking.c
    │ (发送响应)          │
    └────────────────────┘
             │
             ▼
        [客户端] 收到 "+OK\r\n"
```

### processCommand() - 命令预处理

```c
// 文件位置：server.c:4158-4539

/*
 * processCommand() - 命令处理的核心函数
 *
 * 【调用时机】
 * 从 processInputBuffer() 调用，此时命令已经解析完成
 *
 * 【功能】
 * 在执行命令前做各种检查和预处理：
 * - 查找命令：从命令表中查找命令
 * - 权限检查：ACL 权限验证
 * - 参数检查：参数数量是否正确
 * - 状态检查：是否正在加载、内存是否足够
 * - 集群检查：键是否在正确的槽位
 * - 事务处理：MULTI/EXEC 队列
 *
 * 如果所有检查通过，调用 call() 执行命令
 *
 * 【返回值】
 * - C_OK: 命令已执行
 * - C_ERR: 命令被拒绝
 */
int processCommand(client *c) {
    /* ============ 1. 查找命令 ============ */

    // 4165-4175: 从命令表中查找命令 ⭐
    c->cmd = c->lastcmd = c->realcmd = lookupCommand(c->argv, c->argc);

    if (!c->cmd) {
        // 4180-4185: 命令不存在
        rejectCommandFormat(c, "unknown command '%s'", (char*)c->argv[0]->ptr);
        return C_OK;
    }

    /* ============ 2. 参数数量检查 ============ */

    // 4190-4210: 检查参数数量
    if ((c->cmd->arity > 0 && c->cmd->arity != c->argc) ||
        (c->cmd->arity < 0 && c->argc < -c->cmd->arity)) {

        // 4195-4200: 参数数量错误
        rejectCommandFormat(c, "wrong number of arguments for '%s' command",
                          c->cmd->fullname);
        return C_OK;
    }

    /* ============ 3. 权限检查（ACL）============ */

    // 4215-4230: ACL 权限验证
    int acl_errpos;
    int acl_retval = ACLCheckAllPerm(c, &acl_errpos);

    if (acl_retval != ACL_OK) {
        // 4220-4228: 权限不足
        if (acl_retval == ACL_DENIED_CMD) {
            rejectCommandFormat(c, "user '%s' has no permissions to run the '%s' command",
                              c->user->name, c->cmd->fullname);
        } else {
            rejectCommandFormat(c, "user '%s' has no permissions to access key '%s'",
                              c->user->name, (char*)c->argv[acl_errpos]->ptr);
        }
        return C_OK;
    }

    /* ============ 4. 集群模式检查 ============ */

    // 4235-4260: 如果启用了集群，检查键是否在正确的节点
    if (server.cluster_enabled &&
        !(c->flags & CLIENT_MASTER) &&
        !(c->flags & CLIENT_LUA) &&
        !(c->cmd->getkeys_proc == NULL && c->cmd->firstkey == 0 &&
          c->cmd->proc != execCommand)) {

        // 4245-4260: 检查键的槽位
        int error_code;
        clusterNode *n = getNodeByQuery(c, c->cmd, c->argv, c->argc, &hashslot, &error_code);

        if (n == NULL || n != server.cluster->myself) {
            // 4252-4258: 键不在当前节点，返回 MOVED/ASK
            clusterRedirectClient(c, n, hashslot, error_code);
            return C_OK;
        }
    }

    /* ============ 5. 内存检查 ============ */

    // 4265-4285: 检查内存是否足够
    if (server.maxmemory && !server.lua_timedout) {
        int out_of_memory = (performEvictions() == EVICT_FAIL);

        // 4270-4280: 如果内存不足且命令会增加内存使用
        if (out_of_memory &&
            (c->cmd->flags & CMD_DENYOOM ||
             (c->flags & CLIENT_MULTI && c->cmd->proc != execCommand &&
              c->cmd->proc != discardCommand))) {

            rejectCommand(c, shared.oomerr);
            return C_OK;
        }
    }

    /* ============ 6. 只读从库检查 ============ */

    // 4290-4305: 如果是只读从库，拒绝写命令
    if (server.masterhost && server.repl_slave_ro &&
        !(c->flags & CLIENT_MASTER) &&
        (c->cmd->flags & CMD_WRITE)) {

        rejectCommand(c, shared.roslaveerr);
        return C_OK;
    }

    /* ============ 7. Pub/Sub 模式检查 ============ */

    // 4310-4325: 如果客户端在 Pub/Sub 模式，只能执行订阅相关命令
    if ((c->flags & CLIENT_PUBSUB) &&
        c->cmd->proc != pingCommand &&
        c->cmd->proc != subscribeCommand &&
        c->cmd->proc != unsubscribeCommand &&
        c->cmd->proc != psubscribeCommand &&
        c->cmd->proc != punsubscribeCommand &&
        c->cmd->proc != quitCommand &&
        c->cmd->proc != resetCommand) {

        rejectCommandFormat(c,
            "Can't execute '%s': only (P|S)SUBSCRIBE / "
            "(P|S)UNSUBSCRIBE / PING / QUIT / RESET are allowed in this context",
            c->cmd->fullname);
        return C_OK;
    }

    /* ============ 8. 加载状态检查 ============ */

    // 4330-4345: 如果正在加载数据，只允许特定命令
    if (server.loading && !(c->cmd->flags & CMD_LOADING)) {
        rejectCommand(c, shared.loadingerr);
        return C_OK;
    }

    /* ============ 9. Lua 脚本超时检查 ============ */

    // 4350-4365: 如果 Lua 脚本超时，只允许 SHUTDOWN/SCRIPT KILL
    if (server.lua_timedout &&
        c->cmd->proc != authCommand &&
        c->cmd->proc != helloCommand &&
        c->cmd->proc != replconfCommand &&
        c->cmd->proc != shutdownCommand &&
        !(c->cmd->proc == scriptCommand &&
          c->argc == 2 &&
          tolower(((char*)c->argv[1]->ptr)[0]) == 'k')) {

        rejectCommand(c, shared.slowscripterr);
        return C_OK;
    }

    /* ============ 10. 事务队列处理 ============ */

    // 4370-4395: 如果在 MULTI 上下文中，将命令入队
    if (c->flags & CLIENT_MULTI &&
        c->cmd->proc != execCommand &&
        c->cmd->proc != discardCommand &&
        c->cmd->proc != multiCommand &&
        c->cmd->proc != watchCommand &&
        c->cmd->proc != resetCommand) {

        // 4380-4390: 将命令加入队列，不立即执行
        queueMultiCommand(c);
        addReply(c, shared.queued);
        return C_OK;
    }

    /* ============ 11. 执行命令 ============ */

    // 4400-4405: 调用 call() 执行命令 ⭐
    call(c, CMD_CALL_FULL);

    // 4407-4415: 命令执行后的处理
    c->woff = server.master_repl_offset;
    if (listLength(server.ready_keys))
        handleClientsBlockedOnKeys();

    return C_OK;
}
```

### call() - 命令执行核心

```c
// 文件位置：server.c:3752-3977

/*
 * call() - 命令执行的核心函数
 *
 * 【调用路径】
 * processCommand() -> call()
 *
 * 【功能】
 * 实际调用命令处理函数，并处理命令传播（AOF、复制）
 *
 * 【参数】
 * - c: 客户端对象
 * - flags: 调用标志
 *   - CMD_CALL_NONE: 无特殊处理
 *   - CMD_CALL_PROPAGATE_AOF: 传播到 AOF
 *   - CMD_CALL_PROPAGATE_REPL: 传播到从库
 *   - CMD_CALL_FULL: 完整处理（AOF + 复制）
 *
 * 【执行流程】
 * 1. 记录开始时间
 * 2. 调用命令处理函数（如 setCommand）
 * 3. 更新统计信息
 * 4. 传播命令（AOF + 复制）
 * 5. 记录慢日志
 * 6. 更新延迟直方图
 */
void call(client *c, int flags) {
    long long dirty;
    monotime call_timer;
    int client_old_flags = c->flags;

    /* ============ 1. 准备阶段 ============ */

    // 3760-3765: 记录开始时间（用于慢日志和延迟统计）
    elapsedStart(&call_timer);

    // 3767-3770: 记录执行前的 dirty 计数
    dirty = server.dirty;

    // 3772-3775: 增加命令执行标志
    c->flags |= CLIENT_EXECUTING_COMMAND;

    /* ============ 2. 调用命令处理函数 ============ */

    // 3780-3785: 调用命令的处理函数 ⭐
    // 比如 setCommand(), getCommand(), lpushCommand() 等
    c->cmd->proc(c);

    /* ============ 3. 执行后处理 ============ */

    // 3790-3795: 计算执行耗时
    const long long duration = elapsedUs(call_timer);
    c->duration += duration;

    // 3797-3800: 清除执行标志
    c->flags &= ~CLIENT_EXECUTING_COMMAND;

    // 3802-3805: 更新 dirty 计数（有多少个键被修改）
    dirty = server.dirty - dirty;
    if (dirty < 0) dirty = 0;

    /* ============ 4. 统计信息更新 ============ */

    // 3810-3820: 更新命令统计
    c->cmd->microseconds += duration;
    c->cmd->calls++;

    // 3822-3825: 更新服务器统计
    server.stat_numcommands++;

    /* ============ 5. 命令传播（AOF + 复制）============ */

    // 3830-3880: 如果命令修改了数据，需要传播 ⭐
    if (flags & CMD_CALL_PROPAGATE &&
        (c->flags & CLIENT_PREVENT_PROP) != CLIENT_PREVENT_PROP) {

        int propagate_flags = PROPAGATE_NONE;

        // 3835-3845: 判断是否需要传播到 AOF
        if (dirty && (flags & CMD_CALL_PROPAGATE_AOF))
            propagate_flags |= PROPAGATE_AOF;

        // 3847-3857: 判断是否需要传播到从库
        if (dirty && (flags & CMD_CALL_PROPAGATE_REPL))
            propagate_flags |= PROPAGATE_REPL;

        // 3860-3870: 立即传播命令
        if (propagate_flags != PROPAGATE_NONE) {
            propagateNow(c->db->id, c->argv, c->argc, propagate_flags);
        }
    }

    /* ============ 6. 慢日志记录 ============ */

    // 3885-3900: 如果命令执行太慢，记录到慢日志
    if (!(c->cmd->flags & CMD_SKIP_SLOWLOG) && duration > server.slowlog_log_slower_than) {
        slowlogPushEntryIfNeeded(c, c->argv, c->argc, duration);
    }

    /* ============ 7. 延迟直方图更新 ============ */

    // 3905-3910: 更新命令延迟直方图（用于 LATENCY 命令）
    if (c->cmd->latency_histogram) {
        hdr_record_value(c->cmd->latency_histogram, duration);
    }

    /* ============ 8. 恢复客户端标志 ============ */

    // 3915-3920: 恢复客户端标志
    c->flags &= ~(CLIENT_FORCE_AOF | CLIENT_FORCE_REPL | CLIENT_PREVENT_PROP);
    c->flags |= client_old_flags & (CLIENT_FORCE_AOF | CLIENT_FORCE_REPL | CLIENT_PREVENT_PROP);

    /* ============ 9. 触发模块事件 ============ */

    // 3925-3930: 通知模块命令执行完成
    moduleFireServerEvent(REDISMODULE_EVENT_COMMAND_FINISHED,
                         REDISMODULE_SUBEVENT_COMMAND_FINISHED_OK,
                         &cmd_info);
}
```

---

## 关机流程

```c
// 文件位置：server.c:4682-4750

/*
 * prepareForShutdown() - 准备关机
 *
 * 【调用时机】
 * - 收到 SIGTERM/SIGINT 信号
 * - 执行 SHUTDOWN 命令
 *
 * 【参数】
 * flags: 关机标志
 * - SHUTDOWN_NOFLAGS: 正常关机
 * - SHUTDOWN_SAVE: 强制保存
 * - SHUTDOWN_NOSAVE: 不保存
 * - SHUTDOWN_NOW: 立即关机，不等从库
 * - SHUTDOWN_FORCE: 强制关机，忽略错误
 *
 * 【关机步骤】
 * 1. 停止接受新连接
 * 2. 保存数据（RDB 或 AOF）
 * 3. 等待从库同步（可选）
 * 4. 关闭 AOF 文件
 * 5. 移除 PID 文件
 */
int prepareForShutdown(int flags) {
    /* ============ 1. 检查是否允许关机 ============ */

    // 4687-4692: 如果有子进程且不是强制关机，等待子进程完成
    if (hasActiveChildProcess() && !(flags & SHUTDOWN_FORCE)) {
        serverLog(LL_WARNING, "There is a child saving an .rdb. Waiting for it to complete.");
        return C_ERR;
    }

    /* ============ 2. 停止接受新连接 ============ */

    // 4694-4700: 关闭所有监听 socket
    for (int j = 0; j < CONN_TYPE_MAX; j++) {
        connListener *listener = &server.listeners[j];
        if (listener->ct == NULL) continue;

        for (int i = 0; i < listener->count; i++) {
            aeDeleteFileEvent(server.el, listener->fd[i], AE_READABLE);
            close(listener->fd[i]);
        }
    }

    /* ============ 3. 保存数据 ============ */

    // 4702-4730: 根据标志决定是否保存数据
    if (!(flags & SHUTDOWN_NOSAVE)) {
        serverLog(LL_NOTICE, "User requested shutdown...");

        // 4710-4720: 尝试 RDB 保存
        if (server.rdb_child_pid == -1) {
            serverLog(LL_NOTICE, "Saving the final RDB snapshot before exiting.");

            if (server.saveparamslen > 0) {
                rdbSaveInfo rsi, *rsiptr;
                rsiptr = rdbPopulateSaveInfo(&rsi);

                if (rdbSave(SLAVE_REQ_NONE, server.rdb_filename, rsiptr, RDBFLAGS_NONE) != C_OK) {
                    // 4715-4718: RDB 保存失败
                    if (!(flags & SHUTDOWN_FORCE)) {
                        serverLog(LL_WARNING, "Error trying to save the DB, can't exit.");
                        return C_ERR;
                    }
                }
            }
        }

        // 4722-4730: 刷新 AOF 缓冲区
        if (server.aof_state != AOF_OFF) {
            if (flushAppendOnlyFile(1) == C_ERR) {  // 1 = force fsync
                if (!(flags & SHUTDOWN_FORCE)) {
                    serverLog(LL_WARNING, "Error trying to flush the AOF, can't exit.");
                    return C_ERR;
                }
            }
        }
    }

    /* ============ 4. 移除 PID 文件 ============ */

    // 4732-4735: 删除 PID 文件
    if (server.pidfile) {
        serverLog(LL_NOTICE, "Removing the pid file.");
        unlink(server.pidfile);
    }

    /* ============ 5. 打印关机信息 ============ */

    // 4737-4740: 日志输出
    serverLog(LL_WARNING, "%s is now ready to exit, bye bye...",
             server.sentinel_mode ? "Sentinel" : "Redis");

    return C_OK;
}
```

---

## 完整启动流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                      Redis 完整启动流程                          │
└─────────────────────────────────────────────────────────────────┘

    程序启动（main）
         │
         ▼
    ┌────────────────────┐
    │ 第1阶段：基础初始化 │
    ├────────────────────┤
    │ - 随机数种子        │
    │ - initServerConfig()│   <-- 设置默认配置
    │ - ACL 初始化        │
    │ - 模块系统初始化    │
    │ - TLS 初始化        │
    └────────────────────┘
         │
         ▼
    ┌────────────────────┐
    │ 第2阶段：配置加载   │
    ├────────────────────┤
    │ - 解析命令行参数    │
    │ - 加载 redis.conf   │
    │ - 覆盖默认配置      │
    └────────────────────┘
         │
         ▼
    ┌────────────────────┐
    │ 第3阶段：系统检查   │
    ├────────────────────┤
    │ - 内存警告          │
    │ - 内核 bug 检测     │
    └────────────────────┘
         │
         ▼
    ┌────────────────────┐
    │ 第4阶段：守护进程化 │
    ├────────────────────┤
    │ - daemonize()       │
    │   (可选)            │
    └────────────────────┘
         │
         ▼
    ┌────────────────────────────────────┐
    │ 第5阶段：核心初始化（最重要！）     │
    ├────────────────────────────────────┤
    │ initServer()                       │
    │ ├─ 创建共享对象                     │
    │ ├─ 创建数据库数组（16个）           │
    │ ├─ 创建客户端列表                   │
    │ ├─ 创建事件循环 ⭐                 │
    │ ├─ 注册 beforeSleep/afterSleep     │
    │ ├─ 创建 serverCron 定时任务        │
    │ ├─ 初始化 Pub/Sub                  │
    │ ├─ 初始化复制缓冲区                 │
    │ ├─ 初始化 Lua 脚本环境              │
    │ ├─ 初始化慢日志                     │
    │ └─ 初始化模块系统                   │
    │                                    │
    │ clusterInit() (可选)               │
    │ └─ 初始化集群状态                   │
    │                                    │
    │ moduleLoadFromQueue()              │
    │ └─ 加载模块                         │
    │                                    │
    │ initListeners()                    │
    │ ├─ 绑定 TCP 端口（6379）           │
    │ ├─ 绑定 TLS 端口（可选）            │
    │ └─ 绑定 Unix Socket（可选）        │
    │                                    │
    │ InitServerLast()                   │
    │ └─ 启动 IO 线程                     │
    └────────────────────────────────────┘
         │
         ▼
    ┌────────────────────────────────────┐
    │ 第6阶段：加载数据 ⭐                │
    ├────────────────────────────────────┤
    │ loadDataFromDisk()                 │
    │ ├─ 优先加载 AOF 文件                │
    │ │  └─ loadAppendOnlyFiles()        │
    │ │                                  │
    │ └─ 其次加载 RDB 文件                │
    │    └─ rdbLoad()                    │
    │                                    │
    │ 打开 AOF 文件（如果启用）           │
    │ └─ openAofForAppend()              │
    └────────────────────────────────────┘
         │
         ▼
    ┌────────────────────────────────────┐
    │ 第7阶段：启动监听                   │
    ├────────────────────────────────────┤
    │ 为每个监听 socket 创建文件事件      │
    │ └─ aeCreateFileEvent()             │
    │    └─ 回调：acceptTcpHandler()     │
    │                                    │
    │ 打印启动信息                        │
    │ └─ "Ready to accept connections"   │
    └────────────────────────────────────┘
         │
         ▼
    ┌────────────────────────────────────┐
    │ 第8阶段：进入事件循环 ⭐            │
    ├────────────────────────────────────┤
    │ aeMain(server.el)                  │
    │                                    │
    │ ┌──────────────────────────────┐   │
    │ │   事件循环（永不退出）        │   │
    │ │                              │   │
    │ │   1. beforeSleep()           │   │
    │ │      - 写 AOF                │   │
    │ │      - 发送响应              │   │
    │ │      - 清理过期键            │   │
    │ │                              │   │
    │ │   2. aeProcessEvents()       │   │
    │ │      - 等待网络事件          │   │
    │ │      - 处理客户端请求        │   │
    │ │      - 执行定时任务          │   │
    │ │                              │   │
    │ │   3. afterSleep()            │   │
    │ │      - 更新时间缓存          │   │
    │ │      - 获取模块 GIL          │   │
    │ │                              │   │
    │ │   循环回到 1                 │   │
    │ └──────────────────────────────┘   │
    └────────────────────────────────────┘
```

---

## 总结

### server.c 的核心函数

| 函数名 | 行号 | 调用时机 | 功能 |
|--------|------|----------|------|
| **main()** | 7455-7799 | 程序启动 | 主函数，启动流程 |
| **initServerConfig()** | 2223-2356 | main 早期 | 初始化配置默认值 |
| **initServer()** | 2794-3022 | main 中期 | 初始化核心组件 |
| **initListeners()** | 3024-3094 | main 后期 | 绑定监听端口 |
| **loadDataFromDisk()** | 7173-7288 | main 后期 | 加载 RDB/AOF |
| **beforeSleep()** | 1799-1986 | 事件循环前 | 睡眠前处理 |
| **afterSleep()** | 1991-2029 | 事件循环后 | 唤醒后处理 |
| **serverCron()** | 1442-1712 | 定时任务 | 每秒 10 次 |
| **processCommand()** | 4158-4539 | 命令解析后 | 命令预处理 |
| **call()** | 3752-3977 | processCommand 后 | 执行命令 |
| **prepareForShutdown()** | 4682-4750 | 关机时 | 关机准备 |

### Redis 的三大核心循环

1. **事件循环**（aeMain）
   - 处理网络 I/O
   - 执行定时任务
   - 永不返回

2. **定时任务**（serverCron）
   - 每秒 10 次
   - 清理过期键
   - 维护数据库
   - 触发保存

3. **命令处理**（processCommand -> call）
   - 解析命令
   - 权限检查
   - 执行命令
   - 传播命令

---

**文件位置**：`D:\projects\redis\analysis\server.c核心注释.md`

**下一步**：如果需要其他文件的注释（如 `networking.c`, `db.c`, `t_string.c`），随时告诉我！😊
