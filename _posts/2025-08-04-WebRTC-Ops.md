---
layout: post
title: WebRTC 服务端的热更新实现
categories: [Ops]
description: 实现webrtc 服务的热更新.
keywords: linux，livekit, 多人视频，webrtc
comments: false
---
在 WebRTC 项目中，实现滚动部署（rolling deployment）是为了确保在服务升级或变更期间，系统能够持续提供服务且不中断正在进行的实时音视频通信。由于 WebRTC 通信对实时性和状态保持要求高，因此相比于传统的 Web 服务部署，滚动部署需要更加小心地处理“连接不中断”、“信令状态迁移”和“媒体转发稳定”等问题。

### 实际需求
我手上现有一个livekit 服务端的业务服务(后面称bus-server)，现处于开发迭代业务功能阶段，经常需要发版升级，考虑到在线客户端的稳定性，需要设计一个能够实现热更新的bus-server。
要实现目标：
1. 实现服务的无缝升级
2. 保持已有的 WebRTC 会话不中断
3. 控制新老版本服务的平滑切换

### 服务工作模式
bus-server作为一个伴随 WebRTC 房间运行的协作处理服务，主要流程如下：
#### 🧩 1. 客户端连接到 LiveKit Server（LK Server）
- 客户端通过 WebRTC 协议建立与 LiveKit Server（LK Server）的连接。
- 每个客户端连接到指定的“房间”，建立媒体通道和信令连接。

#### 📨 2. LiveKit 通过 Webhook 通知bus-server有新客户端加入
- LK Server 配置了 Webhook 地址，所有房间事件（如 participant_joined）将发送到bus-server。
- 当有客户端加入房间时，LiveKit 会发出 webhook 请求，告知bus-server：
  - 房间 ID
  - 客户端（参与者） ID
  - 加入时间等元信息

#### 🔁 3. bus-server自动加入该房间，开始处理业务数据
- 在收到“客户端加入”事件后，bus-server主动以一个虚拟客户端或“机器人”身份加入该房间。
- 加入方式可能为：
  - 使用 LiveKit 提供的 API 创建 token + 加入房间
  - 使用专用 SDK 建立媒体/数据通道连接

- 加入后，bus-server将：
  - 持续监听客户端上报的业务数据（如 data track、信令、chat、传感器数据等）
  - 进行处理、记录或响应动作（如分析、转发、控制命令等）

#### 🔌 4. 客户端断开连接
- 客户端主动关闭，或因网络异常断线，或离开房间。

📤 5. LiveKit 通过 Webhook 通知你的服务客户端离开
- LK Server 触发 participant_disconnected webhook 事件。
- bus-server收到该 webhook 后，识别是哪一个房间的哪个客户端断开了。

### 实现滚动部署方案
#### 前提条件：
- 服务部署方式	Docker 容器
- 请求入口	所有 webhook 与 API 流量统一通过 Nginx
- Webhook 地址	固定不变，LiveKit 长期绑定
- 对外接口	HTTP API（REST 或 WebSocket）
- 部署平台	 Docker + Nginx
#### 架构图（单活部署）
```
                       WebRTC / 客户端
                             │
                             ▼
                        ┌──────────┐
                        │  NGINX   │
                        └────┬─────┘
                             │ proxy_pass
                             ▼
                   ┌────────────────────┐
                   │ webrtc-service:v1  │ ← 正在运行的容器
                   └────────────────────┘
```
#### 部署流程（覆盖式滚动）


### 
