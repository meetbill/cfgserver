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
        * [2.1.4 配置管理](#214-配置管理)
        * [2.1.5 自身高可用](#215-自身高可用)
    * [2.2 MongoDB config servers](#22-mongodb-config-servers)
    * [2.3 SSDB cfgServer](#23-ssdb-cfgserver)
* [3 设计思路与折衷](#3-设计思路与折衷)
    * [3.1 一致性](#31-一致性)
    * [3.2 CfgServer 区分主从](#32-cfgserver-区分主从)
    * [3.3 Proxy 与 cfgServer 交互](#33-proxy-与-cfgserver-交互)
    * [3.4 MetaDB 选取](#34-metadb-选取)
* [4 总体设计](#4-总体设计)
    * [4.1 架构图](#41-架构图)
    * [4.2 cfgServer 概要设计](#42-cfgserver-概要设计)
    * [4.3 MetaDB 概要设计](#43-metadb-概要设计)
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

Controller 工作在地域级别，往往一个地域有多个逻辑机房或物理机房，一个地域可以部署多个 controller，但是同一地域内只有一个实际工作。

当实际工作的 controller 挂掉后，集群会进行重新选择新主。

> 功能
```
节点读写权限控制
节点主从切换
主故障自动选主
从故障自动封禁
数据迁移控制
数据 Rebalance
```

### 2.1.1 路由管理

拓扑信息由 redis cluster 进行维护

### 2.1.2 单机止损

感知层及决策层由 redis cluster 处理

cc 获取一致的拓扑信息，然后进行执行操作，即 cc 主要负责执行层

### 2.1.3 集群扩缩容

通过下发命令到 redis cluster 集群进行迁移 slot 进行扩缩容

### 2.1.4 配置管理

管理集群相关的配置，比如是否进行从库自动封禁，是否自动切主

### 2.1.5 自身高可用

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

多个 CfgServer 均可正常进行读写，只是由 cfgServer 执行操作的模块需要抢主进行单一操作

## 3.3 Proxy 与 cfgServer 交互

Proxy 需要获取集群拓扑信息，业界有很多实现，可以分为两类。

> * 第一类主动更新，Proxy 主动拉取信息
> * 第二类被动更新，cfgServer 发现变化后通知 Proxy 更新

备注：启动 Proxy 时，需要拉取到 topo 信息

折衷考虑，选择被动更新方式，通过版本号区分是否需要更新

## 3.4 MetaDB 选取

MetaDB 用于存放 topo 数据以及配置数据，可以使用如下方式实现

> * 自研模块：基于 https://github.com/bakwc/PySyncObj 上实现 raft 模块
> * MongoDB : 自带副本集，可保障强一致性和自身高可用
> * Redis   : 主从模式，无法保障自身高可用，如果是多地域管控需求时，可以使用此方式

MongoDB 副本集，可以保障自身高可用，可满足需求, 故选择使用 MongoDB

# 4 总体设计

## 4.1 架构图

cfgServer + metaDB 方式，将计算与数据分离

## 4.2 cfgServer 概要设计

## 4.3 MetaDB 概要设计


# 5 wiki

https://github.com/meetbill/cfgserver/wiki
