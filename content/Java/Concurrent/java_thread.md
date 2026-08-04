---
tags:
  - Java/Concurrent
title: Java 线程
draft: true
---
## Thread.start

```java
    public void start() {
        synchronized (this) {
            // zero status corresponds to state "NEW".
            if (holder.threadStatus != 0)
                throw new IllegalThreadStateException();
            start0();
        }
    }
```

>[!warning] 
>
>不能两次启动THread，否则会出现IllegalThreadStateException异常。

