---
layout: post
title: 从代码到容器：实战 Jenkins/GitLab CI/CD 流水线自动发布应用到 Kubernetes
categories: [k8s]
description: 本文详细介绍了如何构建一个完整的CI/CD流水线，利用Jenkins或GitLab CI自动完成代码构建、Docker镜像打包、推送至镜像仓库，并最终自动部署应用到Kubernetes集群的全过程，包含详细的配置步骤和代码示例。
keywords: CI/CD, Jenkins, GitLab, Kubernetes, Docker
comments: false
---

# 从代码到容器：实战 Jenkins/GitLab CI/CD 流水线自动发布应用到 Kubernetes

在现代软件开发中，持续集成和持续部署（CI/CD）已成为提升交付效率、保证软件质量的基石。当我们将应用部署的目标环境转向 Kubernetes（K8s）时，自动化流程显得尤为重要。本文将手把手带你搭建两条主流的自动化流水线：基于 **Jenkins** 和基于 **GitLab CI/CD**，实现从代码提交到应用在 K8s 集群中自动更新的完整过程。

## 核心流程与架构概览

无论使用哪种工具，我们的自动化流水线都遵循一个经典流程：

1.  **代码提交**：开发者将代码推送到 Git 仓库（如 GitLab、GitHub）。
2.  **触发构建**：代码推送事件自动触发 CI/CD 流水线。
3.  **代码质量检查**：运行单元测试、代码静态分析等。
4.  **构建镜像**：使用 Docker 将应用打包成容器镜像。
5.  **推送镜像**：将构建好的镜像推送到镜像仓库（如 Docker Hub、Harbor、GitLab Registry）。
6.  **更新部署**：更新 Kubernetes 的部署清单（如 Deployment YAML），触发应用在 K8s 集群中滚动更新。
7.  **状态验证**：检查新 Pod 是否健康启动。

整个流程的架构如下图所示（以 GitLab 为例）：
```
[GitLab Repo] -> [GitLab CI Runner / Jenkins Agent] -> [Docker Build] -> [Image Registry] -> [Kubectl Apply] -> [Kubernetes Cluster]
```

下面，我们分别详解两种实现方案。

## 方案一：基于 Jenkins 的自动化流水线

Jenkins 是一个功能强大、插件生态丰富的开源自动化服务器。我们使用经典的 **Pipeline-as-Code** 方式，将流水线定义在项目根目录的 `Jenkinsfile` 中。

### 前提准备

1.  **Jenkins 服务器**：已安装并配置好必要的插件，如 `Pipeline`、`Docker Pipeline`、`Kubernetes CLI` (kubectl)。
2.  **Kubernetes 集群**：拥有一个可访问的 K8s 集群，并准备好 kubeconfig 文件。
3.  **镜像仓库凭证**：在 Jenkins 中配置访问 Docker Hub 或私有仓库的 Username/Password 凭证（假设凭证ID为 `docker-hub-cred`）。
4.  **Kubernetes 凭证**：在 Jenkins 中配置 kubeconfig 文件或 ServiceAccount 令牌凭证（假设凭证ID为 `k8s-cluster-cred`）。

### Jenkinsfile 详解

以下是一个面向 Spring Boot 应用的 `Jenkinsfile` 示例，它定义了完整的构建、推送、部署阶段。

