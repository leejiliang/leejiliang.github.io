---
layout: post
title: PostgreSQL高可用基石：深入解析流复制与热备配置
categories: [PG]
description: 本文深入探讨PostgreSQL流复制（Streaming Replication）的核心原理与配置实践，详细讲解wal_sender与wal_receiver进程的工作机制，指导如何构建主从复制架构以实现数据高可用与读负载扩展。
keywords: PostgreSQL, 流复制, 热备, 主从复制, WAL
comments: false
---

# PostgreSQL高可用基石：深入解析流复制与热备配置

在当今数据驱动的时代，数据库的高可用性和可扩展性已成为系统架构设计的核心考量。PostgreSQL作为一款功能强大的开源关系型数据库，其内置的**流复制（Streaming Replication）** 功能是构建高可用集群、实现读写分离和灾难恢复的基石。本文将深入解析流复制的工作原理，并手把手指导你完成主从环境的配置。

## 一、流复制：核心概念与工作原理

流复制是PostgreSQL实现物理复制的主要方式。其核心思想是：主库（Primary）将预写日志（Write-Ahead Logging, WAL）记录实时地、以流式传输的方式发送给一个或多个从库（Standby）。从库接收并重放这些WAL记录，从而使其数据状态与主库保持同步。

### 关键进程
*   **WAL Sender (`wal_sender`)**: 运行在主库上的进程，负责将WAL数据发送给从库。
*   **WAL Receiver (`wal_receiver`)**: 运行在从库上的进程，负责从主库接收WAL数据，并写入从库的本地磁盘。
*   **Startup Process**: 在从库上，负责应用已接收的WAL记录到数据文件。

### 复制模式
1.  **异步复制（Asynchronous）**: 主库提交事务后，无需等待从库确认接收WAL即向客户端返回成功。性能最好，但存在极小概率的数据丢失窗口。
2.  **同步复制（Synchronous）**: 主库提交事务时，必须等待至少一个指定的从库确认已接收并写入WAL后，才向客户端返回成功。保证了数据的强一致性，但会增加事务提交的延迟。

## 二、实战：构建主从复制环境

下面我们一步步搭建一个基于异步流复制的主从环境。

### 环境准备
*   主库：IP `192.168.1.10`， 端口 `5432`
*   从库：IP `192.168.1.11`， 端口 `5432`
*   PostgreSQL版本：14（配置项在不同版本间可能略有差异）
*   操作系统：Linux
*   假设已在两台服务器上安装好PostgreSQL，且数据目录为 `/var/lib/pgsql/14/data`。

### 第一步：主库配置

1.  **创建复制专用用户**：
    在主库上，使用 `psql` 连接，创建一个用于复制的用户。
    ```sql
    CREATE USER replicator WITH REPLICATION LOGIN ENCRYPTED PASSWORD 'YourStrongPassword123!';
    ```

2.  **配置访问权限 (`pg_hba.conf`)**：
    编辑主库的 `pg_hba.conf` 文件，添加一行，允许从库的 `replicator` 用户连接进行复制。
    ```
    # 在文件末尾添加
    host    replication    replicator    192.168.1.11/32    scram-sha-256
    ```
    这表示允许来自 `192.168.1.11` 的用户 `replicator` 通过 `scram-sha-256` 加密方式连接进行流复制。

3.  **修改主配置文件 (`postgresql.conf`)**：
    编辑主库的 `postgresql.conf` 文件，确保以下参数已设置：
    ```properties
    listen_addresses = '*'          # 监听所有IP，或指定为'localhost,192.168.1.10'
    port = 5432                     # 监听端口
    max_wal_senders = 10            # 最大WAL发送进程数，至少大于从库数量
    wal_level = replica             # WAL级别，`replica` 或 `logical` 是流复制所必需的
    hot_standby = on                # 从库开启热备模式（允许只读查询）
    # 可选：设置同步复制（此处先配置异步，后续可调整）
    # synchronous_standby_names = 'standby_1' # 指定同步备库名
    ```

4.  **重启主库服务**：
    ```bash
    sudo systemctl restart postgresql-14
    ```

### 第二步：从库配置

1.  **停止从库服务并清空数据目录**：
    ```bash
    sudo systemctl stop postgresql-14
    rm -rf /var/lib/pgsql/14/data/*
    ```

