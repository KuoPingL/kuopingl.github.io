---
layout: post
title:  "Android - 深入理解 Your First Coroutine"
date:   2022-08-08 16:15:07 +0800
categories: [coroutine, android, intermediate]
---

### 背景
這篇將帶領大家學習如何使用 Kotlin Coroutine 以及 Coroutine 的運作。

### Your first coroutine﻿
來源： [Coroutines basics﻿](https://kotlinlang.org/docs/coroutines-basics.html#your-first-coroutine) 

這是最簡單的 Coroutine 的使用：

```Kotlin
fun main() = runBlocking { // this: CoroutineScope
    launch { // launch a new coroutine and continue
        delay(1000L) // non-blocking delay for 1 second (default time unit is ms)
        println("World!") // print after delay
    }
    println("Hello") // main coroutine continues while a previous one is delayed
}
```
