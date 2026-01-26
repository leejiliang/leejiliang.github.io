---
layout: post
title: 实战指南：使用Prometheus+Grafana全方位监控K8s集群
categories: [k8s]
description: 本文详细介绍了如何在Kubernetes集群中部署和配置Prometheus与Grafana，实现从节点资源到Pod应用的全链路指标监控，并提供了完整的部署脚本和可视化仪表盘配置。
keywords: Kubernetes, Prometheus, Grafana, 监控, 指标
comments: false
---

# 实战指南：使用Prometheus+Grafana全方位监控K8s集群

在云原生时代，Kubernetes已成为容器编排的事实标准。然而，随着集群规模的扩大和微服务架构的复杂化，如何实时洞察集群的健康状态、资源利用率以及应用性能，成为运维和开发团队面临的核心挑战。一个强大、可视化的监控系统不再是“锦上添花”，而是保障服务稳定性的“生命线”。本文将手把手带你搭建基于Prometheus和Grafana的Kubernetes监控方案，实现对集群的“全景式”观测。

## 一、监控方案架构与核心组件

我们的监控体系主要基于以下两个核心开源项目：

1.  **Prometheus**：CNCF毕业项目，一个强大的时间序列数据库和监控系统。它通过主动“拉取”（Pull）模式从目标收集指标，并提供了强大的查询语言PromQL。
2.  **Grafana**：领先的开源数据可视化平台，可以将Prometheus收集的指标数据转化为直观的图表、仪表盘和告警。

在K8s中，我们通常需要监控多个层次：
*   **节点层**：CPU、内存、磁盘、网络等物理资源。
*   **集群层**：API Server、Scheduler、Controller Manager等核心组件。
*   **应用层**：Deployment、Pod、Service的运行状态与自定义业务指标。

为了自动发现和采集这些指标，我们需要在集群中部署一系列Exporter和监控对象。

## 二、部署Prometheus监控栈

我们将使用`prometheus-community/kube-prometheus-stack`这个Helm Chart，它集成了Prometheus Operator、Grafana以及一系列针对K8s的监控规则和仪表盘，极大简化了部署流程。

### 1. 前提条件
*   一个运行中的Kubernetes集群（v1.16+）
*   已安装`kubectl`和`helm`命令行工具
*   集群中有可用的StorageClass（用于持久化存储）

### 2. 添加Helm仓库并部署

```bash
# 添加Prometheus社区Helm仓库
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# 创建独立的命名空间
kubectl create namespace monitoring

# 安装kube-prometheus-stack
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
    --namespace monitoring \
    --set prometheus.prometheusSpec.storageClass="<your-storage-class-name>" \
    --set grafana.persistence.enabled=true \
    --set grafana.persistence.storageClassName="<your-storage-class-name>"
```

请将`<your-storage-class-name>`替换为你集群中实际的存储类名称。

### 3. 验证部署

部署完成后，检查相关Pod是否正常运行：

```bash
kubectl get pods -n monitoring
```

你应该能看到类似以下的Pod：
- `prometheus-kube-prometheus-stack-prometheus-0`
- `kube-prometheus-stack-grafana-*`
- `kube-prometheus-stack-operator-*`
- 以及多个`kube-state-metrics-*`和`node-exporter-*`。

## 三、访问Grafana与Prometheus UI

部署的组件默认以ClusterIP类型服务暴露。为了方便访问，我们可以临时使用端口转发，或创建Ingress规则。

### 1. 端口转发访问Grafana

```bash
# 获取Grafana的Service名称
kubectl get svc -n monitoring | grep grafana

# 端口转发到本地
kubectl port-forward svc/kube-prometheus-stack-grafana -n monitoring 3000:80
```

现在，在浏览器中访问 `http://localhost:3000`。默认用户名是`admin`，密码可以通过以下命令获取：

```bash
kubectl get secret --namespace monitoring kube-prometheus-stack-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

### 2. 端口转发访问Prometheus

```bash
kubectl port-forward svc/kube-prometheus-stack-prometheus -n monitoring 9090:9090
```

访问 `http://localhost:9090` 即可使用Prometheus的原生查询界面。

## 四、探索预置的监控仪表盘

登录Grafana后，你会发现已经预置了大量开箱即用的仪表盘。在左侧导航栏点击 **Dashboards -> Browse**，可以搜索到诸如：
*   **Kubernetes / Compute Resources / Cluster**：集群整体资源视图。
*   **Kubernetes / Compute Resources / Namespace (Pods)**：命名空间下Pod的资源使用情况。
*   **Kubernetes / Compute Resources / Pod**：单个Pod的详细资源监控。
*   **Node Exporter / Nodes**：集群节点的详细系统指标（CPU、内存、磁盘IO、网络等）。
*   **Prometheus / Targets**：显示Prometheus所有抓取目标的状态。

