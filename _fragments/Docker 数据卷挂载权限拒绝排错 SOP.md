---
layout: fragment
title: Docker 数据卷挂载权限拒绝排错 SOP
tags: [mqtt]
description: 小case
keywords: mqtt, volumes
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

Docker 数据卷挂载权限拒绝排错 SOP
【问题症状】
使用 Docker 挂载本地目录时，容器不断重启，执行 docker logs 看到报错：
mkdir: cannot create directory ... Permission denied

【根本原因】
宿主机自动创建的文件夹属于 root 用户超级管理员。
容器内部运行程序的通常是普通用户（如 UID 1000）。
普通用户没有权限向 root 的文件夹写入数据，导致权限拒绝。

【标准处理步骤 (3步法)】

第 1 步：停止报错的容器，释放占用。
执行命令：
docker-compose down

第 2 步：将宿主机目录的所有权移交给容器内的普通用户（UID 1000）。
执行命令：
sudo chown -R 1000:1000 ./data ./log

第 3 步：重新启动容器并查看日志。
执行命令：
docker-compose up -d
执行命令：
docker logs emqx

【最佳实践】
以后写完 docker-compose.yml 后，在第一次执行启动前，先手动建好文件夹并改好权限：
mkdir -p ./data ./log
sudo chown -R 1000:1000 ./data ./log
docker-compose up -d
