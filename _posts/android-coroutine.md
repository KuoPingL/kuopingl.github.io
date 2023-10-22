---
layout: post
title:  "Kotlinx.Coroutine - 啟發"
date:   2022-08-08 16:15:07 +0800
categories: [coroutine, android, intermediate]
---
### 閱讀前須知

想要真正理解 Coroutine 你需要對 **Thread** 有一定的了解。


### 背景

有寫過或看過 C 語言的人應該知道 `goto` 這個詞。

// TODO : 講解一下 goto 的缺點與來點綴 Coroutine 的必要性
-

雖然會讀這篇文章的多多少少都知道 Kotlin.Coroutine 這個 Library。但有多少人知道 Coroutine 是如何開始的呢？

我們將透過 [Roman Elizarov](https://www.linkedin.com/in/relizarov/) 在 [Hydra 2019](https://www.youtube.com/watch?v=Mj5P47F6nJg) 的 "**Roman Elizarov — Structured concurrency**" 的演講中暸解一二。也順便探討一下 Coroutine 與 Flow 是如何運用，以及運作原理。


### 啟發

#### async/await
Coroutine 的設計是啟發於由 **C#** 所推出的 **async/await** 關鍵詞。

`async/await` 的使用方法如下：

<center>
<img src = "/images/posts/jekyll/kotlin/coroutine/howItStarted/cs_async_await_example.png" style = "width:80%"/>
</center>


1. `RequestToken` 會先運行，但 **await** 會確保 `token` 有值之後再調用 `CreatePost`。
2. 同樣地， **await** 也會確保 `CreatePost` 完成後再調用 `ProcessPost`。

<details>
<summary> await 的補充</summary>

**await** 可被視為一個 **suspension point** ( 暫緩點 )。
暫緩的時候就會將 **await** 所對應的方法的 **狀態** 擱置一旁。等到方法完成後再回到 **重量級線程** 並往下執行。

因為 `async/await` 的 **non-blocking** 特性，與一般線程相比，他算是 **輕量級線程**。
</details>

<br>




由於大家對 **async/await** 如浪潮般的好評，所以 JS、 Python、 Dart 、 TypeScript 、 Rust 及 C++ 也陸續推出以 **async/await** 為概念的功能。

自然， Kotlin 也不落人後。 初始版也就如此誕生了。

#### Initial Prototype

初始版是使用 **Kotlin DSL** 來設計的。

除了定義了回傳 `future` 的 `async/await` 還定義了會回傳 `sequence` 的 `generate/yield`。

<center>
<img src = "/images/posts/jekyll/kotlin/coroutine/howItStarted/coroutine_init_dsl.png" style = "width:80%"/>
</center>

與

<center>
<img src = "/images/posts/jekyll/kotlin/coroutine/howItStarted/coroutine_init_dsl_2.png" style = "width:80%"/>
</center>

<br>


雖說這個版本與 C# 的十分相像，但還有兩點可以改進：
1. 竟然我們想要讓 `requestToken` 與 `createPost` 皆為 suspending fucntion， 那是不是可以將 `await()` 給省掉呢？
2. 如果我們知道 `postItem` 需要得到一個 `future`，那為何不直接將 `postItem` 設為 suspending function 呢？

也就如此， `suspend` 修飾詞誕生了：

<center>
<img src = "/images/posts/jekyll/kotlin/coroutine/howItStarted/coroutine_init_suspended.png" style = "width:80%"/>
</center>

<br>

這樣一來，程式碼的可讀性就大大增加了。

但由於無法清晰看見 `async/await` 字眼， Kotlin 團隊依舊對此不滿。 他們擔心使用者是否會有不同的聲音。

這是直到他們看到 Gopher (Golang) 的寫法：

<center>
<img src = "/images/posts/jekyll/kotlin/coroutine/howItStarted/gopher_coroutine.png" style = "width:80%"/>
</center>

<br>

以及使用者對這個設計的嘉許後， Kotlin 團隊才放下心中的大石頭，邁向下一個目標。

#### Prototyping Library

竟然 Gopher 已經有類似的設計，Kotlin 團隊在建立函式庫雛形時就以 Gopher 視為模板。

在使用 DSL 建立 Coroutine 時，也誕生了以下新詞：
- `delay`：一個與 `sleep` 相似但不會阻塞線程的方法。
- `launch`：一個原本叫 `go` 的 **Coroutine Builder**
- `runBlocking`： 一個原名叫 `mainBlocking` 的 **Coroutine Builder**。 主要功能是讓 `main()` 方法可以使用。

經過不懈的努力， Coroutine 函式庫終於可以做到任何 Coroutine 語言可做的事了。 他們還將 Go 使用 `chan` (channel) 運行的費伯南西係數用 Coroutine 寫出來：

<center>
<img src = "/images/posts/jekyll/kotlin/coroutine/howItStarted/fib_in_go.png" style = "width:80%"/>

<img src = "/images/posts/jekyll/kotlin/coroutine/howItStarted/fib_in_coroutine.png" style = "width:80%"/>
</center>

<br>

這些都解決了，那下一步又是什麼呢？

#### Thread-Bound UI Programming

不像 Go 只是給後端運用， Kotlin 是個廣泛運用的語言。
因此， Kotlin Coroutine 必須要有能力指定在 UI 線程上建立 Coroutine。

因此 `launchUI` 就誕生了：

```Kotlin
  launchUI {
    try {
      // a suspended function to asynchronously makes request
      val result = makeRequest()
      // display result on UI
      display(result)
    } catch (exception: Throwable) {
      // process exception on UI
    }
  }
```

**但 `display` 要如何確定會在主線程發生呢？**

因此 **CoroutineContext** 的概念就出現了：


```Kotlin
// UI 為 Coroutine Context
launch(UI) {
  try {
    // a suspended function to asynchronously makes request
    val result = makeRequest()
    // display result on UI
    display(result)
  } catch (exception: Throwable) {
    // process exception on UI
  }
}
```

增加了 **Coroutine Context** 參數便可將內部的 **ThreadPool** 指名說要使用 UI 的 Coroutine Context。

但為了要將結果傳回 主線程，我們就需要搭配 High order function。 這裡我們可以使用 `run` ：


```Kotlin
// UI 為 Coroutine Context
// 更正確的 UI 是 CoroutineDispatcher
launch(UI) {
  try {
    // a suspended function to asynchronously makes request
    val result = makeRequest()
    // display result on UI
    run {
      display(result)
    }
  } catch (exception: Throwable) {
    // process exception on UI
  }
}
```

><br>
>
> **PS**：`run` 其實是不需要的，至少現在的版本是不需要的。 因為 `launch(UI)` 會確保內部除了 Suspend Function 外，都會從 UI Thread 進行。
> <br><br/>







### What is Coroutine ?



### How to use Coroutine ?




### Understand Coroutine Structure



### How does Coroutine works ?


### Flow


### Channel


### RxJava

### 額外話題

#### Thread 算重量級？

當我第一次聽到 Coroutine 是 **輕量級的 Thread** 時，我真的聽得一頭霧水。 何為重量級？ 何為輕量級？ 難道他們可以被秤出來的？

後來經過一段時間的學習，


---
何為輕量級？
-

在編程中，量級的輕重取決於運行時所需要消耗的資源，包括：
- 記憶體
- 運算時間
- 其他資源

相對於 **Process** 而言， **Thread** 就被視為「輕量級進程」。
這是因為 **Thread** 所使用的資源與進程相比少了許多。

而這裡 **Thread** 與 **async-await** 相比，所花的資源卻是比較多的。

至於為什麼，之後會一一解釋。

---
