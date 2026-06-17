---
title: Service
tags:
  - Android
draft: true
---

Android系统提供了Service的系统组件可以让App在后台执行一些长时间的操作。如数据同步，资源下载，后台听歌，文件I/O同步等等吗还能和contentProvider进行交互。

## 使用Service

要创建Service其实不复杂，首先要自定义一个Service类继承Service类或者它现有的子类，然后在[[Service  Manifest Element|`AndroidManifest`]]文件中注册我们自定义的Service，最后根据自己的需要去处理服务生命周期的方法。

## Service 生命周期

Service的生命周期根据创建的方式可分为两种。

| 创建方式     | 生命周期                                                                                                                                                                                                       |
| ------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| startService | 当其他组件调用`startService()`时，系统将自动创建该服务，该服务将无限期运行。自身调用`stopSelf()`或者其他组件调用`stopService()`来停止服务。**服务停止后，系统会将其销毁。**                                    |
| bindService  | 当其他组件（客户端）调用`bindService()`，系统将自动创建该服务。客户端通过调用`unbindService()`来关闭连接。多个客户端可以绑定到统一服务，直至所有客户端都解除绑定时，系统将会销毁该服务。**服务无需自行停止。** |

> [!info] 提示
> 如果服务已经通过`startService()`启动后，也可以同时调用`bindService()`来绑定。此刻调用`stopService()`或`stopSelf()`将不会停止和销毁服务。

![Service Lifecycle](https://developer.android.com/static/images/service_lifecycle.png)

### onCreate

在服务首次创建时系统将会执行onCreate方法，此刻可以做一些初始化的设置。

### onStartCommand

如果是通过`startService`启动的服务，系统将会执行`onStartCommand`方法。

## Service 类型

[[Foreign Service|前台服务]]
[[Background Service|后台服务]]