这些仪表盘已经配置了关键的PromQL查询，为我们提供了立即可用的监控能力。

## 五、核心监控指标与PromQL示例

理解关键指标和PromQL是定制化监控和告警的基础。以下是一些常用场景的示例：

### 1. 集群节点CPU使用率
```promql
# 计算所有节点非空闲CPU的平均使用率
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

### 2. 集群节点内存使用率
```promql
# 计算所有节点内存使用百分比
(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100
```

### 3. Pod内存使用量（前10名）
```promql
# 按Pod统计内存使用量（RSS），并排序
topk(10, sum by (pod, namespace) (container_memory_working_set_bytes{container!="", image!=""}))
```

### 4. K8s Deployment副本可用性
```promql
# 显示每个Deployment期望副本数与就绪副本数的对比
kube_deployment_spec_replicas{namespace="default"}
vs
kube_deployment_status_replicas_available{namespace="default"}
```

你可以在Grafana的“Explore”页面或Prometheus的“Graph”页面中尝试运行这些查询。

## 六、配置自定义告警规则

Prometheus Operator通过`PrometheusRule` CRD资源管理告警规则。我们可以创建自己的规则文件。

创建一个名为 `custom-alerts.yaml` 的文件：

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: custom-k8s-alerts
  namespace: monitoring
  labels:
    release: kube-prometheus-stack # 此标签很重要，确保规则被关联
spec:
  groups:
  - name: kubernetes-resources
    rules:
    - alert: NodeHighCPUUsage
      expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "高CPU使用率 (实例 {{ $labels.instance }})"
        description: "节点 {{ $labels.instance }} 的CPU使用率持续高于80% 达10分钟，当前值为 {{ $value }}%。"
    - alert: PodCrashLooping
      expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
      for: 2m
      labels:
        severity: critical
      annotations:
        summary: "Pod频繁重启 ({{ $labels.namespace }}/{{ $labels.pod }})"
        description: "Pod {{ $labels.pod }} 在过去15分钟内重启了 {{ $value }} 次，可能应用存在故障。"
```

应用这个规则：
```bash
kubectl apply -f custom-alerts.yaml
```

告警产生后，Prometheus会将它们发送给Alertmanager（已随Chart部署），由Alertmanager进行分组、抑制并路由到如邮件、Slack、Webhook等接收器。

## 七、监控自定义应用（以Spring Boot为例）

要监控业务应用，需要应用本身暴露Prometheus格式的指标端点。对于Spring Boot应用，这非常简单：

1.  在`pom.xml`中添加依赖：
    ```xml
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
    </dependency>
    ```

2.  在`application.yml`中启用端点：
    ```yaml
    management:
      endpoints:
        web:
          exposure:
            include: prometheus,health,info
      metrics:
        tags:
          application: ${spring.application.name}
    ```

3.  应用部署到K8s后，需要创建`ServiceMonitor`资源，告诉Prometheus Operator去抓取这个新目标。创建一个 `myapp-service-monitor.yaml`：
    ```yaml
    apiVersion: monitoring.coreos.com/v1
    kind: ServiceMonitor
    metadata:
      name: springboot-app-monitor
      namespace: your-app-namespace # 你的应用所在命名空间
    spec:
      selector:
        matchLabels:
          app: your-springboot-app # 匹配你的Service标签
      endpoints:
      - port: http-management # 对应Service中定义的端口名称
        path: /actuator/prometheus
      namespaceSelector:
        matchNames:
        - your-app-namespace
    ```
    应用此配置后，Prometheus会自动发现并开始抓取你的应用指标。

## 八、总结与最佳实践

通过本文的实战，我们成功搭建了一个功能强大的K8s监控平台。总结一下关键点与最佳实践：

1.  **分层监控**：清晰划分节点、集群、应用、中间件等监控层次。
2.  **标签（Labels）是关键**：为Pod、Service等资源定义清晰、一致的标签，这是Prometheus数据建模和高效查询的基础。
3.  **指标黄金法则**：遵循“USE”（Utilization, Saturation, Errors）和“RED”（Rate, Errors, Duration）方法论来定义核心监控指标。
4.  **告警有效性**：避免告警疲劳。确保告警具有明确的触发条件、修复预案，并区分严重等级。
5.  **持久化存储**：务必为Prometheus和Grafana配置持久化存储，防止数据丢失。
6.  **定期维护**：定期审查和优化PromQL查询，清理过期指标，调整数据保留策略。

Prometheus + Grafana的组合为我们提供了从基础设施到业务应用的完整可观测性视角。掌握它，就如同为你的Kubernetes集群装上了“全景仪表盘”和“预警雷达”，是保障系统稳定、高效运行的基石。现在，就动手为你自己的集群部署一套，开始你的监控之旅吧！