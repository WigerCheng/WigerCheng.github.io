---
date: "2025-08-25T11:10:46+08:00"
title : 'Kotlin协程的构造'
---
## 协程的创建

```kotlin
import kotlin.coroutines.*

val continuation = suspend {
    println("5")
    5
}.createCoroutine(object : Continuation<Int> {
    override val context: CoroutineContext
        get() = EmptyCoroutineContext

    override fun resumeWith(result: Result<Int>) {
        println("Coroutine End:$result")
    }
})
```

* 标准库提供了一个`createCoroutine`函数，可通过它来创建协程，但这个协程并不会立即执行。
* 标准库也提供了`startCoroutine`函数，可以创建协程后就立即让它开始执行。

```kotlin
//createCoroutine 函数声明
fun <T> (suspend () -> T).createCoroutine(
    completion: Continuation<T>
): Continuation<Unit>

//startCoroutine 函数声明
fun <T> (suspend () -> T).startCoroutine(  
    completion: Continuation<T>  
)
```

* `suspend () -> T` 是 *createCoroutine* 函数的Receiver，是协程的**执行体** ，称为**协程体**。
* 参数`completion`在协程执行完后调用，是协程的**完成回调**。
* `createCoroutine`函数的返回值是一个[Continuation本体|Continuation](#continuation本体)对象，`startCoroutine`函数的返回值是Unit。

## 协程的启动

* 通过调用`continuation.resume(Unit)`之后，协程体会立即开始执行。
* `continuation.resumeWithException(Exception("Custom Exception"))`

## 协程体的Receiver

```kotlin
fun <R, T> (suspend R.() -> T).createCoroutine(  
    receiver: R,  
    completion: Continuation<T>  
): Continuation<Unit>

fun <R, T> (suspend R.() -> T).startCoroutine(  
    receiver: R,  
    completion: Continuation<T>  
)
```

* `suspend R.() -> T`多了一个Receiver类型R。R可以为协程体提供一个作用域，即可以在**协程体中直接使用作用域内提供的函数或者状态等**。

* ⬇️封装的launchCoroutine方法间接实现声明带有Receiver的Lambda表达式的语法。

```kotlin
fun <R, T> launchCoroutine(receiver: R, block: suspend R.() -> T) {  
    block.startCoroutine(receiver, object : Continuation<T> {  
        override fun resumeWith(result: Result<T>) {  
            println("Coroutine End:$result")  
        }  
  
        override val context: CoroutineContext  
            get() = EmptyCoroutineContext  
    })  
}
```

* ⬇️例子

```kotlin
//1.创建一个模拟一个生成器协程的作用域
//@RestrictsSuspension 
class ProducerScope<T> {  
    suspend fun produce(value: T) {  
        delay(1000L)  
        println("value = $value")  
    }  
}  
  
fun callLaunchCoroutine() {  
    launchCoroutine(ProducerScope<Int>()) {  
     //2.添加了作用域ProducerScope作为Receiver，即可以直接调用ProducerScope的produce函数。
        println("In Coroutine")  
        produce(1024)//作用域的挂起函数
        delay(2000L)//外部挂起函数（若启用了@RestrictsSuspension注解，这句将会报错）
        produce(2048)  
    }  
}
```

* `@RestrictsSuspension`注解的作用下，协程体就无法调用外部的挂起函数。

## Continuation本体

* continuation的类名类似于`<FileName>Kt$<FunctionName>$continuation$1`
  * `<FileName>` 指代的是代码所在的文件名
  * `<FunctionName>` 指代的是代码所在的函数名
* Continuation本体是一个匿名内部类，编译器在编译后生成一个匿名内部类，这个类继承自`SuspendLambda`类，同时又是`Continuation`接口的实现类。
