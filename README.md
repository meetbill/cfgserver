# cfgServer
【星控】基于 Butterfly 框架 + raft 协议实现一个 Redis 集群的管控服务
<!-- vim-markdown-toc GFM -->

* [1 前言](#1-前言)
    * [1.1 Arch](#11-arch)
    * [1.2 目标](#12-目标)
    * [1.3 功能](#13-功能)
* [2 调研](#2-调研)
    * [2.1 Redis cluster cc](#21-redis-cluster-cc)
        * [2.1.1 路由管理](#211-路由管理)
        * [2.1.2 单机止损](#212-单机止损)
        * [2.1.3 集群扩缩容](#213-集群扩缩容)
        * [2.1.4 自身高可用](#214-自身高可用)
    * [2.2 MongoDB config servers](#22-mongodb-config-servers)
    * [2.3 SSDB cfgServer](#23-ssdb-cfgserver)
* [3 设计思路与折衷](#3-设计思路与折衷)
    * [3.1 一致性](#31-一致性)
    * [3.2 CfgServer 区分主从](#32-cfgserver-区分主从)
    * [3.2 Proxy 与 cfgServer 交互](#32-proxy-与-cfgserver-交互)
* [4 总体设计](#4-总体设计)
    * [4.1 架构图](#41-架构图)
* [5 wiki](#5-wiki)

<!-- vim-markdown-toc -->

# 1 前言

## 1.1 Arch

```
  +---------+     +---------+     +---------+         +-------------------+
  |twemproxy|     |twemproxy|     |twemproxy|         |                   |
  +---------+     +---------+     +---------+    \    |  +-------------+  |
        | \          /   \          /  |          \   |  |cfg primary  |  |
        |    \     /       \     /     |           \  |  +-------------+  |
        |       X             X        |            \ |                   |
        |    /     \       /     \     |             \|  +-------------+  |
        |  /          \ /          \   |              |  |cfg secondary|  |
  +------------------+   +------------------+        /|  +-------------+  |
  |    +--------+    |   |    +--------+    |       / |                   |
  |    | master |    |   |    | master |    |      /  |  +-------------+  |
  |    +--------+    |   |    +--------+    |     /   |  |cfg secondary|  |
  | +-----+  +-----+ |   | +-----+  +-----+ |    /    |  +-------------+  |
  | |slave|  |slave| |   | |slave|  |slave| |         |                   |
  | +-----+  +-----+ |   | +-----+  +-----+ |         |                   |
  +------------------+   +------------------+         +-------------------+
```
## 1.2 目标

仅管理单地域 Redis 集群，并保证高可靠性和高可用性。

## 1.3 功能

> * 路由管理
> * 单机止损
> * 集群扩缩容
> * 配置管理

# 2 调研

## 2.1 Redis cluster cc

https://github.com/meetbill/cc

### 2.1.1 路由管理

拓扑信息由 redis cluster 进行维护

### 2.1.2 单机止损

感知层及决策层由 redis cluster 处理

cc 获取一致的拓扑信息，然后进行执行操作，即 cc 主要负责执行层

### 2.1.3 集群扩缩容

通过下发命令到 redis cluster 集群进行迁移 slot 进行扩缩容

### 2.1.4 自身高可用

拓扑信息在 redis cluster 中，本身是无状态节点，通过主备方式实现（部署多个节点，然后在 zookeeper 上进行注册抢主）

## 2.2 MongoDB config servers

https://github.com/mongodb/mongo

## 2.3 SSDB cfgServer

https://github.com/ksarch-saas/cfgServer

# 3 设计思路与折衷

## 3.1 一致性

强一致模型

> * 优点是社区实现很多，开发成本低，控制模块集群规模较小，性能可接受
> * 缺点是跨地域支持较差

## 3.2 CfgServer 区分主从

当 Redis 集群主实例故障时，cfgServer 需要执行切主操作，所以需要进行区分主从，通过外部依赖进行选主。租约 + 锁的形式实现简单高效

## 3.2 Proxy 与 cfgServer 交互

Proxy 需要获取集群拓扑信息，业界有很多实现，可以分为两类。

> * 第一类主动更新，Proxy 主动拉取信息
> * 第二类被动更新，MetaDB 发现变化后通知 Proxy 更新

备注：启动 Proxy 时，需要拉取到 topo 信息

# 4 总体设计

## 4.1 架构图

cfgServer + metaDB 方式，将计算与数据分离


# 5 wiki

https://github.com/meetbill/cfgserver/wiki
