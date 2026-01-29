---
layout: post
title: 守护容器生命线：为什么K8s中的Liveness与Readiness探针不可或缺
categories: [k8s]
description: 本文深入探讨Kubernetes中Liveness和Readiness探针的核心原理、配置方法及实际应用场景。通过具体示例，阐述如何利用这两种健康检查机制提升应用的自愈能力与发布稳定性，避免服务中断与流量损失。
keywords: Kubernetes, 容器健康检查, Liveness探针, Readiness探针, 服务可用性
comments: false
---

# 守护容器生命线：为什么K8s中的Liveness与Readiness探针不可或缺

在Kubernetes中部署应用时，我们常常关注于镜像构建、资源配置和网络策略，却容易忽略一个至关重要的环节：**容器健康检查**。许多人认为只要容器能启动，服务就能运行，因此跳过了Liveness和Readiness探针的配置。然而，这种疏忽往往是生产环境不稳定的罪魁祸首。本文将深入解析这两种探针，揭示它们为何是保障应用高可用的“生命线”。

## 一、健康检查：不仅仅是“是否在运行”

在传统物理机或虚拟机时代，我们通常通过监控进程是否存在来判断服务健康。但在微服务和容器化架构中，情况变得复杂：
*   进程存在，但应用可能因死锁、内存泄漏或依赖服务不可用而无法正常工作。
*   应用正在启动，但尚未完成初始化（如加载配置、连接数据库、预热缓存），此时不应接收外部流量。

Kubernetes通过**探针（Probe）** 机制来解决这些问题。探针是kubelet定期对容器执行的诊断操作，主要分为两类：**Liveness Probe（存活探针）** 和 **Readiness Probe（就绪探针）**。

## 二、Liveness Probe：你的应用真的“活着”吗？

**核心作用**：判断容器是否处于“运行但已故障”的状态。如果探测失败，kubelet会杀死并重启容器。

**适用场景**：
1.  **应用死锁**：程序仍在运行，但已停止响应。
2.  **内部状态异常**：例如，某个关键后台线程崩溃，导致服务功能部分失效。
3.  **内存泄漏导致的僵死**：进程还在，但已无法处理新请求。

**不配置的风险**：一个“僵尸”容器会持续占用资源，并可能导致用户请求失败，而K8s却对此一无所知，不会自动恢复。

### 配置示例与详解

Liveness Probe支持三种探测方式：HTTP GET、TCP Socket和Exec命令。

**1. HTTP GET 示例（最常见）**
假设我们有一个Web应用，健康检查端点为 `/healthz`。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp-liveness
spec:
  containers:
  - name: webapp
    image: my-webapp:latest
    ports:
    - containerPort: 8080
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
        # 自定义HTTP头（可选）
        httpHeaders:
        - name: Custom-Header
          value: Awesome
      initialDelaySeconds: 30  # 容器启动后30秒开始第一次探测
      periodSeconds: 10        # 每10秒探测一次
      timeoutSeconds: 5        # 探测超时时间5秒
      successThreshold: 1      # 连续1次成功视为成功
      failureThreshold: 3      # 连续3次失败视为失败，触发重启
```

*   `initialDelaySeconds` 至关重要。必须给应用留出足够的启动时间，否则容器可能一启动就被杀。
*   `failureThreshold` 和 `periodSeconds` 共同决定了从故障发生到重启的响应时间。上例中，最坏情况是 `30 + 10 * 3 = 60` 秒后重启。

**2. TCP Socket 示例**
适用于非HTTP服务，如数据库、Redis等。

```yaml
    livenessProbe:
      tcpSocket:
        port: 6379
      initialDelaySeconds: 15
      periodSeconds: 20
```

**3. Exec 命令示例**
在容器内执行命令，如果命令退出码为0则视为成功。

```yaml
    livenessProbe:
      exec:
        command:
        - /bin/sh
        - -c
        - pgrep -f “java.*MyApp” || exit 1
      periodSeconds: 30
