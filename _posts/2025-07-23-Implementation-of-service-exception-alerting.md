---
layout: post
title: Implementation of service exception alerting.
categories: [Develop]
description: Implementation of service exception alerting.
keywords: server exception, alerting, feishu, monitor
comments: false
---
应用上线以后，为了确保能够为用户提供24小时可用的服务，需要用上各种手段来保证服务的可用性，例如典型的分布式集群部署。
实际项目中除了提高可用性，在服务异常时能够在第一时间内提醒开发人员和运维人员处理异常，恢复系统。

### 利用现有资源合理选择方案

在国内公司使用的通讯工具主要有三大阵营，飞书，钉钉，企业微信。功能其实大同小异，在具体选择时肯定是基于业务公司现有的资源来决策，
本次主要讨论方案就是基于飞书的场景。

### 方案设计
[服务异常] --> [飞书机器人@张三]
                    |
       [5分钟内无反馈 or 状态未恢复]
                    ↓
           [调用API打电话给张三]
