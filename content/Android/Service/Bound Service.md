---
title: Android 绑定服务
draft: true
tags:
  - Android/Service
---
绑定服务(BoundService)是C/S接口的服务器。当activity绑定到服务后，它可以处理发送请求、接收响应已经执行进程间通信（IPC）。

## 绑定服务特别的生命周期

如果是通过`bindService`启动的服务，

如果这个服务没有与任何客户端进行绑定的话
