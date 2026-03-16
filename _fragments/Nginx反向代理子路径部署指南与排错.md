---
layout: fragment
title: Nginx反向代理子路径部署指南与排错 SOE
tags: [nginx]
description: 小case
keywords: nginx, 404, kafka-ui
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---
# Nginx 反向代理子路径部署指南与排错 SOP

## 📝 问题背景
在通过 Nginx 将服务（如 Kafka UI、Grafana 等）代理到特定子路径（例如 `/kafka`）时，经常会遇到以下经典问题：
- **症状：** 可以访问到基础 HTML 页面，但所有静态资源（JS/CSS/图片）全部报 `404 Not Found`。
- **原因：** 后端应用默认运行在根目录 `/` 下，生成的静态资源链接为绝对路径（如 `/assets/main.js`）。当浏览器去请求这些资源时，Nginx 无法正确将其路由到对应的子路径服务中。

---

## 💡 核心心法：应用层优先，Nginx 辅助
**当静态资源 404 时，不要第一时间去 Nginx 里写复杂的 `rewrite` 规则！** 最优雅、最彻底的解决方案是：**修改后端应用的“上下文路径（Context Path / Base URL）”**，让应用自己知道它运行在哪个子路径下。

---

## 🛠️ 标准处理 4 步曲

### 第 1 步：配置应用的 Base Path (以 Docker 为例)
查阅目标应用的官方文档，找到修改 Context Path 或 Base URL 的环境变量。

以 Kafka UI 为例，修改 `docker-compose.yml`：
```yaml
services:
  kafka-ui:
    image: provectuslabs/kafka-ui:v0.7.2
    # ... 其他配置 ...
    environment:
      # 告诉 Kafka UI 它的根目录是 /kafka
      - SERVER_SERVLET_CONTEXT_PATH=/kafka
      - AUTH_TYPE=NONE
第 2 步：彻底重建 Docker 容器 (⚠️ 极易踩坑)
修改 docker-compose.yml 后，单纯执行 restart 是无效的，环境变量不会被重新加载。必须销毁旧容器并重建：

Bash

# 停止并移除旧容器
docker-compose down

# 用最新配置启动新容器
docker-compose up -d
第 3 步：隔离验证（绕过 Nginx 直接测试后端）
在修改 Nginx 前，先确认后端应用是否已经正确挂载到了子路径。在服务器上使用 curl 进行测试：

Bash

# 测试 1：访问带前缀的目录，预期返回 HTTP 200 OK
curl -I [http://127.0.0.1:8082/kafka/](http://127.0.0.1:8082/kafka/)

# 测试 2：访问根目录，预期返回 404 (说明服务确实搬家了)
curl -I [http://127.0.0.1:8082/](http://127.0.0.1:8082/)
注：只有当测试 1 返回 200 OK 时，才能进行下一步。

第 4 步：套用 Nginx “神圣斜杠法则”
遵循反向代理的黄金公式：location 结尾带斜杠，proxy_pass 结尾坚决不带斜杠（且不包含任何路径）。 这样可以保证请求原汁原味地透传给后端。

完整的 Nginx server 块配置参考如下：
```
Nginx

server {
    listen 80;
    server_name localhost; # 或你的域名/IP

    # ==========================================
    # 补丁：当用户手滑只输入 /kafka (无斜杠) 时，自动 301 重定向到 /kafka/
    # 防止严格匹配模式下直接报 404
    # ==========================================
    location = /kafka {
        return 301 /kafka/;
    }

    # ==========================================
    # 核心业务转发模块
    # ==========================================
    location /kafka/ {
        # 纯净透传：结尾千万不要加 / ，也不要加 /kafka/
        proxy_pass [http://192.168.101.89:8082](http://192.168.101.89:8082);
        
        # 超时设置
        proxy_connect_timeout 5s;
        proxy_send_timeout 300s;
        proxy_read_timeout 300s;
        
        # 传递客户端真实请求信息给后端（必加三件套）
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
🔍 终极排错口诀 (Troubleshooting Cheat Sheet)
下次再遇到 404 问题，按以下顺序排查：

浏览器 Network 报 404： 检查请求的 URL 里，有没有带上预期的子路径前缀？（如果请求的是 IP/assets.js 而不是 IP/kafka/assets.js，说明第 1 步没生效）。

Nginx 日志报 404： 检查 proxy_pass 结尾是不是多写了斜杠或路径，导致 Nginx 触发了路径替换（剥离了子路径）。

后端日志报 404： 后端应用根本不知道自己住在子路径里，去检查 Docker 环境变量是否正确注入（参考第 2 步）。
