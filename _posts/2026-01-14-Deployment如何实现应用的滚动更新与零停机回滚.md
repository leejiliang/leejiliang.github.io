---
layout: post
title: 掌握Kubernetes部署艺术：实现滚动更新与零停机回滚的实战指南
categories: [k8s]
description: 本文深入探讨在Kubernetes中如何实现应用的滚动更新与零停机回滚。通过详细的策略解析、YAML配置示例和实际操作步骤，帮助您构建高可用、可平滑升级的服务部署体系。
keywords: Kubernetes, 滚动更新, 零停机部署, Deployment, 回滚策略
comments: false
---

# 掌握Kubernetes部署艺术：实现滚动更新与零停机回滚的实战指南

在现代云原生应用的交付流程中，如何在不中断服务的前提下安全地发布新版本，并在出现问题时快速、优雅地回滚，是每个DevOps工程师和SRE必须掌握的核心技能。Kubernetes的`Deployment`资源对象，正是为此而生。它抽象了Pod的管理，并内置了强大的滚动更新（Rolling Update）和回滚（Rollback）机制。本文将深入解析其工作原理，并提供从配置到实战的完整指南。

## 一、 核心概念：为什么需要滚动更新与回滚？

在传统部署中，发布新版本通常意味着“停机发布”——先停止所有旧实例，再启动新实例。这会导致服务中断，影响用户体验，甚至造成业务损失。而“滚动更新”则是一种渐进式替换策略，它逐步用新版本的Pod替换旧版本的Pod，在整个过程中始终确保有足够数量的Pod在运行以处理请求，从而实现**零停机（Zero Downtime）**。

“回滚”则是发布安全的保险丝。当新版本上线后出现不可预见的Bug或性能问题时，能够一键式、快速地将应用状态恢复到上一个（或任意指定）稳定版本，同样是保障服务高可用的关键。

Kubernetes的`Deployment`控制器完美地封装了这两种能力。

## 二、 Deployment滚动更新策略深度解析

创建一个`Deployment`时，其更新策略由`spec.strategy`字段定义。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
spec:
  replicas: 4
  strategy:
    type: RollingUpdate # 指定滚动更新策略
    rollingUpdate:
      maxSurge: 1        # 最大激增Pod数
      maxUnavailable: 0  # 最大不可用Pod数
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp-container
        image: myapp:v1.0.0 # 初始镜像版本
        ports:
        - containerPort: 80
```

**关键参数解读：**

1.  **`strategy.type`**: 可以是`RollingUpdate`（滚动更新）或`Recreate`（重建）。`Recreate`会先杀掉所有旧Pod，再创建新Pod，会导致服务中断，仅适用于开发测试环境。
2.  **`maxSurge`**: 更新过程中，允许创建的超过`replicas`数量的最大Pod数。可以是一个绝对数字（如`1`）或百分比（如`25%`）。例如，`replicas=4, maxSurge=1`，则在更新时，Pod总数最多可以达到5个（4旧+1新，或3旧+2新等）。这决定了更新的“激进”程度。
3.  **`maxUnavailable`**: 更新过程中，允许不可用的Pod实例数（相对于`replicas`）。同样可以是数字或百分比。例如，`replicas=4, maxUnavailable=0`，意味着Kubernetes会确保在更新时，始终有4个Pod可用（通过先创建新Pod，再删除旧Pod的方式）。这决定了服务的“可用性”底线。

**滚动更新流程（假设`replicas=4, maxSurge=1, maxUnavailable=0`）：**
1.  当更新镜像（如`image: myapp:v1.0.0` -> `myapp:v2.0.0`）并应用配置后，Deployment控制器会创建一个新的ReplicaSet（RS-v2）。
2.  由于`maxSurge=1`，RS-v2可以创建一个新Pod（此时总Pod数为5）。
3.  新Pod进入`Ready`状态后，旧ReplicaSet（RS-v1）中的一个Pod被删除（此时总Pod数回到4，其中3旧1新）。
4.  重复步骤2和3，直到所有4个Pod都被RS-v2管理的新Pod替换。
5.  在整个过程中，由于`maxUnavailable=0`，始终保证至少有4个Pod可用，服务流量不会中断。

## 三、 实战：触发与观察滚动更新

**1. 触发更新：**
最常用的方式是更新Pod模板中的镜像标签。

```bash
# 方法1：使用kubectl set image
kubectl set image deployment/myapp-deployment myapp-container=myapp:v2.0.0

