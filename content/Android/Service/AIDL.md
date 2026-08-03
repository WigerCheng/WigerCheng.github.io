---
title: Android 接口定义语言 (AIDL)
draft: true
---

1. 在需要使用aidl的module的build.gralde文件中添加`buildFeatures.aidl = true`来启用aidl。
2. 通过右键新建New->AIDL->AIDL File创建aidl文件
```java
// IRemoteService.aidl  
package io.wiger.myviewapplication;  
  
// Declare any non-default types here with import statements  
  
interface IRemoteService {  
    /**  
     * Demonstrates some basic types that you can use as parameters     
     * and return values in AIDL.     
     * 
     * */   
     void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat, double aDouble, String aString);  
}
```
3. AIDL支持的数据类型
	- Java的原始类型byte，short，char，int，long，float，double和boolean
	- 任何类型的数组
	- String
	- CharSequence
	- List
	- Map
4. 