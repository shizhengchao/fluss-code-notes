# Fluss 源码阅读笔记 - RPC 与启动架构分析

本文整理了对 Fluss 源码中 RPC 模块和启动架构的分析问答。

---

## 目录

- [1. NettyServer 实现分析](#1-nettyserver-实现分析)
- [2. Endpoint 与 Server 的关系](#2-endpoint-与-server-的关系)
- [3. NettyClient 连接机制](#3-nettyclient-连接机制)
- [4. 多个 Endpoint 如何选择](#4-多个-endpoint-如何选择)
- [5. 启动脚本分析](#5-启动脚本分析)
- [6. 架构分工：Coordinator vs TabletServer](#6-架构分工coordinator-vs-tabletserver)
- [7. 客户端查询流程](#7-客户端查询流程)
- [8. 元数据缓存机制](#8-元数据缓存机制)

---

## 1. NettyServer 实现分析

### 启动流程图

```
+------------------------------------------+
|  创建NettyServer实例                      |
|  - 保存配置/端点列表                      |
|  - 加载协议插件(SPI)                      |
|  - 创建RequestProcessorPool              |
+------------------+-----------------------+
                   |
                   v
+------------------------------------------+
|  NettyServer.start()                     |
|  - 创建acceptorGroup (1线程)             |
|  - 创建selectorGroup (N网络线程)         |
|  - 启动workerPool                        |
+------------------+-----------------------+
                   |
                   v
+------------------------------------------+
|  建立 监听器名称 -> 协议插件 映射          |
+------------------+-----------------------+
                   |
                   v
+------------------------------------------+
|  遍历每个Endpoint                      ◄───┘
|  ↓
|  startEndpoint(endpoint)
|
|  1. 创建ServerBootstrap
|  2. 配置Netty参数
|  3. 创建协议对应的ChannelHandler
|  4. bootstrap.bind(host, port) → 真正启动ServerSocket监听
|  5. 保存Channel和绑定Endpoint到列表
|
+------------------+-----------------------+
                   |
                   ├────────────────────────┘
                   |
                   v (所有端点处理完成)
+------------------------------------------+
|  注册Netty度量指标                       |
|  设置isRunning = true                    |
+------------------+-----------------------+
                   |
                   v
                结束
```

### 线程模型

- **acceptorGroup**: 单线程，专门处理连接接受
- **selectorGroup**: 多线程，处理IO读写，数量由 `NETTY_SERVER_NUM_NETWORK_THREADS` 配置
- **workerPool**: 独立线程池，处理业务逻辑，数量由 `NETTY_SERVER_NUM_WORKER_THREADS` 配置

---

## 2. Endpoint 与 Server 的关系

### 核心结论

> **一个 NettyServer 可以绑定多个 Endpoint**。每个 Endpoint 是一个监听地址（IP + 端口 + 监听器名称），对应不同协议。

### 为什么需要多个 Endpoint？

Fluss 支持多协议，同一个 Server 进程可以对外暴露多种服务：

```
┌───────────── NettyServer ───────────────────┐
│                                              │
│  Endpoint-1: 0.0.0.0:8081  "fluss-internal" │
│   → 供 Fluss 集群内部节点通信                │
│                                              │
│  Endpoint-2: 0.0.0.0:9092  "kafka"          │
│   → 供 外部Kafka客户端 连接读写               │
│                                              │
└──────────────────────────────────────────────┘
```

### IP 参数的作用

IP 参数不是绑定到其他机器，而是选择**本机哪个网卡**监听：

| 绑定IP | 含义 |
|--------|------|
| `0.0.0.0` | 监听所有网卡，接受任何来源连接 |
| `127.0.0.1` | 只接受本机连接，外部无法访问 |
| `192.168.1.100` | 只接受内网连接 |

---

## 3. NettyClient 连接机制

### 连接完整流程

```
开始
  │
  ▼
+------------------------------------------+
| 用户调用 connect(ServerNode)              |
+------------------+-----------------------+
                   |
                   ▼
+------------------------------------------+
| 连接已在 connections 缓存?                 |
|        │                │
|        是                否               |
|        │                │                 │
|   返回已有连接    创建新 ServerConnection  │
|                         │                 │
|                         ▼                 │
|  bootstrap.connect(node.host(), node.port()) ← 真正发起TCP连接
|                         │                 │
|                         ▼                 │
|                   TCP连接建立成功?         │
|                         │        │        │
|                    成功          失败     │
|                         │        │        │
|                         ▼        ▼        │
|                  状态 → CHECKING_API_VERSIONS
|                  发送 ApiVersionsRequest  │
|                  (协商API版本)             │
|                         │                 │
|                  收到响应，验证Server类型  │
|                  保存API版本信息           │
|                         │                 │
|                  开始认证                  │
|                  状态 → AUTHENTICATING    │
|                  发送 AuthenticateRequest │
|                         │                 │
|                  认证完成 → READY          │
|                  发送排队中的请求          │
+------------------+-----------------------+
                   |
                   ▼
              返回 isReady() = true
              连接完成，可以发送请求
```

### 连接是否对应 Endpoint？

**✅ 是的**。`ServerNode` 的 `host()` 和 `port()` 正好来自 Server 端一个 Endpoint：

```java
// ServerInfo.java:88
return new ServerNode(id, endpoint.getHost(), endpoint.getPort(), serverType, rack);
```

### ServerNode 从哪里来？

完整链路：

```
Server 端启动
  └─ 绑定多个 Endpoint
  └─ 注册到 Coordinator
  └─ Coordinator 保存 ServerInfo（包含所有 Endpoint）

Client 需要连接
  └─ 向 Coordinator 查询 "给我某个节点的fluss-internal端点"
  └─ Coordinator 取出对应 Endpoint → 包装成 ServerNode 返回
  └─ Client connect(serverNode) → 连接对应 host:port
```

---

## 4. 多个 Endpoint 如何选择

> **不是随机选择，是按协议选择**。一个 client 只连接它需要的那个 endpoint。

| Client 类型 | 连接哪个 endpoint | 连接数量 |
|------------|------------------|---------|
| Coordinator ↔ TabletServer 内部通信 | `fluss-internal` | 1个 |
| 外部Kafka客户端读写 | `kafka` | 1个 |
| 需要同时访问两种协议 | 两个都连 | 2个（少见） |

---

## 5. 启动脚本分析

### 各脚本作用

| 脚本 | 作用 | 启动组件 |
|------|------|---------|
| `config.sh` | 共享环境配置，被其他脚本引入 | 不启动组件 |
| `coordinator-server.sh` | 启动/停止 Coordinator | **Coordinator Server**（集群管理者） |
| `tablet-server.sh` | 启动/停止 Tablet Server | **Tablet Server**（存储数据节点） |
| `fluss-daemon.sh` | 通用后台守护进程启动器 | 被调用，不直接启动 |
| `fluss-console.sh` | 通用前台启动器（调试用） | 被调用，不直接启动 |
| `local-cluster.sh` | 一键启动本地开发集群 | Zookeeper + Coordinator + TabletServer |

### Fluss 架构

```
┌────────────┐     ┌──────────────┐     ┌──────────────┐
│ Zookeeper │ ←→ │ Coordinator  │ ←→ │ TabletServer │
│  (协调选主)│     │  (元数据管理) │     │  (数据存储)  │
└────────────┘     └──────────────┘     └──────────────┘
```

- **Zookeeper**: 分布式协调，选主，存储集群元数据
- **Coordinator Server**: 管理集群元数据（分区分配，节点列表）
- **Tablet Server**: 实际存储数据，处理客户端读写请求

### 如何一键启动

```bash
# 启动完整本地集群
bin/local-cluster.sh start

# 停止集群
bin/local-cluster.sh stop
```

这个脚本会自动按顺序启动：
1. 内置 Zookeeper
2. Coordinator Server
3. Tablet Server（自动使用随机端口避免冲突）

---

## 6. 架构分工：Coordinator vs TabletServer

### 问题

> Coordinator 会处理客户端请求吗？还是客户端直接与 TabletServer 通信？为什么没有启动 NettyClient 的脚本？

### 分工

| 请求类型 | 谁处理 |
|---------|--------|
| 元数据请求（创建topic，查询分区分布）| **Coordinator** |
| 实际数据读写（生产/消费） | **客户端直接和 TabletServer 通信** |

### 为什么没有 NettyClient 启动脚本

**NettyClient 不是独立进程，它是类库，嵌入在使用它的进程中：**

```
你的应用进程 (Flink/Spark)
    └─┐
      ├─ 创建 NettyClient
      ├─ 连接 Coordinator 获取元数据
      ├─ 连接各个 TabletServer 读写数据
      └─ 应用退出时关闭 NettyClient

↑↑↑ NettyClient 就在你的应用进程里面，不需要单独启动进程。
```

谁会用到 NettyClient：
- Coordinator 需要和 TabletServer 通信 → Coordinator 进程内有 NettyClient
- TabletServer 需要和 Coordinator 通信 → TabletServer 进程内有 NettyClient
- 外部客户端应用 → 应用进程内有 NettyClient

所以：**NettyClient 永远作为某个进程的一部分运行，不需要单独的启动脚本**。

---

## 7. 客户端查询流程

### Flink SQL 点查一条数据的完整流程

```
Flink SQL → Fluss Connector
     │
     ├─ 第一次查询：缓存未命中
     │   └─→ NettyClient → Coordinator："表X的key属于哪个分区，在哪个节点?"
     │   ←─ Coordinator 返回："分区0 在 TabletServer A (host:port)"
     │   ←─  Connector 缓存元数据到本地
     │
     ├─ 根据key计算分区
     ├─ 从缓存得到 TabletServer 地址
     └─→ NettyClient → 直接连 TabletServer A 执行查询
         ←─ TabletServer 返回数据
         ←─ Flink 返回结果给用户

第二次查询同样数据：
     ├─ 直接从缓存拿地址
     └─→ 直接连 TabletServer，不走 Coordinator
```

### 架构特点

- ✅ 元数据由 Coordinator 集中管理
- ✅ 数据读写**不经过 Coordinator**，客户端直连 TabletServer
- ✅ 元数据缓存，避免每次都问 Coordinator
- ✅ Coordinator 不会成为瓶颈

这就是典型的"**元数据集中管理，数据直接访问**"的分布式存储架构，类似 HDFS (NameNode + DataNode)、Kafka (Controller + Broker)。

---

## 8. 元数据缓存机制

### 缓存分多层，每个组件都缓存

### 1. NettyClient 缓存连接

**代码位置**: `NettyClient.java:72`

```java
// key: serverUid, value: 连接
private final Map<String, ServerConnection> connections;

// 只有缓存不命中才新建连接
private ServerConnection getOrCreateConnection(ServerNode node) {
    String serverId = node.uid();
    return connections.computeIfAbsent(
        serverId,
        ignored -> new ServerConnection(bootstrap, node, ...);
    );
}
```

证明：**同一个 ServerNode 只会建立一个连接，连接一直缓存着重复使用**。

### 2. TabletServer 缓存整个集群元数据

**代码位置**: `TabletServerMetadataCache.java`

```java
public class TabletServerMetadataCache implements ServerMetadataCache {
    // 缓存整个集群的元数据快照
    private volatile ServerMetadataSnapshot serverMetadataSnapshot;

    // 直接从缓存拿 ServerNode，不需要问 Coordinator
    @Override
    public Optional<ServerNode> getTabletServer(int serverId, String listenerName) {
        return serverMetadataSnapshot.getAliveTabletServersById(serverId, listenerName);
    }
}
```

### 3. 客户端 Connector 缓存元数据

Flink/Spark Connector 在客户端进程缓存元数据："这张表的这个分区在哪个节点"，避免每次查询都访问 Coordinator。

### 总结缓存层次

| 缓存层次 | 缓存内容 | 位置 |
|---------|---------|------|
| 连接缓存 | 已建立的 TCP 连接 | `NettyClient` 自身 (`connections` 变量) |
| 元数据缓存 | 表/分区/节点分布 | 客户端连接器 (Flink/Spark) + `TabletServerMetadataCache` |
| 集群元数据 | 所有节点信息 | Coordinator 和所有 TabletServer 都有本地缓存 |

---

## 最终架构总图

```
┌──────────────────────────────────────────────────────────────┐
│                     客户端应用 (Flink)                         │
│                                                               │
│  ┌─────────────────────┐                                       │
│  │   Fluss Connector  │                                       │
│  │   缓存: 表 → 分区 → 节点地址                            │
│  └─────────────────────┘                                       │
│          │                                                       │
│          ├─→ [未缓存] 请求元数据 → Coordinator                  │
│          └─→ [已缓存] 直连 → TabletServer                       │
│                                                               │
└──────────────────────────────────────────────────────────────┘
          │                       │
          │ 元数据请求            │ 数据读写请求
          ▼                       ▼
┌──────────────────┐    ┌──────────────────┐
│   Coordinator    │    │   TabletServer   │
│  (元数据管理)    │    │   (数据存储)     │
│  ┌────────────┐  │    │  ┌────────────┐  │
│  │ 元数据缓存 │  │    │  │ 元数据缓存 │  │
│  └────────────┘  │    │  └────────────┘  │
└──────────────────┘    └──────────────────┘
```