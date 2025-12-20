---
layout: post
title: 平滑迁移指南：从MySQL/Oracle到PostgreSQL的数据类型映射、编码与零停机策略
categories: [PG]
description: 本文详细探讨如何将数据库从MySQL或Oracle平滑迁移至PostgreSQL，涵盖核心的数据类型映射、字符编码处理，并提供一套切实可行的方案以最小化甚至消除迁移过程中的业务停机时间。
keywords: PostgreSQL, 数据库迁移, 数据类型映射, 字符编码, 最小化停机
comments: false
---

# 平滑迁移指南：从MySQL/Oracle到PostgreSQL的数据类型映射、编码与零停机策略

随着PostgreSQL在功能、性能和生态上的飞速发展，越来越多的企业和开发者开始考虑将现有业务从MySQL或Oracle迁移至PostgreSQL。迁移过程并非简单的数据搬运，它涉及数据类型的兼容性转换、字符编码的统一以及最关键的——如何保证业务连续性，最小化停机时间。本文将围绕这三个核心挑战，提供一套实战性迁移方案。

## 一、 迁移前的核心考量：数据类型映射

数据类型是数据库的基石，不兼容的类型映射是迁移失败和数据丢失的主要原因。下面我们分别梳理从MySQL和Oracle迁移到PostgreSQL时的关键映射关系。

### 1.1 从 MySQL 到 PostgreSQL

MySQL的数据类型相对灵活，但迁移到强类型的PostgreSQL时需要特别注意。

| MySQL 数据类型 | PostgreSQL 数据类型 | 注意事项与转换策略 |
| :--- | :--- | :--- |
| `INT`, `INTEGER` | `INTEGER` | 直接映射。 |
| `BIGINT` | `BIGINT` | 直接映射。 |
| `VARCHAR(n)` | `VARCHAR(n)` | 注意：PostgreSQL的`n`是字符数，MySQL是字节数（对于多字节编码有差异）。建议统一使用UTF-8编码并适当增加长度。 |
| `TEXT`, `LONGTEXT` | `TEXT` | PostgreSQL的`TEXT`类型无长度限制，可容纳`LONGTEXT`。 |
| `DATETIME` | `TIMESTAMP` | 直接映射。PostgreSQL的`TIMESTAMP`可存储带时区或不带时区的时间。 |
| `TIMESTAMP` | `TIMESTAMPTZ` | 建议映射到带时区的`TIMESTAMPTZ`，以保留MySQL `TIMESTAMP`的时区转换行为。 |
| `ENUM('val1', 'val2')` | `CREATE TYPE ... AS ENUM` | MySQL的ENUM是伪类型，PostgreSQL需要先创建枚举类型。 |
| `BOOLEAN` (TINYINT(1)) | `BOOLEAN` | MySQL常用`TINYINT(1)`模拟布尔值，迁移时需将值0/1转换为`FALSE`/`TRUE`。 |

**代码示例：ENUM类型迁移**
```sql
-- 在PostgreSQL中创建对应的枚举类型
CREATE TYPE user_status AS ENUM ('active', 'inactive', 'pending');

-- 创建表时使用该类型
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    status user_status NOT NULL DEFAULT 'pending'
);

-- 迁移数据时，MySQL的ENUM字符串值可以直接插入PostgreSQL的枚举类型
-- INSERT INTO users (name, status) SELECT name, status FROM mysql_users;
```

### 1.2 从 Oracle 到 PostgreSQL

Oracle的数据类型体系庞大且独特，迁移时需要更多转换工作。

| Oracle 数据类型 | PostgreSQL 数据类型 | 注意事项与转换策略 |
| :--- | :--- | :--- |
| `NUMBER(p, s)` | `NUMERIC(p, s)` | 精度和小数位映射。`NUMBER`（无精度）可映射为`NUMERIC`或`DECIMAL`。 |
| `VARCHAR2(n)` | `VARCHAR(n)` | 直接映射。同样需注意字符与字节的差异。 |
| `DATE` | `TIMESTAMP` | Oracle的`DATE`包含日期和时间，应映射到PostgreSQL的`TIMESTAMP`。 |
| `TIMESTAMP` | `TIMESTAMPTZ` | 建议使用带时区类型以保持信息完整。 |
| `CLOB` | `TEXT` | 直接映射。 |
| `BLOB` | `BYTEA` | PostgreSQL使用`BYTEA`存储二进制数据。 |
| `RAW(n)` | `BYTEA` | 二进制数据映射。 |
| `ROWID` | 无直接对应 | 通常需要删除或使用自定义序列/主键替代。 |

**代码示例：日期类型迁移**
```sql
-- Oracle的DATE包含时间，在PostgreSQL中应使用TIMESTAMP
-- 假设源表为 oracle_orders(order_date DATE)
CREATE TABLE pg_orders (
    order_id SERIAL PRIMARY KEY,
    order_date TIMESTAMP NOT NULL
);

-- 使用迁移工具或ETL过程时，确保日期时间格式正确转换
-- INSERT INTO pg_orders (order_date) SELECT order_date FROM oracle_orders;
```

