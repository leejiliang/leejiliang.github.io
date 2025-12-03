---
layout: post
title: 逻辑复制与CDC：解锁PostgreSQL数据流动的进阶姿势
categories: [PG]
description: 本文深入探讨PostgreSQL的逻辑复制与变更数据捕获（CDC）技术，涵盖原生logical replication与扩展pglogical的核心原理、配置方法、典型应用场景及与物理复制的对比，帮助读者构建更灵活、高效的数据同步与集成方案。
keywords: PostgreSQL, 逻辑复制, CDC, pglogical, 数据同步
comments: false
---

# 逻辑复制与CDC：更灵活的数据复制方式

在数据驱动的时代，如何高效、可靠地在不同数据库实例、甚至不同数据库系统之间同步数据，是许多架构师和开发者面临的挑战。PostgreSQL作为一款功能强大的开源关系型数据库，提供了多种数据复制方案。其中，**逻辑复制（Logical Replication）** 与 **变更数据捕获（Change Data Capture, CDC）** 凭借其基于数据变更逻辑的精细控制能力，正成为构建现代化数据架构的关键技术。本文将聚焦于PostgreSQL的原生逻辑复制与强大的第三方扩展`pglogical`，为你揭示其原理、用法与广阔的应用场景。

## 一、 从物理复制到逻辑复制：一次质的飞跃

在深入逻辑复制之前，有必要先理解其前身——物理复制（Physical Replication，通常指流复制）。

*   **物理复制（流复制）**：其核心是**基于WAL（Write-Ahead Logging）记录的字节级复制**。主库（Primary）将产生的所有WAL记录发送给一个或多个备库（Standby），备库通过重放（Replay）这些WAL记录，在**数据块级别**上实现与主库的完全一致。这相当于对整个数据库集群做了一个“二进制镜像”。
    *   **优点**：高可靠性，支持高可用（HA）和故障转移，数据一致性最强。
    *   **缺点**：灵活性差。通常只能进行整个数据库集群的全量复制，且主备库PostgreSQL版本必须高度兼容。无法选择特定表、特定操作进行复制，也无法向不同版本的PostgreSQL或异构数据库（如Kafka、数据仓库）复制。

*   **逻辑复制**：其核心是**基于数据逻辑变更的复制**。它解析WAL日志，将其中的事务操作（INSERT, UPDATE, DELETE）解码成一种易于理解的中间格式（逻辑解码），然后通过网络传输给订阅者（Subscriber），订阅者再将这些逻辑操作重新应用。
    *   **优点**：
        1.  **表级粒度**：可以订阅一个或多个特定的表，而不是整个数据库。
        2.  **选择性复制**：可以过滤特定的操作（如只复制INSERT）或基于条件过滤行。
        3.  **跨版本与异构**：由于传输的是逻辑SQL操作，因此对PostgreSQL主次版本的要求更宽松，也为向其他系统（实现相同协议）复制数据提供了可能。
        4.  **双向与多主复制**：在特定工具（如`pglogical`）支持下，可以构建更复杂的拓扑。
        5.  **零负载快照**：可以为初始数据同步提供不阻塞主库的初始快照。

简而言之，物理复制是为了**高可用与灾难恢复**，而逻辑复制是为了**数据流动、集成与共享**。

## 二、 PostgreSQL原生逻辑复制详解

从PostgreSQL 10开始，逻辑复制作为核心功能被引入，其架构主要包含两个角色：发布者（Publisher）和订阅者（Subscriber）。

### 核心概念与配置步骤

1.  **准备工作**：
    *   确保`wal_level`参数设置为`logical`（需要重启数据库）。
    *   配置`pg_hba.conf`，允许订阅者连接发布者。
    *   发布者和订阅者上需要被复制的表必须具有相同的名称和模式（列名、数据类型），并且订阅者端的表需要有主键或唯一约束。

