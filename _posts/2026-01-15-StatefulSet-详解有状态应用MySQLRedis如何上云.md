---
layout: post
title: StatefulSet 详解：有状态应用（MySQL/Redis）如何优雅上云
categories: [k8s]
description: 本文深入剖析Kubernetes StatefulSet控制器，详细讲解其核心概念、工作原理以及与Deployment的关键区别。通过部署MySQL主从和Redis哨兵集群的完整实战案例，手把手教你如何将复杂的有状态应用安全、可靠地迁移到Kubernetes云原生环境。
keywords: Kubernetes, StatefulSet, 有状态应用, MySQL, Redis
comments: false
---

# StatefulSet 详解：有状态应用（MySQL/Redis）如何优雅上云

在 Kubernetes 的世界里，`Deployment` 和 `ReplicaSet` 是管理无状态应用（Stateless Application）的利器。它们可以轻松地进行滚动更新、扩缩容，并且 Pod 实例之间完全平等，可以互相替换。然而，当我们需要部署数据库（如 MySQL、PostgreSQL）、消息队列（如 Kafka）、缓存系统（如 Redis Cluster）等**有状态应用（Stateful Application）**时，`Deployment` 就显得力不从心了。

有状态应用的核心特征包括：
1.  **稳定的网络标识**：每个实例需要一个固定且唯一的名称，即使重启或重新调度，客户端也需要能通过这个固定标识访问到它。
2.  **稳定的持久化存储**：每个实例的数据需要独立且持久地保存，并与实例的生命周期绑定，即使 Pod 被重建，数据也不能丢失或错乱。
3.  **有序的部署与扩缩容**：实例的启动、升级、删除往往需要遵循严格的顺序（例如，数据库主节点要先于从节点启动）。

为了满足这些需求，Kubernetes 引入了 **StatefulSet** 这个强大的工作负载控制器。

## StatefulSet 核心概念解析

### 与 Deployment 的关键区别

| 特性 | Deployment (无状态) | StatefulSet (有状态) |
| :--- | :--- | :--- |
| **Pod 标识** | 随机哈希，不固定 (e.g., `webapp-5d89b4f6d-abc1x`) | 固定有序索引 (e.g., `mysql-0`, `mysql-1`) |
| **网络标识** | 共享同一个 Service，负载均衡到任意 Pod | 每个 Pod 有独立的 DNS 记录：`<pod-name>.<svc-name>.<namespace>.svc.cluster.local` |
| **存储** | 所有 Pod 共享 PVC（如果配置），或使用空卷 | 每个 Pod 拥有独立的、稳定的 PVC，通过 `volumeClaimTemplates` 动态创建 |
| **创建/删除顺序** | 并行，无顺序 | **顺序**：从索引 0 到 N-1 创建，逆序删除 |
| **扩缩容顺序** | 无序 | **扩容**：顺序创建新索引 Pod；**缩容**：逆序删除高索引 Pod |
| **更新策略** | RollingUpdate / Recreate | RollingUpdate (分区更新) / OnDelete |

### 核心组件协同工作

一个完整的 StatefulSet 部署通常涉及以下几个部分：
1.  **StatefulSet 控制器**：定义 Pod 模板、副本数、更新策略等。
2.  **Headless Service**：一个 `clusterIP: None` 的 Service，用于为每个 Pod 提供唯一的 DNS 记录，是实现稳定网络标识的关键。
3.  **VolumeClaimTemplate (存储卷申请模板)**：定义每个 Pod 需要申请的 PersistentVolumeClaim (PVC) 的规格。StatefulSet 控制器会按 Pod 索引自动创建形如 `data-mysql-0`， `data-mysql-1` 的 PVC。
4.  **PersistentVolume (PV)**：由集群管理员提供或通过 StorageClass 动态供给的持久化存储资源。

## 实战：在 Kubernetes 中部署 MySQL 主从集群

下面我们通过一个经典的 MySQL 主从复制（Master-Slave）案例来演示 StatefulSet 的强大功能。

### 第一步：创建 Headless Service 和 ConfigMap

首先，创建一个用于服务发现的 Headless Service。

```yaml
# mysql-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  ports:
  - name: mysql
    port: 3306
  clusterIP: None # 这是关键，定义为 Headless Service
  selector:
    app: mysql
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
data:
  master.cnf: |
    [mysqld]
    log-bin=mysql-bin
    server-id=1
  slave.cnf: |
    [mysqld]
    super-read-only=1
    server-id=2
```

### 第二步：创建 StatefulSet

这是最核心的部分。我们定义了一个 3 副本的 StatefulSet，并为每个 Pod 配置了独立的存储和配置。

```yaml
# mysql-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: "mysql" # 必须指向前面创建的 Headless Service
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      initContainers:
      - name: init-mysql
        image: mysql:5.7
        command:
        - bash
        - "-c"
        - |
          # 根据 Pod 序号，拷贝不同的配置文件
          set -ex
          [[ `hostname` =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          if [ $ordinal -eq 0 ]; then
            cp /mnt/config-map/master.cnf /mnt/conf.d/
          else
            cp /mnt/config-map/slave.cnf /mnt/conf.d/
          fi
        volumeMounts:
        - name: conf
          mountPath: /mnt/conf.d
        - name: config-map
          mountPath: /mnt/config-map
      containers:
      - name: mysql
        image: mysql:5.7
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret # 假设已创建包含密码的Secret
              key: password
        ports:
        - name: mysql
          containerPort: 3306
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql # 可选的子路径
        - name: conf
          mountPath: /etc/mysql/conf.d
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
      volumes:
      - name: config-map
        configMap:
          name: mysql-config
  volumeClaimTemplates: # 核心！为每个 Pod 生成独立的 PVC
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "standard" # 替换为你的 StorageClass
      resources:
        requests:
          storage: 10Gi
```

