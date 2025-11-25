# Redis 源码模块分析

本文档总结了 Redis 源码项目的主要模块组成和架构设计。

## 一、核心模块 (src/ 目录)

### 1. 服务器核心
- `server.c/h`       - Redis 服务器主程序和核心数据结构定义
- `networking.c`     - 客户端连接和网络 I/O 处理
- `connection.c`     - 连接抽象层

### 2. 事件驱动系统
- `ae.c/h`           - 异步事件循环核心 (Async Event Library)
- `ae_epoll.c`       - Linux epoll 实现
- `ae_kqueue.c`      - BSD/macOS kqueue 实现
- `ae_select.c`      - 通用 select 实现
- `ae_evport.c`      - Solaris event port 实现

### 3. 数据库操作
- `db.c`             - 数据库操作核心 (GET/SET 等命令)
- `object.c`         - Redis 对象系统 (robj/kvobj)
- `kvstore.c/h`      - 基于槽位的键值存储

### 4. 数据结构实现
- `sds.c/h`          - 简单动态字符串 (Simple Dynamic String)
- `dict.c/h`         - 哈希表实现
- `adlist.c/h`       - 双向链表
- `quicklist.c/h`    - 快速列表 (List 类型底层实现)
- `listpack.c/h`     - 列表打包 (紧凑编码)
- `ziplist.c/h`      - 压缩列表 (已弃用)
- `intset.c/h`       - 整数集合
- `rax.c/h`          - 基数树 (Radix Tree)

### 5. 数据类型实现
- `t_string.c`       - 字符串类型命令
- `t_list.c`         - 列表类型命令
- `t_set.c`          - 集合类型命令
- `t_zset.c`         - 有序集合命令
- `t_hash.c`         - 哈希表类型命令
- `t_stream.c`       - 流数据结构命令

### 6. 持久化系统
- `rdb.c`            - RDB 快照持久化
- `aof.c`            - AOF (Append-Only File) 追加文件持久化

### 7. 集群与高可用
- `cluster.c`        - Redis 集群核心实现
- `cluster_legacy.c` - 传统集群实现
- `cluster_asm.c`    - 集群异步消息处理
- `replication.c`    - 主从复制
- `sentinel.c`       - Redis Sentinel 高可用系统

### 8. 扩展性
- `module.c`         - 模块系统 API (通过 redismodule.h 导出 C API)
- `eval.c`           - Lua 脚本执行
- `functions.c`      - Redis Functions
- `function_lua.c`   - Lua 函数支持

### 9. 内存管理
- `zmalloc.c/h`      - 内存分配封装 (跟踪内存使用)
- `lazyfree.c`       - 异步内存释放
- `defrag.c`         - 内存碎片整理
- `evict.c`          - 内存淘汰策略

### 10. 其他核心功能
- `acl.c`            - 访问控制列表 (Access Control List, 130KB 大型模块)
- `config.c`         - 配置系统 (158KB)
- `commands.c`       - 命令注册
- `blocked.c`        - 客户端阻塞操作
- `bio.c`            - 后台 I/O 线程 (Background I/O)
- `iothread.c`       - I/O 多线程
- `latency.c`        - 延迟监控
- `debug.c`          - 调试功能
- `geo.c/geohash.c`  - 地理位置功能
- `hyperloglog.c`    - HyperLogLog 基数统计
- `bitops.c`         - 位操作命令
- `expire.c`         - 过期键处理


## 二、扩展模块 (modules/ 目录)

这些模块需要使用 `BUILD_WITH_MODULES=yes` 标志编译:

- `redisjson`        - JSON 数据类型支持 (需要 Rust 工具链)
- `redistimeseries`  - 时序数据存储
- `redisearch`       - 查询引擎 (全文搜索、向量搜索)
- `redisbloom`       - 概率数据结构 (Bloom filter, Cuckoo filter)
- `vector-sets`      - 向量集合和向量嵌入


## 三、依赖库 (deps/ 目录)

> 注意: make 不会自动重建依赖项,当依赖项源码更改时需要使用 `make distclean` 清理

- `jemalloc`         - 内存分配器 (Linux 默认),支持主动碎片整理
- `hiredis`          - Redis C 客户端库 (用于 redis-cli、redis-benchmark 和 Sentinel)
- `linenoise`        - readline 替代品 (用于 CLI 命令行交互)
- `lua`              - Lua 5.1 脚本引擎
- `hdr_histogram`    - 命令延迟跟踪


## 四、测试模块 (tests/ 目录)

测试使用 TCL 编写,支持标签系统过滤:

- `tests/unit/`          - 单元测试
- `tests/integration/`   - 集成测试
- `tests/cluster/`       - 集群测试
- `tests/sentinel/`      - Sentinel 高可用测试
- `tests/modules/`       - 模块 API 测试

运行测试命令:
```bash
./runtest              - 运行所有测试
./runtest-cluster      - 集群测试
./runtest-sentinel     - Sentinel 测试
./runtest-moduleapi    - 模块 API 测试
./runtest --tags TAG   - 运行特定标签的测试
```


## 总体架构特点

Redis 采用围绕单线程事件驱动核心的 C 语言实现,可选 I/O 多线程。
架构核心特点包括:

### 1. 事件驱动设计
- 基于 ae 事件循环
- 使用 epoll/kqueue 进行 I/O 多路复用

### 2. 统一对象模型
- 所有数据存储为 robj (redisObject)
- 使用引用计数进行内存管理

### 3. 内存优先方法
- 内存中的数据结构服务器
- 使用 jemalloc 内存分配器
- 支持主动内存碎片整理

### 4. 扩展框架
- 模块 API 支持 C 扩展
- 嵌入式 Lua 脚本支持
- Redis Functions 支持

### 5. 分布式能力
- 内置集群支持
- 主从复制
- Sentinel 高可用

### 6. 持久化机制
- RDB 快照持久化
- AOF 追加文件持久化
- 两者可组合使用

所有核心组件通过 `src/server.c` 中的全局 `server` 变量
(类型为 `struct redisServer`) 进行协调管理。


## 编译说明

基本构建:
```bash
make                                # 构建核心数据结构
make BUILD_WITH_MODULES=yes         # 构建包含所有模块
make BUILD_TLS=yes                  # 构建支持 TLS
make -j $(nproc)                    # 并行构建
```

完整构建 (推荐用于开发):
```bash
export BUILD_TLS=yes BUILD_WITH_MODULES=yes INSTALL_RUST_TOOLCHAIN=yes
make -j $(nproc) all
```

清理构建缓存:
```bash
make distclean
```


## 许可证信息

Redis 8.0+ 采用 RSALv2/SSPLv1/AGPLv3 三重许可证


文档生成时间: 2025-11-25
Redis 版本: 8.4.0
