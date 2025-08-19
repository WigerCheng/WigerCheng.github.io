---
categories:
- Android
- Architecture
date: "2022-04-19T00:00:00Z"
tags:
- Develop
title: Android MVP架构
---

## 什么是MVP？

**MVP的全称为Model-View-Presenter，Model提供数据，View负责显示，Controller/Presenter负责逻辑的处理。**

来自[维基百科](https://zh.wikipedia.org/wiki/Model-view-presenter)的模型描述
> Model-View-Presenter (MVP) 是用户界面设计模式的一种，被广泛用于便捷自动化单元测试和在呈现逻辑中改良分离关注点（separation of concerns）。
>- Model 定义用户界面所需要被显示的资料模型，一个模型包含着相关的业务逻辑。
>- View 视图为呈现用户界面的终端，用以表现来自 Model 的资料，和用户命令路由再经过 Presenter 对事件处理后的资料。
>- Presenter 包含着组件的事件处理，负责检索 Model 获取资料，和将获取的资料经过格式转换与 View 进行沟通。

![MVP图示](android-mvp-flow.png)

## Android 实现
代码参考谷歌官方的Demo。[To-DoApp的todo-mvp-kotlin分支](https://github.com/android/architecture-samples/tree/todo-mvp-kotlin)

### 1. 定义`BaseView`和`BasePresenter`
- 其中`BaseView`持有一个`presenter`成员。实现`BasePresenter`要实现`start()`方法。

```kotlin
interface BaseView<T> {
    var presenter: T
}
```
```kotlin
interface BasePresenter {
    fun start()
}
```

### 2.明确功能需求，编写功能级的View和Presenter
以Task详情页为例，用户能执行4种操作，分别是编辑Task、删除Task、改变Task完成状态（已完成、未完成）。根据上面的需求，我们Task详情页的Presenter代码如下。
```kotlin
interface Presenter : BasePresenter {
    // 编辑Task
    fun editTask()
    // 删除Task
    fun deleteTask()
    // 完成Task
    fun completeTask()
    // 激活Task
    fun activateTask()
}
```

```kotlin
interface View : BaseView<Presenter> {
    val isActive: Boolean
    // 是否显示加载中
    fun setLoadingIndicator(active: Boolean)
    // 显示找不到Task
    fun showMissingTask()
    // Task为空时，隐藏标题
    fun hideTitle()
    // 显示并显示标题
    fun showTitle(title: String)
    // Task为空时，隐藏描述
    fun hideDescription()
    // 显示并显示描述
    fun showDescription(description: String)
    // 显示Task的完成状态
    fun showCompletionStatus(complete: Boolean)
    // 跳转编辑Task的Activity
    fun showEditTask(taskId: String)
    // 执行Task删除后的逻辑
    fun showTaskDeleted()
    // 显示Task已完成的界面
    fun showTaskMarkedComplete()
    // 显示Task已已激活的界面
    fun showTaskMarkedActive()
}
```

### 3.业务代码分析
- `TaskDetailActivity`是容器，具体实现逻辑在`TaskDetailFragment`。
  
1. 在`TaskDetailActivity`里，将`TaskDetailPresenter`和`TaskDetailFragment`*（已实现`View`接口）* 绑定在一起，如何绑定见下文2。
    ```kotlin
        override fun onCreate(savedInstanceState: Bundle?) {
            ...
            //取得TaskDetailFragment实例
            val taskDetailFragment = ...
            TaskDetailPresenter(..., taskDetailFragment)
            ...
        }
    ```
2. `TaskDetailPresenter`接收实现了`View`接口的UI，即`TaskDetailFragment`，并实现`Presenter`接口，在构造方法的时候通过调用 `taskDetailView.presenter = this` 绑定在一起。
    ```kotlin
    class TaskDetailPresenter(
            ...
            private val taskDetailView: TaskDetailContract.View
    ) : TaskDetailContract.Presenter {
    
        init {
            taskDetailView.presenter = this
        }
    
        ...
        //实现TaskDetailContract.Presenter的方法
        ...
    }
    ```
3. `TaskDetailFragment`实现`View`接口，并持有`presenter`实例和Presenter进行交互。在`onResume`的时候，调用Presenter的`start()`方法启动Presenter，开始交互。
    ```kotlin
    class TaskDetailFragment : Fragment(), TaskDetailContract.View {
        
        override lateinit var presenter: TaskDetailContract.Presenter
    
        override fun onResume() {
            super.onResume()
            presenter.start()
        }
    
        ...
        //实现TaskDetailContract.View的方法和属性
        ...
    }
    ```
4. `TaskDetailFragment`调用`TaskDetailPresenter`的方法执行业务逻辑，`TaskDetailPresenter`调用`TaskDetailContract.View`的方法，让`TaskDetailFragment`更新页面UI。
    - 调用`presenter.deleteTask()`删除Task
    ```kotlin
        override fun onOptionsItemSelected(item: MenuItem): Boolean {
            val deletePressed = item.itemId == R.id.menu_delete
            if (deletePressed) presenter.deleteTask()
            return deletePressed
        }
    ```
    - presenter根据实际情况调用`View`的方法更新UI。如果taskId为空，则调用`showMissingTask()`显示没有Task，否则执行删除逻辑并调用`showTaskDeleted()`显示删除成功。
    ``` kotlin
        override fun deleteTask() {
            if (taskId.isEmpty()) {
                taskDetailView.showMissingTask()
                return
            }
            tasksRepository.deleteTask(taskId)
            taskDetailView.showTaskDeleted()
        }
    ```
    - `TaskDetailFragment`执行最终显示UI的方法。
    ```kotlin
        override fun showMissingTask() {
            detailTitle.text = ""
            detailDescription.text = getString(R.string.no_data)
        }
    
        override fun showTaskDeleted() {
            activity?.finish()
        }
    ```