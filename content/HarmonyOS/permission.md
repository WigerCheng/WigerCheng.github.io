---
title: HarmonyOs权限管理
draft: true
---
在HarmonyOS中，系统根据应用的[APL等级](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/app-permission-mgmt-overview#%E6%9D%83%E9%99%90%E6%9C%BA%E5%88%B6%E4%B8%AD%E7%9A%84%E5%9F%BA%E6%9C%AC%E6%A6%82%E5%BF%B5)设置进程域和数据域标签。通过访问控制机制，限制数据访问范围，从机制上减少数据泄露风险。

不同APL等级的应用可申请的权限等级不同，通过严格的分层权限保护抵御恶意攻击，确保系统安全可靠。除了系统资源（如通讯录）、系统能力（如访问摄像头、麦克风）受不同的应用权限保护外，还有一些内核中的资源（如可执行匿名内存的申请）也受到权限保护。这类权限被称为KernelPermission。
