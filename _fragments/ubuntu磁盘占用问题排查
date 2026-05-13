---
layout: fragment
title: Ubuntu系统磁盘异常占用问题排查
tags: [disk, ubuntu]
description: docker 日志 占用大量磁盘空间
keywords: docker, logs, disk
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

查找大文件: 
`sudo du -ah -x / | sort -rh | head -n 20`
如果是docker容器日志: 例如: 

ubuntu@ip-192-168-100-92:/$ sudo du -ah -x / | sort -rh | head -n 20


57G     /var/lib/docker/containers/fa4dc5d38150a9d5d1dad44c130fb3067a6ea98cedbef894d4b57afba0e11843/fa4dc5d38150a9d5d1dad44c130fb3067a6ea98cedbef894d4b57afba0e11843-json.log
57G     /var/lib/docker/containers/fa4dc5d38150a9d5d1dad44c130fb3067a6ea98cedbef894d4b57afba0e11843
19G     /var/lib/docker/containers/2c6a27b6eeee9b7e300fdeba7aa0edbd75271c21c76dc171247b1204af071821/2c6a27b6eeee9b7e300fdeba7aa0edbd75271c21c76dc171247b1204af071821-json.log
19G     /var/lib/docker/containers/2c6a27b6eeee9b7e300fdeba7aa0edbd75271c21c76dc171247b1204af071821

清理命令


`sudo sh -c 'truncate -s 0 /var/lib/docker/containers/*/*-json.log'`

修改方法,限制单个容器的用量

```
services:
  your_service_name:
    image: ...
    # 👇 修改日志限制 👇
    logging:
      driver: "json-file"
      options:
        max-size: "2g"    # 单个日志文件最大 2GB
        max-file: "5"     # 轮转保留最多 5 个文件（包含当前正在写的1个 + 历史被打包的4个）
```

全局限制: 
`sudo nano /etc/docker/daemon.json`
添加内容
```
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "2g",
    "max-file": "5"
  }
}
```