2.  **在发布者（主库）上创建发布（Publication）**：
    ```sql
    -- 连接到发布者数据库
    \c sourcedb

    -- 创建一个发布，包含表 `orders` 和 `order_items`
    CREATE PUBLICATION my_publication FOR TABLE orders, order_items;

    -- 也可以发布所有表
    -- CREATE PUBLICATION all_tables FOR ALL TABLES;

    -- 发布特定操作，例如只发布INSERT和UPDATE
    -- CREATE PUBLICATION insert_update_pub FOR TABLE my_table WITH (publish = 'insert, update');
    ```

3.  **在订阅者（从库）上创建订阅（Subscription）**：
    ```sql
    -- 连接到订阅者数据库（目标库）
    \c targetdb

    -- 创建订阅，连接到发布者
    CREATE SUBSCRIPTION my_subscription
    CONNECTION 'host=192.168.1.100 port=5432 dbname=sourcedb user=repuser password=secret'
    PUBLICATION my_publication;

    -- 创建后，订阅会启动一个后台worker进程，开始同步数据并持续复制变更。
    ```

4.  **管理操作**：
    ```sql
    -- 查看发布与订阅状态
    SELECT * FROM pg_publication;
    SELECT * FROM pg_subscription;

    -- 在发布者端增加或移除表
    ALTER PUBLICATION my_publication ADD TABLE new_table;
    ALTER PUBLICATION my_publication DROP TABLE old_table;

    -- 暂停/恢复/删除订阅
    ALTER SUBSCRIPTION my_subscription DISABLE;
    ALTER SUBSCRIPTION my_subscription ENABLE;
    DROP SUBSCRIPTION my_subscription;
    ```

### 优势与限制

**优势**：内置于核心，无需额外扩展，配置相对简单，是进行跨版本升级、报表库分离、数据汇聚等场景的官方标准方案。

**限制**：
*   初始数据同步（Initial Data Sync）可能对大表造成负载。
*   不支持DDL复制（表结构变更需要手动在订阅端执行）。
*   原生版本对复制冲突的处理和复杂拓扑（如双向复制）支持较弱。
*   必须要有主键或唯一约束。

## 三、 pglogical：更强大的逻辑复制引擎

`pglogical`是由2ndQuadrant开发（现已成为PostgreSQL社区生态的一部分）的一个逻辑复制扩展，它比原生逻辑复制出现得更早，功能也更强大，甚至影响了原生逻辑复制的设计。对于PostgreSQL 9.4到9.6的用户，`pglogical`是使用逻辑复制的唯一选择。

### 核心特性

1.  **无主键表支持**：通过配置`key`列，可以复制没有主键的表。
2.  **更精细的过滤**：
    *   **行过滤**：基于`WHERE`条件复制部分数据。
    *   **列过滤**：只复制指定的列。
    *   **操作过滤**：更灵活地选择复制INSERT/UPDATE/DELETE。
3.  **并行应用**：可以并行应用更改，提高复制吞吐量。
4.  **更好的冲突检测与解决**：提供更多冲突解决策略。
5.  **对复杂拓扑的强力支持**：原生支持**双向复制（Bidirectional Replication）** 和**级联复制**，是实现多主（Multi-Master）架构的基础。
6.  **升级与迁移**：提供了强大的`pglogical_migration`工具，简化跨大版本升级或迁移过程。

### 基本使用示例

1.  **安装扩展**（在发布者和订阅者数据库都需要执行）：
    ```sql
    CREATE EXTENSION IF NOT EXISTS pglogical;
    ```

2.  **在发布者端创建节点（Node）和复制集（Replication Set）**：
    ```sql
    -- 创建提供者节点
    SELECT pglogical.create_node(
        node_name := 'provider_node',
        dsn := 'host=provider_host port=5432 dbname=db user=repuser'
    );

    -- 将表添加到默认复制集‘default’
    SELECT pglogical.replication_set_add_table(
        set_name := 'default',
        relation := 'public.orders',
        synchronize_data := true -- 同步现有数据
    );

    -- 创建自定义复制集并添加带过滤的表
    SELECT pglogical.create_replication_set('important_orders');
    SELECT pglogical.replication_set_add_table(
        'important_orders',
        'public.orders',
        true,
        columns := ARRAY['id', 'customer_id', 'amount', 'status'], -- 列过滤
        row_filter := 'status = “shipped” AND amount > 1000' -- 行过滤
    );
    ```

