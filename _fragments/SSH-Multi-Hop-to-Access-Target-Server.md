---
layout: fragment
title: SSH Multi-Hop to Access Target Server
tags: [golang]
description: 小case
keywords: ssh
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

### 场景描述
我要访问一个云数据库，但是云数据库只能在指定的服务器A（116.62.53.53）上才能访问，所以需要先跳到A上，但是A也不是本地环境可以直接连接的，需要通过一台有公网IP的服务器B（116.62.53.52）才能访问，而且B也不是本地能直接能连的，需要通过办公室的一台指定机器C（10.167.69.123）才能跳转上去，
所以最终的流程大概是： local:5433 -> 10.167.69.123  ->  116.62.53.52 -> 116.62.53.53 -> db：5432
```
ssh -L 5433:pgm-bpddddddddd.pg.rds.aliyuncs.com:5432 user4@116.62.53.53 \
    -J user1@10.167.69.123,user3@116.62.53.52
```
