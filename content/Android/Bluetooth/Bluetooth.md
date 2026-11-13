---
draft: true
tags:
  - Android/Bluetooth
---
如果要App使用蓝牙，必须首先在`Android Manifest`中声明蓝牙的权限。

## 蓝牙权限

```xml
<uses-feature  
    android:name="android.hardware.bluetooth"  
    android:required="true" />  
<uses-feature  
    android:name="android.hardware.bluetooth_le"  
    android:required="true" />  
  
<uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />  
<uses-permission  
    android:name="android.permission.BLUETOOTH_SCAN"  
    android:usesPermissionFlags="neverForLocation"  
    tools:targetApi="31" />  
<uses-permission  
    android:name="android.permission.BLUETOOTH"  
    android:maxSdkVersion="30" />  
<uses-permission  
    android:name="android.permission.ACCESS_FINE_LOCATION"  
    android:maxSdkVersion="30" />
```