# 方法2：使用kubectl edit直接修改Deployment配置
kubectl edit deployment myapp-deployment
# 然后在编辑器中修改 `spec.template.spec.containers[0].image` 字段

# 方法3：修改本地YAML文件后apply
kubectl apply -f deployment.yaml
```

**2. 观察更新过程：**

```bash
# 实时观察Pod的替换状态
kubectl get pods -l app=myapp -w

# 查看Deployment的详细状态和事件
kubectl describe deployment myapp-deployment

# 查看ReplicaSet的演变，可以看到新旧两个RS
kubectl get replicasets -l app=myapp

# 查看更新状态
kubectl rollout status deployment/myapp-deployment
```
输出`rollout status`会显示类似“Waiting for rollout to finish: 3 out of 4 new replicas have been updated...”的信息，直到最终完成。

## 四、 实现零停机回滚：安全网机制

即使测试再充分，线上问题也难以完全避免。Kubernetes为Deployment保存了历史版本记录（受修订版本历史限制`revisionHistoryLimit`控制），使得回滚操作极其简单。

**1. 查看发布历史：**
```bash
kubectl rollout history deployment/myapp-deployment
# 输出会显示各个修订版本（REVISION）和其使用的镜像

# 查看特定版本的详细信息
kubectl rollout history deployment/myapp-deployment --revision=2
```

**2. 执行回滚：**
```bash
# 回滚到上一个版本（最常用）
kubectl rollout undo deployment/myapp-deployment

# 回滚到指定的历史版本（例如版本2）
kubectl rollout undo deployment/myapp-deployment --to-revision=2
```
`kubectl rollout undo`命令会触发一次新的“滚动更新”，只不过目标版本是历史版本，从而实现**零停机回滚**。

**3. 回滚原理：**
回滚操作本质上是将Deployment的Pod模板（`.spec.template`）替换为指定历史修订版本中存储的模板。控制器会再次执行一次滚动更新流程，用旧版本的Pod替换新版本的Pod。

## 五、 高级场景与最佳实践

**1. 配置`revisionHistoryLimit`：**
默认情况下，Deployment会保留10个旧的ReplicaSet以供回滚。你可以根据需求调整，平衡历史记录和资源占用。
```yaml
spec:
  revisionHistoryLimit: 5 # 只保留最近5个修订版本
```

**2. 集成就绪探针（Readiness Probe）：**
这是实现**真正零停机**的关键。如果新Pod的进程启动但服务尚未准备好（如数据库连接未建立），流量不应被导入。就绪探针能确保只有完全准备好的Pod才会被加入Service的端点（Endpoint）列表，接收流量。
```yaml
spec:
  template:
    spec:
      containers:
      - name: myapp-container
        image: myapp:v2.0.0
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 5 # 容器启动后5秒开始探测
          periodSeconds: 5       # 每5秒探测一次
          successThreshold: 1
          failureThreshold: 3
```

**3. 暂停与恢复更新：**
在复杂的多步骤更新中（如先更新镜像，再更新环境变量），可以使用暂停功能。
```bash
# 暂停更新
kubectl rollout pause deployment/myapp-deployment
# 此时可以进行多次配置修改
kubectl set image deployment/myapp-deployment myapp-container=myapp:v2.0.1
kubectl set env deployment/myapp-deployment DEPLOY_ENV=prod

# 恢复更新，一次性应用所有更改
kubectl rollout resume deployment/myapp-deployment
```

**4. 设置更新期限（Progress Deadline）：**
如果更新过程卡住（例如新镜像无法启动），可以设置一个超时时间，超时后Kubernetes会将更新标记为失败。
```yaml
spec:
  progressDeadlineSeconds: 600 # 10分钟后标记为失败
```
可以通过`kubectl rollout status`查看是否超时，超时后需要人工介入排查。

## 六、 总结

通过合理配置Kubernetes Deployment的滚动更新策略（`maxSurge`和`maxUnavailable`），并结合**就绪探针**，我们可以构建出能够实现**零停机发布**的健壮部署流程。而内置的`kubectl rollout`历史与回滚命令，则为这趟发布之旅提供了可靠的安全网，使得故障恢复变得快速且自动化。

将这套流程与CI/CD管道（如Jenkins、GitLab CI、ArgoCD等）集成，就能打造出高效、安全、可信赖的现代化软件交付生产线，真正做到“持续部署，信心发布”。