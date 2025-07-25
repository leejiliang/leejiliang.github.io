---
layout: post
title: LJL-VPN Requirements Document
categories: [Develop]
description: Make learning English a daily habit.
keywords: english, habit, learning
comments: false
---
学习了一段时间的golang开发，公司的项目比较偏向业务，且侧重点在于高并发方面，对于网络编程方面的涉及比较少，最多的场景就是发起一个http调用，综合考虑下来，我决定自己开发一个小小的越狱工具，没有任何不良意图。

### 原理
简单说就是：绕过网络封锁（如中国的 GFW）的方法，通过各种技术手段，将用户的网络请求“伪装、加密、转发”，让你能访问被封锁的网站（如 Google、YouTube、ChatGPT 等）。
后面可以专门抽出来一点时间去了解一下GFW的种种。
```
你电脑 → [翻墙客户端] → 加密数据 → [远程代理服务器（境外）] → 访问真实目标网站
                                                     ↑
                           将网站内容返回 → 解密 → 你本地显示
```


### 第一阶段开发目标
- 本地监听 SOCKS5 请求  浏览器或系统连接   
- 将请求封装为自定义协议     加密或加标识     
- 发送给远程服务端       用 TCP 连接   
- 服务端解包并转发       建立到目标网站的连接 
- 回传响应           给客户端解包 

### 工程结构规划
```
vpn-proxy/
├── cmd/
│   ├── client.go        # 本地代理客户端
│   └── server.go        # 远程转发服务端
├── internal/
│   ├── socks/           # SOCKS5 协议处理
│   ├── protocol/        # 自定义协议封装/解封
│   └── crypto/          # 加解密模块
├── config/
│   └── config.yaml
├── go.mod
└── README.md
```
