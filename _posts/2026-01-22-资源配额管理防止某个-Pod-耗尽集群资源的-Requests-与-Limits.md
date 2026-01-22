---
layout: post
title: Kubernetes资源保卫战：巧用Requests与Limits，避免Pod“吃光”集群
categories: [k8s]
description: 本文深入探讨Kubernetes中资源配额管理的核心机制——Requests与Limits。通过实际场景分析，解释如何为Pod设置合理的CPU和内存资源请求与限制，并介绍ResourceQuota与LimitRange等策略，从而有效防止单个Pod耗尽集群资源，保障集群的稳定与公平。
keywords: Kubernetes, 资源管理, Requests, Limits, ResourceQuota
comments: false
---

# Kubernetes资源保卫战：巧用Requests与Limits，避免Pod“吃光”集群

在Kubernetes集群中，多个工作负载共享计算资源是其核心优势之一。然而，这种共享也带来了潜在风险：一个行为异常的Pod（例如，因内存泄漏或陷入死循环）可能会贪婪地消耗掉节点甚至整个集群的CPU和内存资源，导致其他关键应用因资源不足而崩溃，引发“一颗老鼠屎坏了一锅粥”的连锁反应。

为了避免这种灾难性场景，Kubernetes提供了精细化的资源管理机制，其中 **Requests（请求）** 和 **Limits（限制）** 是保护集群资源的基石。本文将深入解析这两者，并展示如何结合其他策略，构建一道坚固的资源防线。

## 一、核心概念：Requests 与 Limits 是什么？

在Kubernetes中，你可以为每个容器指定两类资源需求：`requests` 和 `limits`。它们主要针对两种资源：`cpu` 和 `memory`。

*   **`requests` (资源请求)**： 定义容器**启动时所需的最小资源量**。调度器（Scheduler）在决定将Pod放置到哪个节点时，会确保该节点上所有Pod的`requests`总和不超过节点可分配资源总量。`requests`是Pod的“资源保障”，也是调度依据。
*   **`limits` (资源限制)**： 定义容器**运行期间允许使用的资源上限**。如果容器尝试使用超过其`limits`的资源，Kubernetes会采取强制措施（如杀死容器）以防止其影响系统或其他Pod。`limits`是Pod的“资源天花板”。

简单比喻：`requests` 好比你在餐厅预订座位时承诺的最低消费，餐厅（调度器）会据此为你预留位置和基础食材；`limits` 则是你本次用餐的消费上限，超过这个额度，餐厅保安（Kubelet）会 politely but firmly 请你离开。

## 二、为什么需要两者配合？

只设置`limits`而不设`requests`：调度器不知道这个Pod需要多少资源，可能会将其调度到一个资源已经紧张的节点上，虽然运行后会被限制，但可能因节点资源碎片化导致调度效率低下或“吵闹的邻居”问题。

只设置`requests`而不设`limits`：Pod获得了资源保障，但如果其失控，仍然可能耗尽节点资源，因为缺少硬性约束。

因此，**同时设置`requests`和`limits`**，是实现资源“既保障又隔离”的最佳实践。通常建议 `requests <= limits`。

## 三、实战配置：在Pod中定义资源

下面是一个经典的Pod定义示例，展示了如何为容器设置CPU和内存的`requests`与`limits`。

```yaml
# pod-with-resources.yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
spec:
  containers:
  - name: app-container
    image: nginx:latest
    resources:
      requests:
        memory: "64Mi"   # 请求64兆字节内存
        cpu: "250m"      # 请求250毫核CPU (即0.25个CPU核心)
      limits:
        memory: "128Mi"  # 内存使用上限为128兆字节
        cpu: "500m"      # CPU使用上限为500毫核 (即0.5个CPU核心)
```

**单位说明：**
*   **CPU**： 通常以毫核（m）为单位。`1000m` = 1个CPU核心。`250m` = 0.25核心。也支持小数，如`0.25`。
*   **内存**： 可以使用Mi（Mebibyte）， Gi（Gibibyte）， 也可以使用M（Megabyte）， G（Gigabyte）。推荐使用Mi/Gi，因为1Gi = 1024Mi，更精确。

## 四、超限后果：Kubernetes如何“执法”？

当容器试图突破为其设置的`limits`时，Kubernetes会根据资源类型采取不同行动：

1.  **CPU超限**： CPU是可压缩资源。如果容器使用的CPU超过其`limits`，Kubernetes会**限制其CPU使用率，但不会杀死容器**。容器会感觉到CPU被“限速”，性能下降，但进程仍在运行。
2.  **内存超限**： 内存是不可压缩资源。如果容器使用的内存超过其`limits`，Kubernetes会将此容器判定为 **`OOMKilled` (内存溢出杀死)**。Pod状态会变为`CrashLoopBackOff`，并尝试重启。

