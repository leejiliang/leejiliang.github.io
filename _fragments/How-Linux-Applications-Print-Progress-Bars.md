---
layout: fragment
title: How Linux Applications Print Progress Bars
tags: [golang]
description: 小case
keywords: golang, 进度条
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

### 实现思路
1. 计算出当前已完成任务的百分比
2. 设定进度条的总长度
3. 计算出已完成部分占进度条的总长度
4. 通过 fmt.Printf 以 \r 回车符开头，回到本行行首；先打印 doneLen 个“█”，再打印剩余空白“ ”；后面接显示百分比和当前完成计数

```
func printProgress(done, total int) {
	percent := float64(done) / float64(total)
	barLen := 50
	doneLen := int(percent * float64(barLen))
	fmt.Printf("\r[%s%s] %.2f%% (%d/%d)", strings.Repeat("█", doneLen), strings.Repeat(" ", barLen-doneLen), percent*100, done, total)
}
```
