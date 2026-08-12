---
date: 2022-04-19T00:00:00Z
title: Android 架构演进：手把手带你理解并落地 MVP 模式
tags:
  - Android
---

# Android 架构演进：手把手带你理解并落地 MVP 模式

随着 Android 应用规模的不断扩大，传统的 MVC 模式（Model-View-Controller）在 Android 开发中逐渐显得力不从心。由于 Android 系统的特殊设计，Activity/Fragment 常常需要同时兼顾布局生命周期与业务数据请求，导致代码越写越臃肿，变成了难以维护的“万能类”（Fat Activity）。

为了解决**关注点分离（Separation of Concerns）**与**便捷的单元测试**，**Model-View-Presenter (MVP)** 架构模式应运而生。本文将基于谷歌官方经典的 `todo-mvp` 示例，带你一步步落地并吃透 MVP 模式。

---

## 一、 什么是 MVP？

**MVP 的全称为 Model-View-Presenter，它将应用拆分为三个核心部分：Model 提供数据与核心业务，View 负责 UI 渲染与用户交互，Presenter 负责两者之间的调度逻辑。**

```
+------------------+         +-------------------+         +-----------------+
|                  |  Calls  |                   |  Calls  |                 |
|      View        |-------->|     Presenter     |-------->|     Model       |
| (Activity/Frag)  |<--------|                   |<--------| (Repo/Data)     |
|                  | Updates |                   | Events  |                 |
+------------------+         +-------------------+         +-----------------+
```

根据维基百科的模型描述：
- **Model（模型）**：定义用户界面所需要显示的资料模型，包含相关的业务数据逻辑（如网络请求、数据库操作、本地缓存等）。
- **View（视图）**：呈现用户界面的终端，用以表现来自 Model 的数据，并将用户触发的命令事件路由传给 Presenter 对齐处理。在 Android 中，View 通常由 Activity、Fragment 或自定义 View 来充当。
- **Presenter（呈递者）**：View 与 Model 之间的纽带。它负责检索 Model 获取数据，并将获取的数据经过格式转换（数据清洗）后交由 View 进行呈现；同时，它也消费 View 传递过来的用户事件。

### MVP 与 MVC 的本质区别：被动视图 (Passive View)
在经典的 MVC 中，Model 的改变可以直接通知 View 更新。而在 MVP 中，**View 与 Model 彻底解耦，它们之间不存在任何直接联系**，所有的交互都必须通过 Presenter 作为中介来进行。此时的 View 变成了纯粹的“被动视图”——只负责听从 Presenter 的指令改变 UI，或者将手指触摸事件直接汇报给 Presenter。

![MVP 流程图示](android-mvp-flow.png)

---

## 二、 谷歌官方 MVP 设计规范与实现

以下代码参考谷歌官方架构样板 [To-DoApp的todo-mvp-kotlin分支](https://github.com/android/architecture-samples/tree/todo-mvp-kotlin)。

### 1. 定义基础基类 `BaseView` 和 `BasePresenter`

为了规范化契约绑定，我们定义了两个基础接口：

```kotlin
interface BaseView<T> {
    // 每个 View 都持有一个 presenter 的引用，便于交互
    var presenter: T
}
```

```kotlin
interface BasePresenter {
    // 规定 Presenter 启动的时机，通常在 View 的 onResume 阶段调用以加载初始数据
    fun start()
}
```

---

### 2. 编写功能契约：Contract 机制

谷歌官方 MVP 推荐使用一个 **Contract（契约类）** 接口将特定页面下的 `View` 和 `Presenter` 接口放在一起。这样做的好处是**一目了然**，任何人点开契约类都能立刻明白这个页面有哪些 UI 展现形式，支持哪些用户操作。

以 **Task 详情页** 为例，用户能执行 4 种操作，分别是编辑 Task、删除 Task、改变 Task 完成状态（已完成/未完成）。我们定义 `TaskDetailContract` 如下：

```kotlin
interface TaskDetailContract {

    interface Presenter : BasePresenter {
        // 用户操作：编辑 Task
        fun editTask()
        // 用户操作：删除 Task
        fun deleteTask()
        // 用户操作：完成 Task
        fun completeTask()
        // 用户操作：激活 Task
        fun activateTask()
    }

    interface View : BaseView<Presenter> {
        val isActive: Boolean
        
        // UI 更新：是否显示加载中进度条
        fun setLoadingIndicator(active: Boolean)
        // UI 更新：显示找不到 Task 错误
        fun showMissingTask()
        // UI 更新：Task为空时，隐藏标题
        fun hideTitle()
        // UI 更新：显示标题
        fun showTitle(title: String)
        // UI 更新：Task为空时，隐藏描述
        fun hideDescription()
        // UI 更新：显示描述
        fun showDescription(description: String)
        // UI 更新：显示Task的完成状态
        fun showCompletionStatus(complete: Boolean)
        // UI 更新：跳转到编辑Task的页面
        fun showEditTask(taskId: String)
        // UI 更新：执行Task删除后的UI跳转逻辑
        fun showTaskDeleted()
        // UI 更新：显示Task已完成的提示界面
        fun showTaskMarkedComplete()
        // UI 更新：显示Task已激活的提示界面
        fun showTaskMarkedActive()
    }
}
```

---

## 三、 业务代码交互分析

在实际的业务中，Activity 充当装载容器，具体实现逻辑写在 Fragment 中，Presenter 负责桥接。

### 1. Activity 层：组件装配与绑定
`TaskDetailActivity` 作为入口容器，在 `onCreate` 时实例化 `TaskDetailFragment`（实现了 `View` 接口）和 `TaskDetailPresenter`，并将它们绑定在一起。

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.taskdetail_act)

    // 1. 获取或创建 TaskDetailFragment 实例
    val taskDetailFragment = supportFragmentManager
        .findFragmentById(R.id.contentFrame) as TaskDetailFragment?
        ?: TaskDetailFragment.newInstance(taskId).also {
            replaceFragmentInActivity(it, R.id.contentFrame)
        }

    // 2. 实例化 Presenter，并将 View 作为参数传过去完成依赖注入
    TaskDetailPresenter(
        taskId,
        Injection.provideTasksRepository(applicationContext),
        taskDetailFragment
    )
}
```

### 2. Presenter 层：接收 View 并反向绑定
`TaskDetailPresenter` 接收了实现了 `View` 接口的 Fragment。在构造方法中，通过 `taskDetailView.presenter = this` 将自身引用回传给 View，完成了**双向绑定**。

```kotlin
class TaskDetailPresenter(
    private val taskId: String,
    private val tasksRepository: TasksRepository,
    private val taskDetailView: TaskDetailContract.View
) : TaskDetailContract.Presenter {

    init {
        // 完成双向引用的绑定
        taskDetailView.presenter = this
    }

    override fun start() {
        openTask()
    }

    private fun openTask() {
        if (taskId.isEmpty()) {
            taskDetailView.showMissingTask()
            return
        }
        taskDetailView.setLoadingIndicator(true)
        tasksRepository.getTask(taskId, object : TasksDataSource.GetTaskCallback {
            override fun onTaskLoaded(task: Task) {
                if (!taskDetailView.isActive) return
                taskDetailView.setLoadingIndicator(false)
                showTask(task)
            }

            override fun onDataNotAvailable() {
                if (!taskDetailView.isActive) return
                taskDetailView.setLoadingIndicator(false)
                taskDetailView.showMissingTask()
            }
        })
    }
    // ...
}
```

### 3. Fragment（View）层：生命周期对齐与命令传递
`TaskDetailFragment` 实现了 `View` 接口，并持有 `presenter` 引用。在生命周期 `onResume()` 中，调用 `presenter.start()` 触发 Presenter 去 Model 请求数据。

```kotlin
class TaskDetailFragment : Fragment(), TaskDetailContract.View {
    
