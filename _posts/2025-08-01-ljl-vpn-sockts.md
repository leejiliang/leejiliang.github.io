---
layout: post
title: LJL-VPN Sockt Develop
categories: [Develop]
description: Develop the socket implement of my vpn.
keywords: vpn, network, learning, golang
comments: false
---
socks.go 文件是这个代理项目的协议适配层，它的主要作用是：提供标准化的 SOCKS5 代理接口；解析和处理 SOCKS5 协议数据；让各种应用能够无缝使用这个代理服务；实现协议转换，连接用户应用和代理系统

### 握手处理函数
```
func Handshake(conn net.Conn) error {
    buf := make([]byte, 258)
    // 读取VER, NMETHODS, METHODS
    // 读取版本和方法数量
    if _, err := io.ReadFull(conn, buf[:2]); err != nil {
        return err
    }
    ver, nMethods := buf[0], buf[1]
    // 版本检查
    if ver != socks5Version {
        return errors.New("unsupported socks version")
    }
    // 读取认证方法
    if _, err := io.ReadFull(conn, buf[:nMethods]); err != nil {
        return err
    }
    // 只支持无认证
    _, err := conn.Write([]byte{socks5Version, 0x00})
    return err
}
```
- SOCKS5 握手协议格式：
```
客户端发送: [VER][NMETHODS][METHODS...]
服务端回复: [VER][METHOD]
```
#### 详细解析
- 读取版本和方法数量
- 版本检查
- 读取认证方法
- 回复选择无认证

### 请求解析函数
```
func ParseRequest(conn net.Conn) (string, error) {
    buf := make([]byte, 263)
    // 读取前4字节
    if _, err := io.ReadFull(conn, buf[:4]); err != nil {
        return "", err
    }
    ver, cmd, _, atyp := buf[0], buf[1], buf[2], buf[3]
    if ver != socks5Version || cmd != socks5CmdConnect {
        return "", errors.New("unsupported command or version")
    }
    var addr string
    switch atyp {
    case socks5AddrTypeIPv4:
        // 处理 IPv4 地址
    case socks5AddrTypeDomain:
        // 处理域名地址
    default:
        return "", errors.New("unsupported address type")
    }
    // 回复客户端连接成功
    _, _ = conn.Write([]byte{
        socks5Version, 0x00, 0x00, 0x01,
        0, 0, 0, 0, 0, 0,
    })
    return addr, nil
}
```
SOCKS5 请求协议格式：
```
客户端发送: [VER][CMD][RSV][ATYP][DST.ADDR][DST.PORT]
服务端回复: [VER][REP][RSV][ATYP][BND.ADDR][BND.PORT]
```
#### 详细解析
- 读取请求头
- 验证版本和命令，确保时SOCKET5和CONNECT命令
- 根据地址类型解析目标地址
  - IPV4地址处理
    IPV4格式：[4字节IP][2字节端口]
     - 读取 6 字节：4 字节 IP + 2 字节端口
     - 使用 binary.BigEndian.Uint16 解析大端序端口号
     - 组合成 "IP:端口" 格式
  - 域名地址处理
    域名格式： [1字节域名长度][域名][2字节端口]
      - 先读取 1 字节获取域名长度
      - 再读取域名 + 端口（域名长度 + 2 字节）
      - 解析域名和端口，组合成 "域名:端口" 格式
- 回复连接成功
  回复格式： [0x05][0x00][0x00][0x01][0x00][0x00][0x00][0x00][0x00][0x00]
  - 0x05：SOCKS5 版本
  - 0x00：成功状态
  - 0x00：保留字段
  - 0x01：IPv4 地址类型
  - 0x00,0x00,0x00,0x00：绑定地址（这里用 0.0.0.0）
  - 0x00,0x00：绑定端口（这里用 0）

### 设计特点
- 简化实现：只支持无认证和 CONNECT 命令
- 地址支持：支持 IPv4 和域名，不支持 IPv6
- 错误处理：对不支持的版本、命令、地址类型返回错误
- 标准兼容：完全按照 RFC 1928 标准实现
  
这个 SOCKS5 实现为整个代理系统提供了标准的代理接口，让各种应用都能无缝使用这个代理服务。
