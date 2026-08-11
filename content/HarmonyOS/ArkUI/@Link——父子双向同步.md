---
draft: true
tags:
  - HarmonyOS/ArkUI
---

- @Link装饰的变量与其父组件中的数据源共享相同的值。
- @Link子组件从父组件初始化@State的语法为Comp({ aLink: this.aState })。同样Comp({aLink: $aState})也支持。
- @Link的类型必须和父组件完全相同