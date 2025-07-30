---
layout: post
title: LJL-VPN Client Develop
categories: [Develop]
description: Develop the client implement of my vpn.
keywords: vpn, network, learning, golang
comments: false
---
首先实现客户端，作为客户端代理，需要监听在某个端口上，然后需要走vpn的网络请求通过该接口进行转发处理。

### 初始化项目
```
go mod init vpn-proxy
mkdir -p cmd internal/socks internal/protocol internal/crypto config
touch cmd/client.go cmd/server.go internal/socks/socks.go internal/protocol/protocol.go internal/crypto/crypto.go config/config.yaml README.md
```

### 客户端实现
#### 主函数实现
```
func main() {
	listenAddr := "127.0.0.1:1080"
	ln, err := net.Listen("tcp", listenAddr)
	if err != nil {
		log.Fatalf("Listen error: %v", err)
	}
	fmt.Println("SOCKS5 proxy listening on", listenAddr)
	for {
		conn, err := ln.Accept()
		if err != nil {
			log.Println("Accept error:", err)
			continue
		}
		go handleConn(conn)
	}
}
```
- 在本地 127.0.0.1:1080 端口启动 TCP 监听
- 当有新的 SOCKS5 客户端连接时，为每个连接启动独立的 goroutine 处理
- 使用 go handleConn(conn) 实现并发处理多个客户端

#### 连接处理函数（handleConn)
```
func handleConn(conn net.Conn) {
	defer conn.Close()
	if err := socks.Handshake(conn); err != nil {
		log.Println("SOCKS handshake failed:", err)
		return
	}
	targetAddr, err := socks.ParseRequest(conn)
	if err != nil {
		log.Println("ParseRequest failed:", err)
		return
	}
	log.Println("Request to", targetAddr)

	// 连接远程服务器
	serverConn, err := net.Dial("tcp", serverAddr)
	if err != nil {
		log.Println("Connect to server failed:", err)
		return
	}
	defer serverConn.Close()

	// 发送目标地址到服务器（简单协议：先发地址长度+地址字符串）
	addrBytes := []byte(targetAddr)
	addrLen := byte(len(addrBytes))
	_, err = serverConn.Write(append([]byte{addrLen}, addrBytes...))
	if err != nil {
		log.Println("Send target address failed:", err)
		return
	}

	// 建立双向转发
	go io.Copy(serverConn, conn)
	io.Copy(conn, serverConn)
}

```
##### 阶段一：SOCKS5 握手
```
if err := socks.Handshake(conn); err != nil {
    log.Println("SOCKS handshake failed:", err)
    return
}
```
- 与 SOCKS5 客户端进行协议握手
- 协商认证方式（目前只支持无认证）
- 如果握手失败，关闭连接

##### 阶段二：解析 SOCKS5 请求
```
targetAddr, err := socks.ParseRequest(conn)
if err != nil {
    log.Println("ParseRequest failed:", err)
    return
}
log.Println("Request to", targetAddr)
```
- 解析 SOCKS5 CONNECT 请求
- 提取目标服务器地址（如 www.google.com:80）
- 回复客户端连接成功
- 记录请求日志

##### 阶段三：连接远程服务器
```
serverConn, err := net.Dial("tcp", serverAddr)
if err != nil {
    log.Println("Connect to server failed:", err)
    return
}
defer serverConn.Close()
```
- 建立到远程代理服务器的 TCP 连接
- 如果连接失败，关闭本地连接

##### 阶段四：发送目标地址到服务器
```
addrBytes := []byte(targetAddr)
addrLen := byte(len(addrBytes))
_, err = serverConn.Write(append([]byte{addrLen}, addrBytes...))
```
- 将目标地址通过自定义协议发送给远程服务器
- 协议格式：[地址长度][地址字符串]
- 例如：[15][www.google.com:80]

##### 阶段五：建立双向数据转发
```
go io.Copy(serverConn, conn)  // 客户端 -> 服务器
io.Copy(conn, serverConn)     // 服务器 -> 客户端
```
- 使用 io.Copy 实现双向数据转发
- 第一个 io.Copy 在 goroutine 中运行，将本地客户端数据转发到远程服务器
- 第二个 io.Copy 在主线程中运行，将远程服务器数据转发回本地客户端

### 数据流向图
```
浏览器/应用 → [本地客户端:1080] → [远程服务器:9000] → [目标网站]
     ↑                                                      ↓
     ← [本地客户端:1080] ← [远程服务器:9000] ← [目标网站] ←
```

### 关键设计点
1. 并发处理
每个客户端连接都在独立的 goroutine 中处理
支持多个客户端同时使用代理
2. 简单协议
客户端与服务器间使用简单的自定义协议
格式：[地址长度][地址字符串]
后续可以扩展为更复杂的加密协议
3. 双向转发
使用 io.Copy 实现高效的数据转发
一个方向在 goroutine 中，避免阻塞
4. 错误处理
每个阶段都有错误检查
出错时及时关闭连接，避免资源泄露

### 使用场景
- 浏览器代理：设置浏览器 SOCKS5 代理为 127.0.0.1:1080
- 系统代理：设置系统全局 SOCKS5 代理
- 应用程序代理：支持 SOCKS5 的应用程序都可以使用
- 这个实现提供了基础的 SOCKS5 代理功能，后续可以在此基础上添加加密、认证、配置管理等功能。
