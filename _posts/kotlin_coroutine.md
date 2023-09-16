---
layout: post
title:  "kotlinx.coroutines - 從 Thread 到 Coroutine"
date:   2022-08-08 16:15:07 +0800
categories: [coroutine, android, intermediate]
---
# 簡介

在學編程時，我們第一個接觸到的編程模式是 **循序程式設計** ( Sequential Programming )。 循序程式的特色就是可以在 **單一執行緒** 中按我們所指定的順序來進行方法的調用，以確保一致的結果，就如同以下：

```Kotlin
fun sequentialProgramming() {
  doA()
  doB()
  doC()
}
```
無論 `sequentialProgramming()` 跑幾回結果都會是一致的，且方法的調用皆會是 `doA() -> doB() -> doC()` 。

但很多時候我們的程序需要與遠端的伺服器或是從本地端的資料庫溝通來讀取，包括照片、檔案、音樂 等等，資料。這些行為通常都需要等待伺服器或資料庫的回應，因此都需要相對 **長時間的等候**。

由於循序程式設計會確保方法們都在 *相同執行緒執行*，所以便會導致執行緒的堵塞。

```kotlin
doA() -> doLongB() - <等待 B 完成ing> -> doC()
```

在 Android 中，如果 `doLongB` 是在 Main Thread 進行的話，那就很有可能導致 **ANR** 了。