## 二、 字符编码：避免乱码的基石

字符编码不一致是迁移后出现乱码的罪魁祸首。PostgreSQL默认使用`UTF-8`编码，它支持全球所有字符，是最佳选择。

**迁移策略：**
1.  **统一为UTF-8**：在迁移前，确保源数据库（MySQL/Oracle）的库、表、连接客户端都使用或能正确输出`UTF-8`编码。对于MySQL，检查`character_set_database`和`character_set_server`；对于Oracle，检查`NLS_CHARACTERSET`。
2.  **在PostgreSQL中创建UTF-8数据库**：
    ```sql
    CREATE DATABASE myapp WITH ENCODING 'UTF8' LC_COLLATE='en_US.UTF-8' LC_CTYPE='en_US.UTF-8' TEMPLATE=template0;
    ```
3.  **迁移过程中指定编码**：在使用`pg_dump`/`pg_restore`或ETL工具时，明确指定连接和文件的编码为`UTF-8`。

## 三、 最小化停机窗口的实战方案

业务停机时间是迁移成功与否的关键指标。我们的目标是实现“零停机”或“秒级停机”迁移。

### 3.1 迁移流程总览
一个完整的平滑迁移流程包含以下阶段：
1.  **评估与准备**：分析源库结构、数据量、业务峰值。
2.  **结构迁移与转换**：在目标PostgreSQL创建转换后的表结构。
3.  **全量数据同步**：首次将全部历史数据迁移至PostgreSQL。
4.  **增量数据同步**（关键）：在业务运行期间，持续将源库的变更同步到PostgreSQL。
5.  **数据验证与业务测试**：对比数据一致性，在PG上运行测试业务。
6.  **流量切换**：将应用连接从源库切换到PostgreSQL。
7.  **回滚预案**：准备快速切回源库的方案。

### 3.2 核心：基于逻辑复制的增量同步

这是实现最小化停机的核心技术。我们可以在迁移后期，建立一个从源库到PostgreSQL的“准实时”数据同步通道。

**对于MySQL源库：**
可以使用 **Debezium** 捕获MySQL的binlog变更，并将其转换为PostgreSQL可消费的格式，再通过自定义应用或**Kafka Connect JDBC Sink**写入PostgreSQL。

**对于Oracle源库：**
可以使用 **Oracle GoldenGate**、**Debezium with Oracle LogMiner** 或 **DSG (Data Synchronization Guard)** 等商业或开源工具，实时捕获Redo Log的变更并应用到PostgreSQL。

**简化流程示例（概念性）：**
```
MySQL/Oracle (主库，业务运行中)
        |
        | 实时捕获变更 (binlog/redo log)
        v
   [变更数据捕获CDC工具，如 Debezium]
        |
        | 流转变更事件 (e.g., Apache Kafka)
        v
[消费者应用 / Kafka Connect JDBC Sink]
        |
        | 应用变更到目标库
        v
   PostgreSQL (备库，持续同步)
```

### 3.3 切换与回滚

当增量同步延迟极低（如秒级）且数据验证通过后，即可执行切换。

1.  **短暂停写**：在业务低峰期，暂停对源数据库的所有写操作（可通过应用配置或网关实现）。
2.  **追平增量**：等待CDC工具将最后一批变更完全同步到PostgreSQL。
3.  **切换连接**：将应用程序的数据库连接字符串指向PostgreSQL集群，并恢复写操作。
4.  **监控与验证**：密切监控PostgreSQL的性能和业务日志。

**回滚预案**：切换后若出现严重问题，需立即将连接切回源库。因此，在切换前务必**确保源库在停写期间的数据被完整保留**，或者同样建立从PostgreSQL回源库的同步通道以备回滚。

## 四、 工具推荐与总结

*   **结构迁移与数据全量同步**：
    *   **pgloader**: 功能强大，支持从MySQL/Oracle直接迁移，能进行简单的数据类型转换。
    *   **ora2pg**: 专用于Oracle到PostgreSQL迁移，提供详细的评估报告和转换脚本。
    *   商业ETL工具：如Informatica, Talend等。
*   **增量数据同步（CDC）**：
    *   **Debezium** + **Kafka**: 开源流式CDC方案，社区活跃，适用于MySQL和Oracle（后者需配置）。
    *   **AWS DMS / Azure DMS**: 云服务商提供的数据库迁移服务，简化流程。
    *   **Oracle GoldenGate**: 成熟的商业CDC工具，对Oracle支持最好。

**总结**
从MySQL或Oracle迁移到PostgreSQL是一项系统工程，成功的关键在于：
1.  **精细的数据类型映射**，确保数据语义和精度不丢失。
2.  **坚定的UTF-8编码策略**，从根本上杜绝乱码。
3.  **采用“全量+增量”的同步模式**，并利用CDC技术实现业务运行期间的持续同步，这是将停机窗口从数小时压缩至分钟甚至秒级的核心。

充分的预迁移测试、完整的数据验证以及可靠的回滚预案，是最终平稳上线的最后保障。希望本文的实战指南能为你的PostgreSQL迁移之旅保驾护航。