2.  **使用 `pg_basebackup` 进行基础备份**：
    这是关键步骤，它从主库拉取一个完整的数据快照。
    ```bash
    pg_basebackup -h 192.168.1.10 -U replicator -p 5432 -D /var/lib/pgsql/14/data -Fp -Xs -P -R
    ```
    *   `-h`: 主库地址
    *   `-U`: 复制用户
    *   `-D`: 目标数据目录
    *   `-Fp`: 以普通文件格式备份
    *   `-Xs`: 在备份开始后启动流式传输WAL
    *   `-P`: 显示进度
    *   `-R`: **重要**！自动创建 `standby.signal` 文件和 `postgresql.auto.conf` 中的连接信息。

3.  **检查生成的配置**：
    `-R` 参数会做两件事：
    *   在数据目录创建 `standby.signal` 文件，标识该实例为备库。
    *   在 `postgresql.auto.conf` 中追加主库连接信息，内容类似于：
        ```properties
        primary_conninfo = 'user=replicator password=YourStrongPassword123! host=192.168.1.10 port=5432 sslmode=prefer sslcompression=0 gssencmode=prefer krbsrvname=postgres target_session_attrs=any'
        ```

4.  **启动从库服务**：
    ```bash
    sudo systemctl start postgresql-14
    ```

### 第三步：验证复制状态

1.  **在主库上查询发送进程状态**：
    ```sql
    SELECT pid, application_name, client_addr, state, sync_state, sent_lsn, write_lsn, flush_lsn, replay_lsn 
    FROM pg_stat_replication;
    ```
    如果看到一行记录，`state` 为 `streaming`，`client_addr` 为从库IP `192.168.1.11`，则说明流复制已成功建立。

2.  **在从库上查询恢复状态**：
    ```sql
    SELECT pg_is_in_recovery(); -- 返回 `t` (true) 表示处于恢复/备库模式
    ```
    ```sql
    SELECT * FROM pg_stat_wal_receiver; -- 查看接收进程状态，包括连接的主库和最新的WAL位置
    ```

3.  **测试数据同步**：
    *   在主库创建一个表并插入数据。
    ```sql
    CREATE TABLE test_repl (id serial PRIMARY KEY, data text);
    INSERT INTO test_repl (data) VALUES ('Hello from Primary!');
    ```
    *   在从库查询该表（注意：从库默认是只读的）。
    ```sql
    SELECT * FROM test_repl; -- 应该能查询到刚插入的数据
    ```

## 三、高级配置与监控

### 1. 切换为同步复制
修改主库的 `postgresql.conf`：
```properties
synchronous_standby_names = 'standby_1'
```
重启主库或执行 `pg_reload_conf()`。
然后在从库的 `postgresql.auto.conf` 中，确保 `primary_conninfo` 里包含了 `application_name` 参数，且与主库配置匹配：
```properties
primary_conninfo = '... application_name=standby_1'
```
重启从库后，在主库 `pg_stat_replication` 中查看，`sync_state` 会从 `async` 变为 `sync` 或 `quorum`。

### 2. 监控延迟
复制延迟是重要监控指标。
```sql
-- 在主库计算每个备库的延迟（字节）
SELECT 
    application_name,
    client_addr,
    pg_wal_lsn_diff(pg_current_wal_lsn(), sent_lsn) AS sent_delay,
    pg_wal_lsn_diff(pg_current_wal_lsn(), flush_lsn) AS flush_delay,
    pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS replay_delay
FROM pg_stat_replication;
```

### 3. 处理连接与网络问题
在 `primary_conninfo` 中可以配置超时和重试参数，增强网络鲁棒性：
```properties
primary_conninfo = '... keepalives_idle=60 keepalives_interval=5 keepalives_count=2'
```

## 四、应用场景与总结

*   **高可用与故障转移**：结合 Patroni、repmgr 等工具，可以实现自动故障切换。
*   **读写分离与读扩展**：将报表、分析等只读查询路由到从库，减轻主库压力。
*   **灾难恢复**：在异地部署从库，作为数据备份。
*   **零停机升级**：可以先升级从库，然后进行主从切换，实现业务无感知升级。

**总结**：PostgreSQL的流复制机制强大而灵活，是构建企业级数据库架构的基础。通过理解 `wal_sender` 和 `wal_receiver` 的协作，并正确配置相关参数，你可以轻松搭建出稳定可靠的主从复制环境，为业务系统提供坚实的数据层保障。在实际生产环境中，建议结合具体的监控告警和集群管理工具，以实现全面的高可用解决方案。