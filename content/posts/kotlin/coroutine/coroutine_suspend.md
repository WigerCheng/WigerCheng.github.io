---
date: "2025-08-25T12:29:07+08:00"
title : 'Kotlin协程之函数的挂起'
---

**⚠️任何一个协程体或者挂起函数中都有一个隐含的Continuation 实例，编译器能够对这个实例进行正确传递。**

## 挂起函数

* 用`suspend` 关键字修饰的函数叫做**挂起函数**，挂起函数只能在协程体内或其他挂起函数内调用。
* **挂起函数可以调用任何函数，普通函数只能调用普通函数。**
* 协程的挂起就是程序执行流程发生异步调用时，当前调用流程的执行状态进入等待的状态。

```kotlin
/**
 * 挂起函数可以像普通函数一样同步返回。
 * 函数类型：suspend(Int)->Unit
 */
suspend fun suspendFunc01(a: Int) {
    return
}

/**
 * 挂起函数可以处理异步逻辑
 * 函数类型：suspend(String, String)->Unit
 */
suspend fun suspendFunc02(a: String, b: String): Int {  
    return suspendCoroutine<Int> { continuation ->  
        thread {  
            continuation.resumeWith(Result.success(5))  
        }  
    }
}
```

* 在`suspendFunc02`的定义中用`suspendCoroutine<T>`获取当前所在协程体的[`Continuation<T>`](../create_coroutine#continuation本体)的实例作为参数将挂起函数当成异步函数来处理，在1️⃣处执行`Continuatation.resumeWith`操作，因此协程调用suspendFunc02无法同步执行，会进入挂起状态，直到结果返回。

## 挂起点

* 在协程内部挂起函数的调用处被称为**挂起点**，挂起点如果出现异步调用，那么当前协程就被挂起，直到对应的Continuation的resume函数被调用才会恢复执行。

```kotlin
/**
 * 不会挂起的挂起函数
 */
suspend fun notSuspend() = suspendCoroutine<Int> { continuation -> continuation.resume(100) }
```

## 异步调用是否发生的条件

* 取决于resume函数与对应的挂起函数的调用**是否在相同的调用栈**上。
* 切换函数调用栈的方法
  1. 切换到其他线程上执行
  2. 不切换线程但在当前函数返回之后的某一个时刻再执行。

## CPS交换

* CPS交换 *(Continuation-Passing-Style Transformation)*，是通过传递Continuation来控制异步调用流程的。
* Kotlin 协程挂起时就将挂起点的信息保存到了Continuation对象中。Continuation 携带了协程继续执行所需要的上下文，恢复执行的时候只需要**执行它的恢复调用**并且把**需要的参数或者异常**传入即可。

### 用Java代码调用挂起函数

```java
// 比普通函数多了一个 Continuation 实例的参数。
Object result = K2Kt.notSuspend(new Continuation<>() {
            @NotNull
            @Override
            public CoroutineContext getContext() {
                return EmptyCoroutineContext.INSTANCE;
            }

            @Override
            public void resumeWith(@NotNull Object o) {
                
            }
        });
```

* `suspend() -> Int`  类型的函数`nonSuspend` 在Java 语言看来实际是`(Continuation<Integer>)->Object`类型，与回调函数相似。

### 返回值Object两种情况

#### 挂起函数同步返回

* 作为参数传入的Continuation的`resumeWith`不会被调用，**函数的实际返回值就是它作为挂起函数的返回值**。

#### 挂起函数挂起，执行异步逻辑

* 函数的实际返回值是一个**挂起标志**， 通过这个标志外部协程就可以知道该函数需要挂起等到异步逻辑执行。

### 用Kotlin反射调用挂起函数

```kotlin
val ref = ::notSuspend
val result = ref.call(object : Continuation<Int> {
    
    override val context: CoroutineContext
        get() = EmptyCoroutineContext

    override fun resumeWith(result: Result<Int>) {
        println("resumeWith: ${result.getOrNull()}")
    }

})
```