    override lateinit var presenter: TaskDetailContract.Presenter

    override fun onResume() {
        super.onResume()
        // 对齐生命周期，启动数据加载
        presenter.start()
    }
    
    // ...
}
```

---

## 四、 核心交互流程实战：以“删除 Task”为例

为了让你彻底看清 MVP 的数据流向，我们以**用户点击删除菜单**为例：

### 步骤 1：View 接收用户点击，路由事件给 Presenter
在 `TaskDetailFragment` 的菜单点击事件中，不去执行任何具体的删除代码，而是直接汇报给 Presenter：

```kotlin
override fun onOptionsItemSelected(item: MenuItem): Boolean {
    val deletePressed = item.itemId == R.id.menu_delete
    if (deletePressed) {
        // 告诉主事者：用户想删掉这个任务
        presenter.deleteTask()
    }
    return deletePressed
}
```

### 步骤 2：Presenter 负责业务调度，并指示 View 改变 UI
`TaskDetailPresenter` 接收到命令，开始进行条件判定并指挥 Model 执行物理删除，然后回调 View 刷新界面：

```kotlin
override fun deleteTask() {
    if (taskId.isEmpty()) {
        taskDetailView.showMissingTask()
        return
    }
    // 1. 调用 Model 层（Repository）执行物理删除
    tasksRepository.deleteTask(taskId)
    // 2. 指挥 View 层更新 UI 状态
    taskDetailView.showTaskDeleted()
}
```

### 步骤 3：View 执行最纯粹的 UI 逻辑
`TaskDetailFragment` 听从指挥，执行纯粹的 UI 关机或提示操作：

```kotlin
override fun showMissingTask() {
    detailTitle.text = ""
    detailDescription.text = getString(R.string.no_data)
}

override fun showTaskDeleted() {
    // 纯粹的 UI 关闭逻辑
    activity?.finish()
}
```

---

## 五、 MVP 的优缺点与设计反思

### 1. 优势
- **极佳的可测试性**：因为 Presenter 层完全不依赖任何 Android 视图框架类（即没有 `import android.view.*` 和 `R`），我们可以直接对 Presenter 编写纯 JUnit 单元测试，不需要依赖 Robolectric 或真机设备。
- **高内聚低耦合**：业务逻辑收拢到 Presenter，UI 的视觉操作收拢到 Fragment。即便以后将布局从传统 XML 迁移到 Jetpack Compose，我们也只需要替换 View 层实现，Presenter 层的业务逻辑不用做任何改动。

### 2. 缺点
- **接口文件爆炸**：每个页面都需要写一个 Contract 契约类和多个内部接口，前期开发会伴随着大量的样板接口代码。
- **Presenter 容易膨胀**：如果页面非常复杂，Presenter 同样会面对代码量过大的问题。
- **内存泄漏隐患**：由于 Presenter 持有了 View（如 Fragment）的引用，而网络数据请求等后台线程可能在页面销毁时仍未返回，极易导致 Activity 无法被回收。因此在实际的工业开发中，我们需要在 Presenter 中增加类似 `detachView()` 的注销机制，或者利用 Jetpack Lifecycle 自动绑定销毁行为。
