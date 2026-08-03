---
title: Service在AndroidManifest的元素
---
在Android中，所有的Service都必须通过`<service>`标签在`AndroidManifest`中注册。

语法如下：

```xml
<service android:description="string resource"
         android:directBootAware=["true" | "false"]
         android:enabled=["true" | "false"]
         android:exported=["true" | "false"]
         android:foregroundServiceType=["camera" | "connectedDevice" |
                                        "dataSync" | "health" | "location" |
                                        "mediaPlayback" | "mediaProjection" |
                                        "microphone" | "phoneCall" |
                                        "remoteMessaging" | "shortService" |
                                        "specialUse" | "systemExempted"]
         android:icon="drawable resource"
         android:isolatedProcess=["true" | "false"]
         android:label="string resource"
         android:name="string"
         android:permission="string"
         android:process="string"
         android:stopWithTask=["true" | "false"]>
    ...

</service>
```

属性描述如下：

| 属性名                        |                    默认值                    | 描述                                                                                                                                                 |
| :---------------------------- | :------------------------------------------: | :--------------------------------------------------------------------------------------------------------------------------------------------------- |
| android:description           |                      -                       | 给用户提供服务的功能描述                                                                                                                             |
| android:directBootAware       |                    false                     | 是否可以在用户解锁设备之前运行                                                                                                                       |
| android:enabled               |                     true                     | 是否启用服务                                                                                                                                         |
| android:exported              | 如果有设置intent filter则为true，否则为false | 是否暴露给其他应用调用。如果是false就只能同一个应用或者具有相同用户 ID(29+已废弃）才可以使用这个服务。                                               |
| android:foregroundServiceType |                      -                       | 前台服务类型                                                                                                                                         |
| android:icon                  |                      -                       | 给用户提供服务的图标                                                                                                                                 |
| android:isolatedProcess       |                      -                       | 服务是否在与系统其余部分隔离的特殊进程下运行                                                                                                         |
| android:label                 |                      -                       | 给用户提供服务的名称。                                                                                                                               |
| android:name                  |                    *必填                     | 服务的类名。如果简写`.MyService`以句点开头，则将用`<manifest>`指定的软件包名称。                                                                     |
| android:permission            |                      -                       | 如果该值或者`<application>`的`permission`不为空，则启用该服务需要声明指定权限，否则`startService()`、`bindService()` 或 `stopService()` 将不起作用。 |
| android:process               |                      -                       | 运行服务的进程的名称。默认所有服务都是在应用的默认进程中执行，进程名和软件包包名一样。<br>的                                                         |
| android:stopWithTask          |                    false                     | 如果任务Task的所有Activity都移除，是否自动停止服务。                                                                                                 |
