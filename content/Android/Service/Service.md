Android的Service是一个可以在后台执行长时间操作的应用组件。当Service启动后，用户切换到其他应用后它可以在后台继续运行一段时间。

## 使用Service


## Service 生命周期

Service的生命周期根据创建的方式可分为两种。

| 创建方式         | 生命周期                                                                                                                      |
| ------------ | ------------------------------------------------------------------------------------------------------------------------- |
| startService | 当其他组件调用`startService()`时，系统将自动创建该服务，该服务将无限期运行。自身调用`stopSelf()`或者其他组件调用`stopService()`来停止服务。**服务停止后，系统会将其销毁。**             |
| bindService  | 当其他组件（客户端）调用`bindService()`，系统将自动创建该服务。客户端通过调用`unbindService()`来关闭连接。多个客户端可以绑定到统一服务，直至所有客户端都解除绑定时，系统将会销毁该服务。**服务无需自行停止。** |

> [!info] 提示
> 
> 如果服务已经通过`startService()`启动后，也可以同时调用`bindService()`来绑定。此刻调用`stopService()`或`stopSelf()`将不会停止和销毁服务。

![Service Lifecycle](https://developer.android.com/static/images/service_lifecycle.png)

## Service 类型

[[Foreign Service|前台服务]]
[[Background Service|后台服务]]