```groovy
pipeline {
    agent any // 指定在任何可用代理上运行
    environment {
        // 定义环境变量
        DOCKER_IMAGE = 'your-dockerhub-username/my-spring-app'
        DOCKER_TAG = "${env.BUILD_ID}" // 使用构建ID作为标签
        K8S_NAMESPACE = 'default'
        DEPLOYMENT_NAME = 'my-spring-app-deployment'
    }
    stages {
        stage('Checkout') {
            steps {
                // 拉取源代码
                checkout scm
            }
        }
        stage('Unit Test') {
            steps {
                sh './mvnw test' // 假设是Maven项目
            }
        }
        stage('Build and Push Docker Image') {
            steps {
                script {
                    // 使用Docker构建镜像
                    docker.build("${DOCKER_IMAGE}:${DOCKER_TAG}")
                    // 登录镜像仓库并推送
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-cred', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh "echo ${DOCKER_PASS} | docker login -u ${DOCKER_USER} --password-stdin"
                        docker.push("${DOCKER_IMAGE}:${DOCKER_TAG}")
                    }
                }
            }
        }
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    // 使用配置的kubectl凭据
                    withCredentials([file(credentialsId: 'k8s-cluster-cred', variable: 'KUBECONFIG')]) {
                        // 方法1：直接使用kubectl set image命令（需提前存在Deployment）
                        sh "kubectl --kubeconfig=${KUBECONFIG} -n ${K8S_NAMESPACE} set image deployment/${DEPLOYMENT_NAME} ${DEPLOYMENT_NAME}=${DOCKER_IMAGE}:${DOCKER_TAG} --record"
                        
                        // 方法2：渲染并应用更新的YAML文件（更通用）
                        // 读取项目内的k8s部署模板，替换镜像标签
                        sh """
                            sed -e 's|{{IMAGE}}|${DOCKER_IMAGE}:${DOCKER_TAG}|g' k8s/deployment.yaml.tpl > k8s/deployment.yaml
                            kubectl --kubeconfig=${KUBECONFIG} -n ${K8S_NAMESPACE} apply -f k8s/deployment.yaml
                        """
                    }
                }
            }
        }
        stage('Verify Deployment') {
            steps {
                script {
                    withCredentials([file(credentialsId: 'k8s-cluster-cred', variable: 'KUBECONFIG')]) {
                        // 等待部署完成，检查新Pod是否就绪
                        sh """
                            kubectl --kubeconfig=${KUBECONFIG} -n ${K8S_NAMESPACE} rollout status deployment/${DEPLOYMENT_NAME} --timeout=300s
                        """
                    }
                }
            }
        }
    }
    post {
        success {
            echo 'Pipeline succeeded! The application has been deployed to Kubernetes.'
        }
        failure {
            echo 'Pipeline failed. Check the logs for details.'
        }
    }
}
```

**关键点说明：**
*   **凭证管理**：使用 `withCredentials` 块安全地注入敏感信息，避免在日志中泄露。
*   **镜像标签**：使用 `BUILD_ID` 确保每次构建的镜像唯一，便于追踪和回滚。
*   **部署策略**：示例展示了两种更新 K8s 应用的方式。`kubectl set image` 更快捷，但要求 Deployment 已存在；通过模板渲染 YAML 文件则更灵活，可以管理完整的资源定义。
*   **健康检查**：`rollout status` 命令会阻塞直到 Deployment 的滚动更新成功或超时，这是一个重要的验证步骤。

## 方案二：基于 GitLab CI/CD 的自动化流水线

GitLab 内置了强大的 CI/CD 功能，配置更加简洁直观。我们通过在项目根目录创建 `.gitlab-ci.yml` 文件来定义流水线。

### 前提准备

1.  **GitLab Runner**：已为项目或群组注册并配置好 Runner。Runner 需要安装 Docker 和 kubectl 命令行工具。
2.  **Kubernetes 集群**：同上。
3.  **GitLab 变量**：在 GitLab 项目的 `Settings > CI/CD > Variables` 中，安全地配置以下变量：
    *   `DOCKER_REGISTRY_USER` / `DOCKER_REGISTRY_PASSWORD`：用于登录镜像仓库。
    *   `KUBECONFIG`：整个 kubeconfig 文件的内容（注意选择 `File` 类型变量，Runner会将其保存为临时文件）。

### .gitlab-ci.yml 详解

