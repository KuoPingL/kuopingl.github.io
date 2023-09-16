---
layout: post
title:  "Android : Thread 與 Coroutine"
date:   2022-08-08 16:15:07 +0800
categories: [thread, coroutine, android, intermediate]
---

### 背景

相信大家對 **線程** 或 **Thread** 並不陌生。畢竟寫 app 時都會使用到線程：
- 與使用者互動時 使用 **Main Thread**
- 長時間的運算使用時 使用 **自定線程**

但你知道線程可分成兩種嗎？

**User Thread** (使用者線程)
- 使用者可對此線程隨時開始、中斷、結束

**System / OS / Native Thread** 系統線程
- 此線程由系統控制，使用者無法對其作出任何







### 參考

1. [Kotlin Coroutines: 入門概念 Coroutine vs Thread](https://medium.com/gogolook-tech/kotlin-coroutines-%E5%85%A5%E9%96%80%E6%A6%82%E5%BF%B5-coroutine-vs-thread-e7d112b0d8ba)
(這個寫得真不錯)