你可以使用`kubectl describe pod <pod-name>`命令查看容器是否曾因OOM被杀死。

## 五、集群级防护：ResourceQuota 与 LimitRange

仅靠Pod级别的`requests/limits`还不够，我们需要在命名空间（Namespace）级别建立全局规则，这就是`ResourceQuota`和`LimitRange`的用武之地。

### 1. ResourceQuota：命名空间资源总配额

`ResourceQuota`用于限制一个命名空间内所有Pod消耗的资源总量，防止某个团队或项目占用整个集群的资源。

```yaml
# quota-mem-cpu.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: mem-cpu-demo
  namespace: dev-team-ns # 应用于特定命名空间
spec:
  hard:
    requests.cpu: "2"     # 该命名空间所有Pod的CPU请求总和不能超过2核心
    requests.memory: 2Gi  # 内存请求总和不能超过2GiB
    limits.cpu: "4"       # 该命名空间所有Pod的CPU限制总和不能超过4核心
    limits.memory: 4Gi    # 内存限制总和不能超过4GiB
    pods: "10"            # 最多只能创建10个Pod
```

创建此配额后，在`dev-team-ns`命名空间中创建任何Pod时，其`requests`和`limits`都必须满足配额的总量约束，否则创建请求会被拒绝。

### 2. LimitRange：命名空间内Pod/容器的默认值与极值

`LimitRange`为命名空间内的Pod或容器设置默认的`requests/limits`，以及允许的最小/最大值。这能有效防止用户创建资源需求过高或过低的Pod，并能为未指定资源的Pod自动添加默认值。

```yaml
# limits.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: limit-range-demo
  namespace: dev-team-ns
spec:
  limits:
  - type: Container # 针对容器
    default:        # 默认值（如果用户未指定limits）
      memory: 512Mi
      cpu: 500m
    defaultRequest: # 默认请求值（如果用户未指定requests）
      memory: 256Mi
      cpu: 100m
    max:            # 允许的最大值
      memory: 1Gi
      cpu: "1"
    min:            # 允许的最小值
      memory: 50Mi
      cpu: 10m
    maxLimitRequestRatio: # 限制与请求的最大比率（用于防止limits远大于requests）
      memory: 2           # 内存limits不能超过其requests的2倍
      cpu: 4              # CPU limits不能超过其requests的4倍
  - type: Pod # 也可以针对整个Pod设置资源总和限制
    max:
      memory: 2Gi
      cpu: "2"
```

## 六、最佳实践与策略

1.  **始终设置Requests和Limits**： 这是黄金法则。对于生产负载，必须明确设置。
2.  **合理估算值**： 基于应用的实际监控数据（如使用Prometheus收集的历史资源使用率P95/P99值）来设置，而非盲目猜测。`requests`可以设置为略高于平均使用量，`limits`设置为峰值使用量。
3.  **使用Quality of Service (QoS) 分类**： Kubernetes会根据Pod的`requests`和`limits`配置，自动将其分为三个QoS等级，在节点资源紧张时，驱逐优先级不同：
    *   **Guaranteed**： 所有容器的`limits`和`requests`相等且均被设置。优先级最高，最不容易被驱逐。
    *   **Burstable**： 至少有一个容器设置了`requests`或`limits`，但不满足Guaranteed条件。优先级次之。
    *   **BestEffort**： 所有容器均未设置`requests`和`limits`。优先级最低，最先被驱逐。
    为关键应用配置`Guaranteed` QoS能显著提高其可用性。
4.  **结合HPA（Horizontal Pod Autoscaler）**： HPA可以根据CPU/内存使用率等指标自动扩缩Pod副本数。HPA计算使用率时，是基于容器的`requests`值作为分母的。因此，合理设置`requests`对自动扩缩的准确性至关重要。
5.  **利用命名空间进行逻辑隔离**： 将不同项目、团队或环境（dev/staging/prod）部署到不同的命名空间，并为每个命名空间配置相应的`ResourceQuota`，实现资源的逻辑隔离和成本分摊。

## 总结

Kubernetes的资源管理模型通过`Requests`和`Limits`提供了一种强大而灵活的方式，在资源共享与隔离之间取得了平衡。从容器级别的定义，到命名空间级别的`ResourceQuota`和`LimitRange`约束，层层递进，构成了一个立体的资源防护体系。

作为集群管理员或应用开发者，深刻理解并正确运用这些机制，是确保Kubernetes集群稳定、高效运行，并防止“资源饥饿”事件发生的关键技能。开始为你所有的Pod定义合理的资源需求吧，这是迈向生产可用的坚实一步。