3.  **在订阅者端创建节点并订阅**：
    ```sql
    -- 创建订阅者节点
    SELECT pglogical.create_node(
        node_name := 'subscriber_node',
        dsn := 'host=subscriber_host port=5432 dbname=db user=repuser'
    );

    -- 创建订阅
    SELECT pglogical.create_subscription(
        subscription_name := 'my_subscription',
        provider_dsn := 'host=provider_host port=5432 dbname=db user=repuser',
        replication_sets := ARRAY['default', 'important_orders'], -- 订阅多个复制集
        synchronize_data := true
    );
    ```

## 四、 典型应用场景

1.  **报表与分析库分离**：
    *   **场景**：主库（OLTP）承受高并发事务，复杂的分析查询会严重影响其性能。
    *   **方案**：使用逻辑复制，将核心业务表实时同步到另一个独立的报表数据库。在报表库上可以创建索引、物化视图，运行重型分析查询，互不干扰。

2.  **数据汇聚与数据仓库ETL**：
    *   **场景**：多个业务系统的数据需要集中到数据仓库（如ClickHouse, Redshift）进行统一分析。
    *   **方案**：每个业务系统的PostgreSQL作为发布者，通过逻辑复制/CDC将变更实时推送到一个Kafka集群。再由ETL工具（如Debezium）消费Kafka消息，转换后加载到数据仓库。这是现代CDC管道的典型架构。

3.  **零停机升级与迁移**：
    *   **场景**：需要将PostgreSQL从9.6版本升级到12版本，或迁移到新硬件，要求业务几乎不中断。
    *   **方案**：在新环境部署高版本PostgreSQL作为订阅者，通过`pglogical`或原生逻辑复制从旧库同步数据。数据追平后，将应用连接切换到新库，实现平滑升级。

4.  **多活（Multi-Active）或分区域部署**：
    *   **场景**：业务全球化，需要在不同地区（如美东、欧洲）部署可写的数据库实例，降低延迟并提高可用性。
    *   **方案**：使用`pglogical`构建**双向复制**。每个地区的实例都可以写入，并同步到其他实例。**这是高阶且复杂的场景**，必须精心设计冲突避免与解决策略（例如，按用户ID或地区ID进行数据分片，确保同一行数据只在一个地区被修改）。

5.  **微服务数据共享**：
    *   **场景**：在微服务架构中，服务A需要服务B所拥有数据的一个只读副本，但不能直接连接服务B的数据库。
    *   **方案**：服务B的数据库将相关表通过逻辑复制发布到服务A的专属数据库，实现数据的解耦与共享。

## 五、 总结与选型建议

| 特性 | PostgreSQL 原生逻辑复制 | pglogical |
| :--- | :--- | :--- |
| **支持版本** | PostgreSQL 10+ | PostgreSQL 9.4+ |
| **核心优势** | 官方内置，简单稳定，维护有保障 | 功能强大，过滤灵活，支持双向/多主 |
| **拓扑结构** | 一主多从 | 一主多从、多主、级联等 |
| **过滤能力** | 弱（仅表级、操作级） | 强（行、列、操作级） |
| **无主键表** | 不支持 | 支持 |
| **DDL复制** | 不支持 | 不支持 |

**选型建议**：

*   **对于PostgreSQL 10+用户**：如果需求是简单的单向表级数据同步（如报表分离、数据汇聚），**优先使用原生逻辑复制**，它更简洁且是未来发展的方向。
*   **对于需要复杂功能或旧版本用户**：如果需要行/列过滤、双向复制，或者正在使用PostgreSQL 9.4-9.6，那么**`pglogical`是更佳选择**。
*   **对于构建CDC数据管道**：无论是原生还是`pglogical`，都可以作为Debezium等CDC工具的可靠数据源，将变更流式传输到Kafka，进而集成到更广阔的数据生态中。

逻辑复制与CDC技术为PostgreSQL打开了数据实时流动的大门。理解并善用这些工具，能够帮助你构建出更灵活、健壮和可扩展的数据架构，从容应对大数据时代的挑战。