```yaml
# 定义流水线阶段
stages:
  - test
  - build
  - deploy

# 使用Docker-in-Docker (dind) 服务来构建镜像
variables:
  DOCKER_IMAGE: $CI_REGISTRY_IMAGE # 使用GitLab容器仓库地址
  DOCKER_TAG: $CI_COMMIT_SHORT_SHA # 使用提交哈希作为标签
  K8S_NAMESPACE: "production"

# 所有Job的默认镜像，包含Maven, Docker, kubectl
image: maven:3.8.4-openjdk-11-slim

before_script:
  - apt-get update && apt-get install -y docker.io kubectl # 确保工具存在

unit-test:
  stage: test
  script:
    - ./mvnw test
  only:
    - merge_requests
    - main

build-push-image:
  stage: build
  services:
    - docker:20.10.12-dind # 启动dind服务
  script:
    - echo "Building Docker image..."
    - docker build -t $DOCKER_IMAGE:$DOCKER_TAG .
    - echo "Logging in to GitLab Container Registry..."
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - echo "Pushing Docker image..."
    - docker push $DOCKER_IMAGE:$DOCKER_TAG
  only:
    - main

deploy-to-k8s:
  stage: deploy
  image: bitnami/kubectl:latest # 使用包含kubectl的镜像
  script:
    - echo "Deploying to Kubernetes..."
    # 由于KUBECONFIG是File类型变量，其路径保存在变量$KUBECONFIG中
    - kubectl --kubeconfig=$KUBECONFIG get nodes # 测试连接
    # 使用envsubst替换k8s部署模板中的变量
    - envsubst < k8s/deployment.yaml.tpl > k8s/deployment.yaml
    - kubectl --kubeconfig=$KUBECONFIG -n $K8S_NAMESPACE apply -f k8s/deployment.yaml
    - echo "Waiting for rollout to complete..."
    - kubectl --kubeconfig=$KUBECONFIG -n $K8S_NAMESPACE rollout status deployment/my-app-deployment --timeout=300s
  environment:
    name: production
    url: https://my-app.example.com # 你的应用访问地址
  only:
    - main
```

**关键点说明：**
*   **内置变量**：GitLab CI 提供了大量预定义变量，如 `CI_COMMIT_SHORT_SHA`、`CI_REGISTRY_IMAGE`，非常方便。
*   **Docker-in-Docker**：通过 `services` 启动 `dind` 服务，使 Runner 能够在容器内构建 Docker 镜像。
*   **GitLab Container Registry**：示例直接使用 GitLab 内置的容器仓库，无需额外配置。
*   **环境部署**：`environment` 关键字会在 GitLab UI 中创建“环境”链接，并显示每次部署的状态，非常直观。
*   **条件执行**：`only` 关键字控制 Job 在哪些分支或事件下触发。例如，测试阶段在合并请求和主分支运行，而构建和部署仅在主分支推送时运行。

### Kubernetes 部署模板示例

无论是 Jenkins 还是 GitLab CI，我们通常使用一个模板文件来定义 Deployment。下面是一个简单的 `k8s/deployment.yaml.tpl` 示例：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-deployment
  namespace: ${K8S_NAMESPACE}
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app-container
        image: ${DOCKER_IMAGE}:${DOCKER_TAG} # 这个变量会被CI/CD流水线替换
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
  namespace: ${K8S_NAMESPACE}
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: ClusterIP
```

## 最佳实践与进阶思考

1.  **安全性**：
    *   永远不要在代码或日志中硬编码密码、令牌。
    *   使用角色的最小权限原则（RBAC）为 CI/CD 系统在 K8s 中创建专用的 ServiceAccount，赋予其仅限于部署命名空间的必要权限，而不是直接使用集群管理员凭证。
2.  **镜像策略**：
    *   考虑使用“不可变标签”，如 `git-commit-sha`，而非 `latest`。
    *   实施镜像漏洞扫描，可以将其集成在“构建”阶段之后。
3.  **部署策略**：
    *   结合使用 `kubectl rollout undo` 可以快速实现回滚。
    *   对于更复杂的发布（如金丝雀发布、蓝绿部署），可以考虑使用 Flagger 或 Argo Rollouts 等高级工具，CI/CD 流水线只需负责提供新镜像。
4.  **流水线优化**：
    *   利用 Docker 层缓存和 Maven/Gradle 缓存来加速构建。
    *   将流水线拆分为更细粒度的 Job，并利用 `needs` 关键字（GitLab CI）或 `parallel` 指令（Jenkins）实现并行执行，缩短整体耗时。

## 总结

通过 Jenkins 或 GitLab CI/CD，我们能够建立起一套高效、可靠的自动化流水线，将开发者从繁琐的构建、部署工作中解放出来，实现“一次提交，处处运行”的云原生开发体验。Jenkins 以其灵活性和强大的插件生态著称，适合复杂、定制化需求高的场景；而 GitLab CI/CD 则提供了开箱即用、与代码仓库深度集成的简洁体验。

选择哪种工具取决于你的团队技术栈、基础设施和个人偏好。但无论选择哪条路径，拥抱自动化、践行 CI/CD 文化，都是通往高效 DevOps 和云原生成功的必经之路。现在，就根据你的项目情况，开始搭建属于你自己的自动化发布流水线吧！