><br>
>
>In Android, when a response time takes more than 5 seconds or a Broadcast that is unable to be finished within 10 seconds , it is considered as an ANR (application not responding). [[來源](#Android_Official_ANR)]
>
> 在安卓中， ANR 會在 **畫面反應時間超過 5 秒** 或是 當 **Broadcast 無法在 10 秒內 完成** 時發生。
><br><br/>

**那我們要如何防範 ANR 的發生呢？**
答案就是讓需要長時間作業的方法在 **另一個執行緒** 執行。

## 何謂線程 / 執行緒？

><br>
>
>A **( kernel / OS ) thread** is a **basic unit of CPU utilization**. It is comprises of **Registers**, **Program Counter**, **ThreadID** and a **Stack**. [[來源](#OS_Concepts_10)]
>
>一個 **( 核心 ) 執行緒** 是 **CPU 的最小操作單位**。 它是由 **暫存器**、 **程序計數器**、 **執行緒 ID** 與 **堆壘 / 棧** 所組成。
><br><br/>



<center>
<img src = "/images/posts/jekyll/kotlin/coroutine/fromThreadToCoroutine/single_thread_vs_multi_thread_structure.png" style = "width:80%"/>

[來源](#OS_Concepts_10)
</center>

由示意圖可看出，執行緒除了各自擁有各自的記憶體外，也共享著 **執行程式** ( code ) 、 **全域變數** ( data ) 與 **資料庫** ( files )。

不過有趣的是，執行緒有兩種：

|執行緒|描述|英文|
|:--|:--|:--|
| **核心**<br>執行緒  | 這是以上描述的執行緒。 <br>這種執行緒並 **不受使用者控制**，而是 **由核心控制**。  |Kernel / OS Thread|
|**使用者**<br>執行緒   | 相對於核心執行緒，這種執行緒可由使用者控制。 <br>這也是我們在 **程式中所創建的執行緒物件**。  |User Thread|

至於這兩者之間的關係則需要看核心如何定義。

但因為現在硬體都有足夠的 **主記憶體** (RAM)，一般的核心系統中，每個使用者執行緒各自會有對應的核心執行緒，也就是 **1:1** 的關係。

接下來我們就看看如何創建核心執行緒吧。

### 執行緒的創建

在 Kotlin，我們可以很輕易地建立一個使用者執行緒：

```Kotlin
val thread = Thread();
```
當我們調用 ：

```Kotlin
thread.start()
```

的時候，程式便會經由 JNI 在核心空間中建立一個新的執行緒來運行我們所需要的事項。

這些都可以從 `start` 的實作中看出一二：

```Kotlin
public synchronized void start() {
    /// ...
    // add current Thread to a ThreadGroup
    group.add(this);

    try {
        nativeCreate(this, stackSize, daemon);
        started = true;
    } finally {

    try {
        if (!started) {
            group.threadStartFailed(this);
        }
    } catch (Throwable ignore) {
        /* do nothing. If start0 threw a Throwable then
          it will be passed up the call stack */
    }
}
```

當然，有興趣的同學可以試著去追蹤 JNI 的代碼，我就不加以贅述了。但你會發現最後會建立一個 `pthread` (POSIX Thread) 後，會調用 `java.lang.Thread` 中的 `run` 方法來執行事項。 [[medium](https://medium.com/kuos-funhouse/android-thread-101-part-i-what-is-a-thread-fb0e5194f33c)]

我們現在知道什麼是執行緒了，接下來就是如何讓它來做我們所需要的事了。

### 執行緒的行為

由於執行緒的行為是由 `run` 來定義：

```java
@Override
public void run() {
    if (target != null) {
        target.run();
    }
}
```

但細看之下，執行緒的行為其實是由 `target` 定義的。 而 `target` 則是一個 **Runnable**，一個 **單一抽象行為介面** ( SAM 或 Single Abstract Method )。

```java
@FunctionalInterface
public interface Runnable {
    public abstract void run();
}
```

所以我們想要定義執行緒的行為只需要重新定義 `run` 的實作即可。你有想到什麼辦法實作 `run` 呢？

其實我們有兩種選擇：

1. **實作** Runnable
2. **繼承** Thread

主要原因就是因為 **Thread** 本身也是實作 **Runnable**：

```java
public
class Thread implements Runnable {...}
```
如果我們將 **Thread** 、 **Runnable** 與 `target` 之間的關係用 UML 來表示，我們就可以得到：



這種設計模式便是結構性模式的 **代理人模式** 的一種。

#### 實作 Runnable

```kotlin
var name = "Jack";

val namePrinterRunnable = Runnable {
  println("Hi, I am $name")
}

Thread(namePrinterRunnable).start() // 建立並啟動 thread
```

或是以 Java

```java
String name = "Jack";

Runnable namePrinterRunnable = new Runnable() {
    @Override
    public void run() {
        for (int i = 0; i<100; i++) {
            System.out.println(String.format("Hi, I am %s", name));
        }
    }
};

Thread thread = new Thread(namePrinterRunnable);
thread.start();
```

#### 繼承 Thread

```kotlin
class MyThread: Thread() {
    override
    fun run() {
        println("This is a new Thread")
    }
}

MyThread().start()
```
Java

```Java
class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println("This is a new Thread");
    }
}

Thread myThread = new MyThread();
myThread.start();
```


#### 使用 Runnable 與 Thread 的時機

從這兩者的實作方式來看， **Runnable** 的實作依舊需要 **Thread** 來執行，所以實作 **Runnable** 的方式似乎比繼承 **Thread** 來的多一個步驟。

但是，若以一個物件的大小來看，在相同的實作下 **Runnable** 所包含的方法會比 **Thread** 少很多。 因此，實作 **Runnable** 會比繼承 **Thread** 來的有效率。

所以－般的情況， **Runnable** 似乎是更好的選擇。 除非我們想要只有單一職責的 **Thread**，我們就可以繼承 **Thread**。

當然，除了這兩種情況外， `java.util.concurrent` 中還有很多支援執行緒的類別，像是 **ExecutorService**、 **ThreadPoolExecutor** 和 **ForkJoinExecutor**。

這些類別都是實作 **Executor** 的類別：

```Java
public interface Executor {
    void execute(Runnable command);
}
```

### 多執行緒

#### 執行緒數量的上限

執行緒的數量其實是沒有上限的，因為要看你的程序被劃分多少記憶體。

但是，同一時間可被執行的執行緒數量則是被 CPU 有多少核心所限制。

譬如：

|Chip|核心數量|可同時處理的執行緒數量|
|:--|:--:|:--:|
|**Intel Core i3**   | 2 <br>+ Hyper Threading | 4  |
|**Intel Core i7**   | 4  | 8   |
|**Intel Core i5**   | 4  | 4  |

https://ark.intel.com/content/www/us/en/ark.html#@Processors


#### 池的運用



### 執行緒之間的溝通


```
## 執行緒的溝通

### join, yield, ...

### CPU 的自動優化 Memory Model
```

### Runnable

```java
@FunctionalInterface
public interface Runnable {
    public abstract void run();
}
```
**Runnable** 其實是一個 **單一抽象行為介面** ( SAM 或 Single Abstract Method )。

其使用方法也很簡單：

```Kotlin
val runnable = Runnable {
    println("Do Something")
}
val thread = Thread(runnable)
thread.start()

// -----terminal 結果-----
// Do Something
```



## Coroutine

### Structural Programming


### Reference
<a id="Android_Official_ANR"></a> 1. Google

<a id="OS_Concepts_10"></a> 2. Silberschatz A., Galvin, P.B. and Gagne, G., Operation System Concepts, 10<sup>ed</sup>, 2018.
