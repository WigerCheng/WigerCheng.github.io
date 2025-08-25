---
date: "2025-08-25T11:16:08+08:00"
title : 'Kotlin协程的上下文'
---

> 上下文：是指执行环境相关的通用数据资源的统一提供者。
---

- 协程的上下文数据结构特征实现与List、Map集合非常类似，协程的上下文作为一个集合，其元素类型是[Element](#element)。
- 协程上下文的内部实现实际上是一个单链表。

## Element
>
> An element of the CoroutineContext. An element of the coroutine context is a singleton context by itself.

```kotlin
    public interface Element : CoroutineContext {
        /**
         * A key of this coroutine context element.
         */
        public val key: Key<*>

        //....
    }
```

- Element接口中有一个属性`key`，其作用就是协程上下文这个集合中元素的"索引"，类似集合元素的下标，但和集合不同的是该“索引”是在数据里面的。

## 协程上下文元素的实现

---
除了`Element`接口外，还有一个`AbstractCoroutineContextElement`的抽象类，能更加方便实现协程上下文的元素。

```kotlin
public abstract class AbstractCoroutineContextElement(public override val key: Key<*>) : Element
```

### 协程名的实现

```kotlin
/**
 * CoroutineNameElement允许我们为协程绑定一个名字
 */
class CoroutineNameElement(name: String) : AbstractCoroutineContextElement(KEY){
    companion object KEY : CoroutineContext.Key<CoroutineNameElement>
}
```

### 协程异常处理器的实现

```kotlin
/**
 * CoroutineExceptionHandlerElement允许我们在启动协程的时候安装一个统一的异常处理器
 */
class CoroutineExceptionHandlerElement(val onErrorAction: (Throwable) -> Unit) : AbstractCoroutineContextElement(KEY) {
    companion object KEY : CoroutineContext.Key<CoroutineExceptionHandlerElement>

    fun onError(error: Throwable) {
        error.printStackTrace()
        onErrorAction(error)
    }
}
```

## 协程上下文的使用

```kotlin
fun main() {
    var coroutineContext: CoroutineContext = EmptyCoroutineContext
    //类似集合一样有两种添加上下文的方式
    //1️⃣
    //coroutineContext += CoroutineNameElement("co-01")
    //coroutineContext += CoroutineExceptionHandlerElement {
    //    println("error::::${it}")
    //}
    //2️⃣
    coroutineContext += CoroutineNameElement("co-01") + CoroutineExceptionHandlerElement {
        println("error::::${it}")
    }
    suspend {
        //在协程内部获取上下文
        val nameElement: CoroutineNameElement? = coroutineContext[CoroutineNameElement]
        println("In Coroutine [${nameElement?.name}].")
        100 / 0
    }.startCoroutine(object : Continuation<Int> {
        override val context: CoroutineContext
            get() = coroutineContext

        override fun resumeWith(result: Result<Int>) {
            result.onFailure {
                context[CoroutineExceptionHandlerElement]?.onError(it)
            }
        }
    })
}
```

- 通过resumeWith里的`context.get()`或者协程体里的`coroutineContext.get()`获取协程上下文元素。因为有可能没设置CoroutineExceptionHandlerElement、CoroutineNameElement...所以`get()`返回的结果是可空类型。
- `context.get(key)`中key是协程上下文的key。
