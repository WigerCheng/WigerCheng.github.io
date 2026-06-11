---
draft: true
---
在Android中，Task（任务）是指用户在应用中进行某项操作时，所涉及的一系列 Activity 的集合。这些Activity按照打开的顺序存在一个栈中，我们称之为返回栈。

绝大多数App的任务是通过设备主屏幕点击应用icon或者shortcut快捷方式启动。如果这个APP不存在任务则新建一个新的任务，并把MainActivity作为这个任务的根Activity放入返回栈中。

![](https://developer.android.com/static/images/fundamentals/diagram_backstack.png)

