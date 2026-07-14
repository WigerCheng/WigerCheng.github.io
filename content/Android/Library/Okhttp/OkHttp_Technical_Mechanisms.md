---
title: OkHttp原理
draft: true
---
OkHttp是安卓开发中最常且最多人使用的网络请求库之一。接下来将研究源码，研究OkHTTP库做了什么。

这是一个简单的同步Get请求。

1. 创建一个OkHttpClient实例。
2. 创建一个[[OkHttp_Request|Request]]。
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

