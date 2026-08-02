---
title: OkHttp原理
draft: true
---
OkHttp是安卓开发中最常且最多人使用的网络请求库之一。接下来将研究源码，研究OkHTTP库做了什么。

这是一个简单的同步Get请求。

1. 创建一个OkHttpClient实例。
2. 创建一个[[okhttp_request|Request]]。
3. 调用client.newCall(request)获得Call对象，并执行execute()。

```kotlin
  private val client = OkHttpClient()

  fun run() {
    val request = Request.Builder()
        .url("https://publicobject.com/helloworld.txt")
        .build()

    client.newCall(request).execute().use { response ->
      if (!response.isSuccessful) throw IOException("Unexpected code $response")

      for ((name, value) in response.headers) {
        println("$name: $value")
      }

      println(response.body!!.string())
    }
  }
```

## 1. newCall()

## 2. execute()

execute() 表示同步执行请求，会阻塞当前线程，直到响应返回或出现错误。它的注释是：Invokes the request immediately, and blocks until the response can be processed or is in error。

```kotlin
private val executed = AtomicBoolean()

private val timeout =  
  object : AsyncTimeout() {  
    override fun timedOut() {  
      this@RealCall.cancel()  
    }  
  }.apply {  
    timeout(client.callTimeoutMillis.toLong(), MILLISECONDS)  
  }

//...

override fun execute(): Response {  
  check(executed.compareAndSet(false, true)) { "Already Executed" }  
  
  timeout.enter()  
  callStart()  
  try {  
    client.dispatcher.executed(this)  
    return getResponseWithInterceptorChain()  
  } finally {  
    client.dispatcher.finished(this)  
  }  
}
```

1. 首先通过`executed` 判断当前 Call 是否已经执行过。如果已经执行，`compareAndSet(false, true)` 会失败，抛出 IllegalStateException("Already Executed")。
2. 通过`timeout.enter()` 启动超时控制。这个超时时间由 OkHttpClient 的 callTimeoutMillis 决定，超时后会执行 cancel()。
3. 执行`callStart()`，向eventListener的监听者分发开始请求的状态。
4. 