```

## 三、Readiness Probe：你的应用准备好“接客”了吗？

**核心作用**：判断容器是否已准备好接收流量。如果探测失败，Kubernetes的Service会将该Pod从负载均衡端点（Endpoint）列表中移除。

**关键区别**：Readiness Probe失败**不会重启容器**，只是将其与网络流量隔离。

**适用场景**：
1.  **应用启动慢**：需要加载大量数据或配置。
2.  **依赖项初始化**：等待数据库连接、外部配置中心就绪。
3.  **临时性故障**：遇到暂时无法处理的请求（如依赖的下游服务超时），需要短暂“下线”自我恢复。
4.  **优雅停止**：在Pod终止前，让Readiness Probe先失败，确保流量不再进入正在关闭的Pod。

**不配置的风险**：在滚动更新或Pod刚启动时，流量可能会被分配到尚未准备好的实例上，导致请求报错，影响用户体验和发布成功率。

### 配置示例

Readiness Probe的配置语法与Liveness Probe完全一致。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app
        image: my-app:v2
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
          # 通常Readiness Probe的成功阈值可以设高一点，避免因瞬时抖动被踢出
          successThreshold: 2
          failureThreshold: 1
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
```

**最佳实践**：为Readiness和Liveness设置不同的端点（如`/ready`和`/health`）。`/ready`的检查可以包含外部依赖状态，而`/health`只检查应用内部状态，避免因数据库临时故障导致应用被不必要的重启。

## 四、实战场景剖析

### 场景一：滚动更新与零停机部署

没有Readiness Probe的更新：
1.  新版本Pod（v2）启动。
2.  Service立即将流量切到v2 Pod。
3.  v2 Pod还在初始化，请求失败。
4.  用户体验到错误。

**配置了Readiness Probe后**：
1.  v2 Pod启动，但`/ready`检查未通过。
2.  Service的Endpoints列表不包含该v2 Pod，流量仍由旧Pod（v1）处理。
3.  v2 Pod完成初始化，`/ready`检查通过，被加入Endpoints列表，开始接收流量。
4.  旧v1 Pod被终止。
**结果**：平滑、无感知的更新。

### 场景二：依赖服务故障时的自保护

假设你的应用强依赖一个外部Redis。当Redis网络闪断时：
*   **Liveness Probe**（检查内部状态）：应继续通过，因为应用进程本身是健康的。
*   **Readiness Probe**（检查`/ready`，包含Redis连接）：应失败，使Pod暂时从Service下线。
*   **效果**：用户请求不会被路由到当前无法提供完整服务的Pod，而是由其他健康的Pod（如果Redis是部分故障）处理，或者快速返回“服务暂不可用”的友好提示。当Redis恢复后，Readiness Probe自动通过，Pod重新上线。**这避免了应用被不必要的重启，也防止了用户看到深层内部错误。**

### 场景三：区分“崩溃”与“繁忙”

一个高负载的Pod可能响应变慢，导致健康检查超时。
*   如果只用Liveness Probe，繁忙的Pod可能被误杀重启，加剧集群动荡。
*   **正确做法**：将Liveness Probe的超时时间（`timeoutSeconds`）设置得宽松一些，或将其检查频率（`periodSeconds`）降低。同时，确保Readiness Probe能更快地失败，将过载的Pod从流量池中摘除，给它时间恢复，而不是直接杀死它。

## 五、配置策略与高级技巧

1.  **参数调优黄金法则**：
    *   `initialDelaySeconds` > 应用最长启动时间。
    *   `periodSeconds`：根据业务敏感度，通常5-10秒。
    *   `timeoutSeconds`：略大于应用健康检查接口的P99响应时间。
    *   `failureThreshold`：Liveness可设2-3次，避免抖动；Readiness可设1-2次，快速下线。
    *   `successThreshold`：对于启动后状态可能波动的应用，Readiness可设为2-3，确保稳定后再上线。

2.  **使用命名端口**：使配置更清晰。
    ```yaml
    ports:
    - name: http
      containerPort: 8080
    livenessProbe:
      httpGet:
        port: http # 使用端口名称而非数字
        path: /health
    ```

3.  **在代码中实现健康检查端点**：不要仅仅返回静态OK。Liveness端点应检查应用核心内部状态（如内存、线程池）；Readiness端点应检查所有关键外部依赖（数据库、消息队列、配置中心）。

## 六、总结：别再跳过它们

Liveness和Readiness探针不是Kubernetes的可选装饰，而是构建**弹性（Resilient）** 和**可观测（Observable）** 应用的基础设施。

*   **Liveness Probe** 是“外科医生”，负责切除坏死的组织（重启故障容器），保证个体活力。
*   **Readiness Probe** 是“交通指挥官”，负责调度流量，确保请求只被导向真正有能力处理的实例，保障整体服务的连贯性。

忽略它们，就等于将应用的健康交给运气。合理配置它们，你的应用将获得自动故障恢复、平滑发布和依赖故障隔离的能力，从而在动态的容器化环境中稳健运行。从下一个部署开始，请务必为你的每个Pod加上这两道“保险”。