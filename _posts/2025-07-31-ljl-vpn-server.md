---
layout: post
title: LJL-VPN Server Develop
categories: [Develop]
description: Develop the server implement of my vpn.
keywords: vpn, network, learning, golang
comments: false
---
server端的主要是启动监听，接受来自客户端的连接，然后请求客户端的目标资源，最后实现双向转发数据.

### 主函数实现
```
func main() {
	listenAddr := "0.0.0.0:9000"
	ln, err := net.Listen("tcp", listenAddr)
	if err != nil {
		log.Fatalf("Server listen error: %v", err)
	}
	fmt.Println("VPN proxy server listening on", listenAddr)
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
指定监听地址和端口 0.0.0.0 表示监听所有网卡，端口为9000
- 创建一个TCP监听器，等待客户端的连接，如果监听失败，打印错误日志并退出程序。
- 进入无限循环，不断接受新的连接，接受连接失败，打印错误并等待下一个连接。
- 接受连接成功，启动一个goroutine 并发处理，避免阻塞主循环。

### 处理连接函数
```
func handleConn(conn net.Conn) {
	defer conn.Close()
	// 读取目标地址
	addrLen := make([]byte, 1)
	if _, err := io.ReadFull(conn, addrLen); err != nil {
		log.Println("Read addrLen failed:", err)
		return
	}
	addrBytes := make([]byte, addrLen[0])
	if _, err := io.ReadFull(conn, addrBytes); err != nil {
		log.Println("Read addrBytes failed:", err)
		return
	}
	targetAddr := string(addrBytes)
	log.Println("Proxy to", targetAddr)

	// 连接目标网站
	targetConn, err := net.Dial("tcp", targetAddr)
	if err != nil {
		log.Println("Connect to target failed:", err)
		return
	}
	defer targetConn.Close()

	// 建立双向转发
	go io.Copy(targetConn, conn)
	io.Copy(conn, targetConn)
}
```

#### 1. 处理函数结束时自动关闭客户端连接，防止资源泄露
```
defer conn.Close()
```

#### 2. 读取目标地址长度
```
   addrLen := make([]byte, 1)
   if _, err := io.ReadFull(conn, addrLen); err != nil {
   	log.Println("Read addrLen failed:", err)
   	return
   }
```
- 先读取1字节，表示后面目标地址的长度（如13 表示后面有13字节的地址字符串）

#### 3. 读取目标地址内容
```
   addrBytes := make([]byte, addrLen[0])
   if _, err := io.ReadFull(conn, addrBytes); err != nil {
   	log.Println("Read addrBytes failed:", err)
   	return
   }
```
- 根据上一步读取到的长度，再读取目标地址本身（如 "www.baidu.com:80"）

#### 4. 解析目标地址
```
   targetAddr := string(addrBytes)
   log.Println("Proxy to", targetAddr)
```
- 将字节数组转为字符串，得到目标服务器的地址，并打印日志。

#### 5. 连接目标服务器
```
   targetConn, err := net.Dial("tcp", targetAddr)
   if err != nil {
   	log.Println("Connect to target failed:", err)
   	return
   }
   defer targetConn.Close()
```
- 作为代理，主动连接目标服务器。如果失败，记录日志并返回。
- 连接成功后，确保函数结束时关闭目标服务器连接。

#### 6. 双向转发数据
```
   go io.Copy(targetConn, conn)
   io.Copy(conn, targetConn)
```
- 启动一个 goroutine，把客户端发来的数据转发到目标服务器。
- 主 goroutine 把目标服务器返回的数据转发回客户端。
- io.Copy 会一直转发，直到一方关闭连接。

### 总结
- 这个服务端程序本质上是一个“TCP 透明代理”。
- 客户端连接到服务端，发送目标地址，服务端帮它连接目标服务器，并转发数据，实现“VPN”或“代理”效果。
- 代码结构简洁，利用 goroutine 实现高并发，io.Copy 实现高效的数据转发。