**关键点解释：**
- `serviceName: "mysql"`：将此 StatefulSet 与 `mysql` 这个 Headless Service 绑定。
- `initContainers`：初始化容器根据 Pod 的序号（`mysql-0`， `mysql-1`...）来区分主从，并拷贝对应的配置文件。
- `volumeClaimTemplates`：这是 StatefulSet 的灵魂。它会为每个 Pod（`mysql-0`， `mysql-1`， `mysql-2`）自动创建独立的 PVC（`data-mysql-0`， `data-mysql-1`， `data-mysql-2`）。Pod 与 PVC 的绑定关系是稳定的，即使 Pod `mysql-0` 被重建，它依然会挂载到 `data-mysql-0` 这个卷上。

### 第三步：验证与操作

1.  **应用配置**：
    ```bash
    kubectl apply -f mysql-service.yaml
    kubectl apply -f mysql-statefulset.yaml
    ```

2.  **观察启动顺序**：
    ```bash
    kubectl get pods -l app=mysql -w
    ```
    你会看到 Pod 严格按照 `mysql-0` -> `mysql-1` -> `mysql-2` 的顺序启动。

3.  **验证稳定的网络标识**：
    ```bash
    # 在集群内另一个Pod中执行
    nslookup mysql-0.mysql.default.svc.cluster.local
    nslookup mysql-1.mysql.default.svc.cluster.local
    ```
    每个 Pod 都有唯一且可解析的 DNS 名称。

4.  **验证稳定的存储**：
    ```bash
    kubectl get pvc -l app=mysql
    ```
    会看到三个分别绑定到三个 PV 的 PVC。

5.  **初始化主从复制**（需手动或通过额外Operator完成）：
    登录 `mysql-0`（主库）创建复制用户，查看 `SHOW MASTER STATUS`。
    登录 `mysql-1` 和 `mysql-2`（从库），执行 `CHANGE MASTER TO ...` 指向 `mysql-0.mysql`，然后 `START SLAVE`。

## 进阶：部署 Redis Sentinel 高可用集群

对于 Redis，我们可以利用 StatefulSet 来部署一个包含 Sentinel 的高可用方案。思路是：用一个 StatefulSet 部署多个 Redis 实例（其中一个为主，其余为从），用另一个 StatefulSet 部署多个 Sentinel 监控实例。

**Redis StatefulSet 示例片段：**
```yaml
# redis-statefulset.yaml (部分)
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
spec:
  serviceName: redis
  replicas: 3
  template:
    spec:
      containers:
      - name: redis
        image: redis:6-alpine
        command: ["redis-server"]
        args: ["/etc/redis/redis.conf"]
        env:
        - name: REDIS_PORT
          value: "6379"
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        # 可以使用环境变量或配置文件，根据 POD_NAME 决定角色
  volumeClaimTemplates:
  - metadata:
      name: redis-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
```

**Sentinel StatefulSet 示例片段：**
```yaml
# redis-sentinel-statefulset.yaml (部分)
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-sentinel
spec:
  serviceName: redis-sentinel
  replicas: 3
  template:
    spec:
      containers:
      - name: sentinel
        image: redis:6-alpine
        command: ["redis-sentinel"]
        args: ["/etc/redis/sentinel.conf"]
        env:
        - name: SENTINEL_PORT
          value: "26379"
        - name: REDIS_MASTER_SERVICE_HOST
          value: "redis" # 指向 Redis 的 Service
```

通过初始化容器或启动脚本，可以自动发现 Redis Pod 的 IP 并配置主从关系和 Sentinel 的监控目标。

## StatefulSet 的更新策略与运维

StatefulSet 支持两种更新策略：
- **`RollingUpdate`（默认）**：可以配合 `partition` 字段进行“金丝雀发布”或“分阶段发布”。例如，设置 `partition: 1`，则只有索引 >=1 的 Pod（如 `mysql-1`， `mysql-2`）会被更新，`mysql-0`（主库）保持不变，实现可控更新。
- **`OnDelete`**：只有手动删除 Pod 后，才会用新模板创建，提供最大控制权。

**运维建议：**
1.  **备份**：定期备份 PV 中的数据。对于数据库，建议使用物理备份（文件快照）或逻辑备份（mysqldump）相结合。
2.  **监控**：密切监控 Pod 状态、存储使用量、数据库连接数和性能指标。
3.  **使用 Operator**：对于生产级的关键有状态应用（如 MySQL, PostgreSQL, Redis, Elasticsearch），强烈推荐使用对应的 **Kubernetes Operator**（如 `Percona Operator for MySQL`， `Redis Operator`）。Operator 封装了领域知识，能自动化处理故障转移、备份恢复、配置更新等复杂操作，大大降低了运维复杂度。

## 总结

StatefulSet 是 Kubernetes 为有状态应用量身定制的解决方案，它通过提供**稳定的网络标识**、**稳定的独立存储**和**有序的 Pod 管理**，使得在云原生环境中运行数据库、缓存等复杂系统成为可能。

将 MySQL、Redis 等有状态应用迁移上云时，StatefulSet 是基础。然而，要构建生产可用的高可用集群，通常需要结合 **Init Container**、**Sidecar Container** 进行初始化配置，并最终考虑采用成熟的 **Operator 框架**来管理全生命周期，从而实现真正的“无忧”上云。通过本文的详解和实战案例，希望你能够掌握 StatefulSet 的精髓，为你自己的有状态应用上云之旅打下坚实的基础。