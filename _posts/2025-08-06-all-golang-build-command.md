---
layout: post
title: A note on Golang cross-compilation and packaging commands.
categories: [Command]
description: 
keywords: linux，command, golang
comments: false
---
✅ 常见平台示例
1. 编译为 Linux（64位）
```bash
GOOS=linux GOARCH=amd64 go build -o app-linux-amd64
```
2. 编译为 Windows（64位，.exe 文件）
```
GOOS=windows GOARCH=amd64 go build -o app-windows-amd64.exe
```
3. 编译为 macOS（64位 Intel）
```
GOOS=darwin GOARCH=amd64 go build -o app-darwin-amd64
```
4. 编译为 macOS（M1/M2 ARM）
```
GOOS=darwin GOARCH=arm64 go build -o app-darwin-arm64
```
5. 编译为 Linux ARM（如树莓派）
```
GOOS=linux GOARCH=arm GOARM=7 go build -o app-linux-armv7
```
6. 编译为freebsd
```
GOOS=freebsd GOARCH=amd64 go build -o app-freebsd main.go
```
