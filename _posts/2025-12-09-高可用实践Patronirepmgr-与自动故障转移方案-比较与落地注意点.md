---
layout: post
title: PostgreSQL高可用实战：Patroni vs repmgr，如何选择与落地？
categories: [PG]
description: 本文深入对比PostgreSQL两大主流高可用方案Patroni与repmgr的核心原理、架构差异与适用场景，并结合实际生产经验，详细阐述自动故障转移的落地要点、常见陷阱与最佳实践，为构建稳定可靠的数据库集群提供决策参考。
keywords: PostgreSQL, 高可用, Patroni, repmgr, 故障转移
comments: false
---

# PostgreSQL高可用实战：Patroni vs repmgr，如何选择与落地？

在当今以数据驱动的业务环境中，数据库的高可用性（High Availability, HA）不再是可选项，而是保障服务连续性的生命线。PostgreSQL作为功能强大的开源关系型数据库，其高可用生态也日趋成熟。在众多方案中，**Patroni** 和 **repmgr** 脱颖而出，成为构建PostgreSQL HA集群的两大主流选择。本文将深入剖析两者的技术内核，比较其优劣，并分享从架构设计到生产落地的关键注意点。

## 一、 核心方案概览：两种不同的哲学

### 1. repmgr：专注复制与故障转移的“实干家”

**repmgr**（Replication Manager）是一个用于管理PostgreSQL流复制集群的套件。它更像是一个“增强工具包”，核心功能围绕**复制监控**和**手动/自动故障转移**展开。

*   **架构**： 相对轻量，主要由`repmgrd`守护进程（负责监控和自动故障转移）和一系列管理工具（如`repmgr` CLI）组成。
*   **共识与领导者选举**： repmgr自身不提供分布式共识机制。在自动故障转移模式下，它通常依赖**预定义的优先级**或简单的**quorum机制**（需要多数节点在线）来决定新的主节点。这要求管理员对集群拓扑有清晰的规划。
*   **配置管理**： 节点配置（如`postgresql.conf`, `pg_hba.conf`）需要手动或通过外部工具（如Ansible, Puppet）同步，repmgr不负责此项。

### 2. Patroni：基于分布式共识的“自治集群”

**Patroni** 是一个完整的、模板化的高可用解决方案。它将PostgreSQL实例与一个**分布式配置存储**（如etcd, ZooKeeper, Consul）紧密集成，实现了高度自动化的集群管理。

*   **架构**： 包含Patroni守护进程（管理PostgreSQL生命周期）和强依赖的分布式键值存储（存储集群状态、配置和领导锁）。
*   **共识与领导者选举**： 其核心优势在于利用**分布式一致性协议**（如Raft）进行领导者选举和状态同步。这确保了在脑裂（Split-Brain）场景下，最多只有一个节点能成为主库，从根本上避免了数据冲突的风险。
*   **配置管理**： Patroni可以将集群的动态配置（如`postgresql.conf`中的部分参数）存储在分布式配置中心，并动态应用到各个节点，实现了配置的集中化管理。

## 二、 深入对比：特性、场景与抉择

| 特性维度 | **repmgr** | **Patroni** |
| :--- | :--- | :--- |
| **核心哲学** | 复制管理工具，专注故障转移 | 完整的、基于共识的HA框架 |
| **共识机制** | 无内置，依赖优先级或简单quorum | 强依赖外部分布式存储（etcd等） |
| **配置管理** | 需外部工具同步 | 支持动态配置，集中化管理 |
| **复杂度** | **相对简单**，组件少，易于理解 | **相对复杂**，需维护额外分布式存储集群 |
| **脑裂防护** | 较弱，依赖外部脚本或人工干预 | **强**，由分布式锁保证 |
| **适用场景** | 中小规模集群，运维团队熟悉PG复制 | 中大规模、对自动化和可靠性要求高的生产环境 |
| **扩展性** | 较好，与外部编排工具（如K8s）集成需定制 | 优秀，原生支持云原生环境，与K8s Operator结合紧密 |

**如何选择？**
*   **选择 repmgr，如果你**： 集群规模不大（如3-5节点），希望方案轻量、直接，团队对PostgreSQL流复制有深入理解，且能接受在极端情况下可能需要手动介入。
*   **选择 Patroni，如果你**： 追求最高级别的自动化与可靠性，集群规模较大或计划扩展，环境复杂（如混合云），且愿意投入精力维护一套分布式配置存储。

## 三、 落地实践：以Patroni为例的部署要点

下面我们以Patroni + etcd的经典组合为例，展示关键配置和注意事项。

### 1. 环境准备与etcd集群部署

首先部署一个3节点的etcd集群，这是Patroni的“大脑”。

```bash
# 示例：在一个节点上启动etcd (实际应为多节点集群)
etcd --name infra0 \
  --data-dir /var/lib/etcd \
  --listen-client-urls http://10.0.1.10:2379 \
  --advertise-client-urls http://10.0.1.10:2379 \
  --listen-peer-urls http://10.0.1.10:2380 \
  --initial-advertise-peer-urls http://10.0.1.10:2380 \
  --initial-cluster-token etcd-cluster-1 \
  --initial-cluster infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380 \
  --initial-cluster-state new
```

**注意点**： etcd集群节点数应为奇数（3，5，7），以确保选举能正常进行。生产环境需确保etcd节点分布在不同的故障域。

### 2. Patroni节点配置

每个PostgreSQL节点都需要安装并配置Patroni。配置文件（如`/etc/patroni.yml`）是关键。

```yaml
scope: pg-cluster-prod # 集群逻辑名称
namespace: /service/   # 在etcd中的路径前缀

restapi:
  listen: 0.0.0.0:8008 # Patroni管理API地址
  connect_address: ${NODE_IP}:8008

etcd:
  hosts:
    - 10.0.1.10:2379
    - 10.0.1.11:2379
    - 10.0.1.12:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576 # 1MB，允许从库落后的最大字节数
    postgresql:
      use_pg_rewind: true # 启用pg_rewind，便于旧主库重新加入
      parameters:
        wal_level: replica
        hot_standby: "on"
        max_connections: 100
        shared_buffers: 1GB
        synchronous_commit: "on" # 开启同步提交以提高数据安全性
        synchronous_standby_names: "*" # 允许所有同步备库

  initdb:
    - encoding: UTF8
    - locale: en_US.UTF-8

  pg_hba:
    - host replication replicator 10.0.0.0/8 md5
    - host all all 10.0.0.0/8 md5

postgresql:
  listen: 0.0.0.0:5432
  connect_address: ${NODE_IP}:5432
  data_dir: /var/lib/postgresql/14/main
  pgpass: /tmp/pgpass
  authentication:
    replication:
      username: replicator
      password: securepassword
    superuser:
      username: postgres
      password: adminpassword
  parameters:
    # 可以覆盖bootstrap中的参数

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: true
```

**关键配置解析与注意点**：
*   `scope` 和 `namespace`： 定义了集群在etcd中的唯一标识，必须全局唯一。
*   `synchronous_commit` 与 `synchronous_standby_names`： 配置同步复制。`synchronous_standby_names: “*”` 表示所有备库都是潜在的同步备库，Patroni会确保至少有一个备库确认后才提交。**注意**：这会影响主库的写入延迟，需根据业务容忍度调整。
*   `use_pg_rewind: true`： **强烈建议开启**。在主备角色切换后，旧主库可能含有新主库没有的WAL日志，`pg_rewind`可以高效地将其回退到一致点，而不是全量重建。
*   `maximum_lag_on_failover`： 控制故障转移时，备库允许落后的程度。设置过小可能导致无合格备库可提升，过大则可能丢失较多数据。
*   **密码管理**： 示例中密码明文存储，**生产环境必须使用**`pgpass`文件或环境变量等更安全的方式，并确保文件权限为`600`。

### 3. 启动与监控

```bash
# 启动Patroni（以后台服务方式运行，如systemd）
patroni /etc/patroni.yml

# 通过Patroni API查看集群状态
curl http://localhost:8008/cluster | jq .
```

监控输出应清晰显示当前领导者、备库列表、复制延迟等信息。

## 四、 故障转移场景与陷阱规避

### 1. 优雅的主库故障
*   **过程**： Patroni通过etcd租约（TTL）监控主库健康。主库Patroni进程停止或与etcd失联后，租约过期，领导锁释放。剩余节点中，复制延迟最小的健康备库将参与选举并尝试获取锁，成功后提升自己为主库。
*   **陷阱**： **网络分区（Network Partition）**。如果主库只是与etcd网络断开，但自身仍在运行，此时备库被提升，就会形成“脑裂”。Patroni通过**强制关闭旧主库的PostgreSQL进程**来防止此情况（前提是它能连接到旧主库的REST API）。确保网络拓扑和防火墙规则允许Patroni节点间通过API端口（默认8008）通信至关重要。

### 2. 备库提升后的数据一致性
*   **过程**： 新主库提升后，会更新etcd中的拓扑信息。其他存活的备库会根据新拓扑，自动重新指向新的主库进行复制。
*   **陷阱**： **异步复制下的数据丢失**。如果采用异步复制（`synchronous_standby_names`为空），故障转移时，尚未复制到备库的已提交事务将永久丢失。**建议**：对数据一致性要求高的业务，至少配置一个**同步备库**。这需要在性能和数据安全间取得平衡。

### 3. 旧主库的重新加入
*   **理想情况**： `pg_rewind`启用且工作正常，旧主库被`rewind`后，作为新备库重新加入集群。
*   **陷阱**： 如果`pg_rewind`失败（例如表空间路径不一致），Patroni会尝试从新主库做基准备份来重建该节点。这会导致该节点长时间不可用，并消耗大量网络和IO资源。**务必在测试环境充分验证`pg_rewind`的成功率**。

## 五、 总结与最佳实践

无论选择Patroni还是repmgr，以下实践原则普遍适用：

1.  **测试，测试，再测试**： 在准生产环境模拟各种故障场景（杀进程、断网、断电、磁盘满），观察集群行为是否符合预期。故障转移演练应常态化。
2.  **监控与告警全覆盖**： 监控不仅包括数据库指标（连接数、延迟、WAL大小），更要涵盖HA组件本身（Patroni/etcd进程状态、API可达性、etcd集群健康度、节点间的网络延迟）。
3.  **文档与流程**： 清晰记录集群架构、IP、恢复步骤。即使全自动故障转移，也要有手动干预的应急预案。
4.  **从简单开始**： 如果刚开始接触PostgreSQL HA，可以从repmgr入手理解基础概念。当集群规模和可靠性要求增长时，再平滑迁移到Patroni。
5.  **备份是最后的防线**： 高可用不等于数据备份。必须建立独立于HA集群的、定期的物理备份和逻辑备份策略，并定期验证恢复流程。

高可用架构的构建是一个系统工程，技术选型是起点，严谨的运维管理和持续的验证改进才是保障其长期稳定运行的基石。希望本文的对比与分析，能帮助你在PostgreSQL高可用的道路上做出更明智的决策，并平稳落地。