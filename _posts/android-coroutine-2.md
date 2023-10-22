---
layout: post
title:  Kotlinx.Coroutine - 從 CoroutineContext 來暸解 Coroutine
date:   2022-08-08 16:15:07 +0800
categories: [coroutine, android, intermediate]
---
# 序文
有使用 **MVVM** 架構寫過 Android 的想必知道我們在 **ViewModel** 中是如何創建 **Job** 的吧？

我們從一開始：
```kotlin
private val viewModelJob = SupervisorJob()
private val uiScope = CoroutineScope(Dispatchers.Main + viewModelJob)
```

到後來通過 `implementation "androidx.lifecycle.lifecycle-viewmodel-ktx$lifecycle_version"` 讓我們可以用 `viewModelScope` 來取代。這讓我們便利了不少。

但你知道這 `viewModelScope` 到底是什麼？可以做什麼？會有什麼作用？

這篇我將從 **Coroutine** 的底層 **CoroutineContext** 來聊聊什麼是 **Coroutine**。

# CoroutineContext
**CoroutineContext** 其實是一個介面：

```kotlin
@SinceKotlin("1.3")
public interface CoroutineContext {
    public operator fun <E : Element> get(key: Key<E>): E?

    public fun <R> fold(initial: R, operation: (R, Element) -> R): R

    public operator fun plus(context: CoroutineContext): CoroutineContext =
        if (context === EmptyCoroutineContext) this else // fast path -- avoid lambda creation
            context.fold(this) { acc, element -> // acc is this
                // removed is the R value after element.key has removed
                val removed = acc.minusKey(element.key)
                if (removed === EmptyCoroutineContext) element else {
                    // make sure interceptor is always last in the context (and thus is fast to get when present)

                    val interceptor = removed[ContinuationInterceptor]
                    if (interceptor == null) CombinedContext(removed, element) else {
                        val left = removed.minusKey(ContinuationInterceptor)
                        if (left === EmptyCoroutineContext) CombinedContext(element, interceptor) else
                            CombinedContext(CombinedContext(left, element), interceptor)
                    }
                }
            }

    public fun minusKey(key: Key<*>): CoroutineContext
    public interface Key<E : Element>
}
```

我們可以從 `plus` 方法中看出 **CoroutineContext** 可以通過 **CombinedContext** 的方法互相連結。
另外，如果其中有 **ContinuationInterceptor** 就會被放到最後。

所以我們可以將 **CoroutineContext** 看作是一個 **LinkedList** 中的 **Node**。

## EmptyCoroutineContext
**CoroutineContext** 最基本的類別叫做 **EmptyCoroutineContext** ：

```kotlin
@SinceKotlin("1.3")
public object EmptyCoroutineContext : CoroutineContext, Serializable {
    private const val serialVersionUID: Long = 0
    private fun readResolve(): Any = EmptyCoroutineContext

    public override fun <E : Element> get(key: Key<E>): E? = null
    public override fun <R> fold(initial: R, operation: (R, Element) -> R): R = initial
    public override fun plus(context: CoroutineContext): CoroutineContext = context
    public override fun minusKey(key: Key<*>): CoroutineContext = this
    public override fun hashCode(): Int = 0
    public override fun toString(): String = "EmptyCoroutineContext"
}
```

**EmptyCoroutineContext** 相當於 **CoroutineContext** 的 `null`。
所以當我們進行 `plus` 或 `fold` 時，它只會回傳帶入的 `context` 或 `initial`。

## CombinedContext
**CombinedContext** 也是另一個直接繼承 **CoroutineContext** 的類別。 但是它有以下主要參數：
- `element`：這是 **CombinedContext** 主要的 **CoroutineContext**，一個 **CoroutineContext.Element**。也可以看作是 **LinkedList** 中的 **Node** 的 `value`。
- `left`：對應著 `element`，這個值相當於 **LinkedList** 的 **Node** 中的 `node` 的存在。它指向著下一個 **CoroutineContext**。


```kotlin
@SinceKotlin("1.3")
internal class CombinedContext(
    private val left: CoroutineContext,
    private val element: Element
) : CoroutineContext, Serializable {

    override fun <E : Element> get(key: Key<E>): E? {
        var cur = this
        while (true) {
            cur.element[key]?.let { return it }
            val next = cur.left
            if (next is CombinedContext) {
                cur = next
            } else {
                return next[key]
            }
        }
    }

    public override fun <R> fold(initial: R, operation: (R, Element) -> R): R =
        operation(left.fold(initial, operation), element)

    public override fun minusKey(key: Key<*>): CoroutineContext {
        element[key]?.let { return left }
        val newLeft = left.minusKey(key)
        return when {
            newLeft === left -> this
            newLeft === EmptyCoroutineContext -> element
            else -> CombinedContext(newLeft, element)
        }
    }

    private fun size(): Int {
        var cur = this
        var size = 2
        while (true) {
            cur = cur.left as? CombinedContext ?: return size
            size++
        }
    }

    private fun contains(element: Element): Boolean =
        get(element.key) == element

    private fun containsAll(context: CombinedContext): Boolean {
        var cur = context
        while (true) {
            if (!contains(cur.element)) return false
            val next = cur.left
            if (next is CombinedContext) {
                cur = next
            } else {
                return contains(next as Element)
            }
        }
    }

    override fun equals(other: Any?): Boolean =
        this === other || other is CombinedContext && other.size() == size() && other.containsAll(this)

    override fun hashCode(): Int = left.hashCode() + element.hashCode()

    override fun toString(): String =
        "[" + fold("") { acc, element ->
            if (acc.isEmpty()) element.toString() else "$acc, $element"
        } + "]"

    private fun writeReplace(): Any {
        val n = size()
        val elements = arrayOfNulls<CoroutineContext>(n)
        var index = 0
        fold(Unit) { _, element -> elements[index++] = element }
        check(index == n)
        @Suppress("UNCHECKED_CAST")
        return Serialized(elements as Array<CoroutineContext>)
    }

    private class Serialized(val elements: Array<CoroutineContext>) : Serializable {
        companion object {
            private const val serialVersionUID: Long = 0L
        }

        private fun readResolve(): Any = elements.fold(EmptyCoroutineContext, CoroutineContext::plus)
    }
}
```

我們從源碼中可以看到他的方法中有以下行為：
- `get`： 這方法會從 **CombinedContext** 中從頭到尾試圖尋找擁有 `key` 的 **CoroutineContext**。
  順序：頭 ➞ 尾
- `fold`：這方法會從尾部開始調用 `fold` 值到頭部。
  順序：尾 ➞ 頭
- `writeReplace`：將 **CoroutineContext** 從尾到頭存放到 **Array<CoroutineContext>** 中並將它包裹成 **Serialized** 進行回傳。
  當進行讀取這個 **Serialized** 時，他會將這 **Array** 中的 **CoroutineContext** 通過 `plus` 的方法連接起來。

## 結論
我們從這裡看得出來 **CoroutineContext** 其實可以被視為 **LinkedList** 中的 **Node**。 而 **CoroutineContext** 介面只負責如何將其他 **CoroutineContext** 連接、移除與尋找。

接下來我們要看看 **CoroutineContext** 中其中一個最重要的子介面， **CoroutineContext.Key** 和 **CoroutineContext.Element**。

# CoroutineContext.Element

**Element** 是最基本的 **CoroutineContext** node：

```kotlin
public interface Element : CoroutineContext {
    public val key: Key<*>

    public override operator fun <E : Element> get(key: Key<E>): E? =
        @Suppress("UNCHECKED_CAST")
        if (this.key == key) this as E else null

    public override fun <R> fold(initial: R, operation: (R, Element) -> R): R =
        operation(initial, this)

    public override fun minusKey(key: Key<*>): CoroutineContext =
        if (this.key == key) EmptyCoroutineContext else this
}
```

其中繼承 **Element** 的最基本的類別有：
- **AbstractCoroutineContextElement**
- **ThreadContextElement**
- **ContinuationInterceptor**
- **Job**


## ThreadContextElement

><br>
>
> **ThreadContextElement** 是用來建構 **Thread** 與 **CoroutineContext** 之間的關係。
><br>

```kotlin
public interface ThreadContextElement<S> : CoroutineContext.Element {
    public fun updateThreadContext(context: CoroutineContext): S
    public fun restoreThreadContext(context: CoroutineContext, oldState: S)
}
```

我們可以從 **ThreadLocalElement** 中看到 **ThreadContextElement** 的基本實作。

### ThreadLocalElement

**ThreadLocalElement** 是一個帶有 `value` 與 **ThreadLocal** 的 **ThreadContextElement**：

```kotlin
internal class ThreadLocalElement<T>(
    private val value: T,
    private val threadLocal: ThreadLocal<T>
) : ThreadContextElement<T> {
    override val key: CoroutineContext.Key<*> = ThreadLocalKey(threadLocal)

    override fun updateThreadContext(context: CoroutineContext): T {
        val oldState = threadLocal.get()
        threadLocal.set(value)
        return oldState
    }

    override fun restoreThreadContext(context: CoroutineContext, oldState: T) {
        threadLocal.set(oldState)
    }

    // this method is overridden to perform value comparison (==) on key
    override fun minusKey(key: CoroutineContext.Key<*>): CoroutineContext {
        return if (this.key == key) EmptyCoroutineContext else this
    }

    // this method is overridden to perform value comparison (==) on key
    public override operator fun <E : CoroutineContext.Element> get(key: CoroutineContext.Key<E>): E? =
        @Suppress("UNCHECKED_CAST")
        if (this.key == key) this as E else null

    override fun toString(): String = "ThreadLocal(value=$value, threadLocal = $threadLocal)"
}
```

我們可以從 `updateThreadContext` 與 `restoreThreadContext` 中看到， 他們的目的是更新 **ThreadLocal** 中的值 或是 state。

而 **ThreadLocal** 本身的作用就是保存一個要給當下線程的一個值。 也就是說，每個線程都會各自擁有不同 copy 的值。

這個值可以是 **User ID** 或 **Transaction ID** 等等。

官方的範例中則是定義了線程 ID：

```java
public class ThreadId {
  // Atomic integer containing the next thread ID to be assigned
  private static final AtomicInteger nextId = new AtomicInteger(0);

  // Thread local variable containing each thread's ID
  private static final ThreadLocal<Integer> threadId =
      new ThreadLocal<Integer>() {
          @Override protected Integer initialValue() {
              return nextId.getAndIncrement();
      }
  };

  // Returns the current thread's unique ID, assigning it if necessary
  public static int get() {
      return threadId.get();
  }
}
```

每一個線程都可以通過 `get` 取得增加 1 的 ThreadID 。

但我們從 **ThreadLocalElement** 其實沒有特別將 **CoroutineContext** 與 **Thread** 做任何的聯繫。

所以我們再看另一個 **ThreadContextElement** 的實作者： **CoroutineId**。

### CoroutineId

```kotlin
internal data class CoroutineId(
    val id: Long
) : ThreadContextElement<String>, AbstractCoroutineContextElement(CoroutineId)
```

這裡的 `AbstractCoroutineContextElement(CoroutineId)` 中的 **CoroutineId** 其實會直接取得 **CoroutineId** 中的 **Key** ：

```kotlin
companion object Key : CoroutineContext.Key<CoroutineId>
```

接下來我們來看看 `updateThreadContext` 與 `restoreThreadContext` 的實作吧。

`restoreThreadContext` 的實作很簡單明瞭，他的作用只是更新當下 **Thread** 的名字：

```kotlin
override fun restoreThreadContext(context: CoroutineContext, oldState: String) {
    Thread.currentThread().name = oldState
}
```

相對的， `updateThreadContext` 則比較複雜，但其作用其實也很簡單。 他的主要目的是將目前 **Coroutine** 的名稱連接在 Thread 名稱的後面。 最後形成擁有 **{oldname} @{coroutineName}#{id}** 架構的新名子：

```kotlin
override fun updateThreadContext(context: CoroutineContext): String {
    val coroutineName = context[CoroutineName]?.name ?: "coroutine"
    val currentThread = Thread.currentThread()
    val oldName = currentThread.name
    var lastIndex = oldName.lastIndexOf(DEBUG_THREAD_NAME_SEPARATOR)
    if (lastIndex < 0) lastIndex = oldName.length
    currentThread.name = buildString(lastIndex + coroutineName.length + 10) {
        append(oldName.substring(0, lastIndex))
        append(DEBUG_THREAD_NAME_SEPARATOR)
        append(coroutineName)
        append('#')
        append(id)
    }
    return oldName
}
```
講完 **ThreadContextElement** 我們來看看 **AbstractCoroutineContextElement** 有什麼用吧。

## AbstractCoroutineContextElement

**AbstractCoroutineContextElement** 主要就是強調 **Key** 的需求：

```kotlin
@SinceKotlin("1.3")
public abstract class AbstractCoroutineContextElement(public override val key: Key<*>) : Element
```

所以我們知道，只要是繼承 **AbstractCoroutineContextElement** 就必定會有 **Key** 的存在。另外，由於 **CoroutineContext** 必須要有 **Key** 才可以被放入 **CombinedContext** 中，所以繼承 **AbstractCoroutineContextElement** 的也必須能成為 **CoroutineContext** 一員。

我們其實之前看到的 **CoroutineId** 便是其中一個。其他的繼承者包括：
- **CoroutineName**
  這是一個擁有 `val name: String` 的類別。 這個 `name` 是由我們設定，主要是用在 debug 的時候。
- **CoroutineDispatcher**
  這是全部 **Dispatchers** 所繼承的基本類別。
- **AndroidExceptionPreHandler**
- **YieldContext**

### CoroutineName
這是一個擁有 `val name: String` 的類別。 這個 `name` 是由我們設定，主要是用在 debug 的時候：

```kotlin
public data class CoroutineName(
    /**
     * User-defined coroutine name.
     */
    val name: String
) : AbstractCoroutineContextElement(CoroutineName)
```

### CoroutineDispatcher

**CoroutineDispatcher** 是我們所使用的 **Dispatchers** 的 base class。

```kotlin
public abstract class CoroutineDispatcher :
    AbstractCoroutineContextElement(ContinuationInterceptor), ContinuationInterceptor {

      @ExperimentalStdlibApi
      public companion object Key : AbstractCoroutineContextKey<ContinuationInterceptor, CoroutineDispatcher>(
          ContinuationInterceptor,
          { it as? CoroutineDispatcher })
}
```

這裡我們要注意到 **CoroutineDispatcher** 中的 **Key** 其實是一個 **AbstractCoroutineContextKey**。 通過 **AbstractCoroutineContextKey** 當我們在調用 `someContext[CoroutineDispatcher]` 時，他會將 **CoroutineDispatcher** 與 **ContinuationInterceptor** 都被認定為是合法的 **Element**。

但是由於他的 `safeCast` lambda 只會檢查是否為 **CoroutineDispatcher**，所以 `someContext[CoroutineDispatcher]` 會回傳 **CoroutineDispatcher?** 。

接下來我們來看看 **CoroutineDispatcher** 的其他行為吧。

```kotlin
public open fun isDispatchNeeded(context: CoroutineContext): Boolean = true
```

`isDispatchNeeded` 是用來告知目前的 **CoroutineContext** 是否需要進行 `dispatch` 行為。預設為 `true`。
若此方法回傳為 `false`，那它就會在目前的線程中進行。 並且為了避免外溢，一個 `event loop` 會被創造出來。這之後會談。

通過覆寫這個方法，我們可以進行 `dispatch` 效能的優化，減少不必要的 `dispatch`。我們之後會在 [MainCoroutineDispatcher.immediate] 看到。

另外，這個方法最好不要拋出 **Exception**，以防導致不明的結果。

```kotlin
public abstract fun dispatch(context: CoroutineContext, block: Runnable)
@InternalCoroutinesApi
public open fun dispatchYield(context: CoroutineContext, block: Runnable): Unit = dispatch(context, block)
```

剛剛談到的 `dispatch` 其實是一個抽象方法，他的實作之後會談談。

```kotlin
@ExperimentalCoroutinesApi
public open fun limitedParallelism(parallelism: Int): CoroutineDispatcher {
    parallelism.checkParallelism()
    return LimitedDispatcher(this, parallelism)
}
```

`limitedParallelism` 主要行為就是創建 **LimitedDispatcher**，一個可以限制一次跑幾個 **Runnable** 的 **CoroutineDispatcher**。

```kotlin
public final override fun <T> interceptContinuation(continuation: Continuation<T>): Continuation<T> =
    DispatchedContinuation(this, continuation)

public final override fun releaseInterceptedContinuation(continuation: Continuation<*>) {
    /*
     * Unconditional cast is safe here: we only return DispatchedContinuation from `interceptContinuation`,
     * any ClassCastException can only indicate compiler bug
     */
    val dispatched = continuation as DispatchedContinuation<*>
    dispatched.release()
}
```

`interceptContinuation` 會將 `continuation` 與 `this` 用 **DispatchedContinuation** 包裹起來。 而 `releaseInterceptedContinuation` 則會調用 **DispatchedContinuation** 的 `release`。

這裡提到的 ＊*DispatchedContinuation** 除了是一個 **Continuation** 外，它還是一個 **DispatchedTask**。

```
DispatchedContinuation <: DispatchedTask <: SchedulerTask <: Task <: Runnable
```

我們會在之後的 **Task** 中看到他們之間的關係。

```kotlin
@Suppress("DeprecatedCallableAddReplaceWith")
@Deprecated(
    message = "Operator '+' on two CoroutineDispatcher objects is meaningless. " +
        "CoroutineDispatcher is a coroutine context element and `+` is a set-sum operator for coroutine contexts. " +
        "The dispatcher to the right of `+` just replaces the dispatcher to the left.",
    level = DeprecationLevel.ERROR
)
public operator fun plus(other: CoroutineDispatcher): CoroutineDispatcher = other

override fun toString(): String = "$classSimpleName@$hexAddress"
```






#### LimitedDispatcher
```kotlin
internal class LimitedDispatcher(
    private val dispatcher: CoroutineDispatcher,
    private val parallelism: Int
) : CoroutineDispatcher(), Runnable, Delay by (dispatcher as? Delay ?: DefaultDelay)
```


## ContinuationInterceptor

從名字就知道這是用來攔截 **Continuation** 的。另外，這

```kotlin
@SinceKotlin("1.3")
public interface ContinuationInterceptor : CoroutineContext.Element {
   companion object Key : CoroutineContext.Key<ContinuationInterceptor>
   public fun <T> interceptContinuation(continuation: Continuation<T>): Continuation<T>
   public fun releaseInterceptedContinuation(continuation: Continuation<*>) { }

   public override operator fun <E : CoroutineContext.Element> get(key: CoroutineContext.Key<E>): E? {
        // getPolymorphicKey specialized for ContinuationInterceptor key
        @OptIn(ExperimentalStdlibApi::class)
        if (key is AbstractCoroutineContextKey<*, *>) {
            @Suppress("UNCHECKED_CAST")
            return if (key.isSubKey(this.key)) key.tryCast(this) as? E else null
        }
        @Suppress("UNCHECKED_CAST")
        return if (ContinuationInterceptor === key) this as E else null
    }


    public override fun minusKey(key: CoroutineContext.Key<*>): CoroutineContext {
        // minusPolymorphicKey specialized for ContinuationInterceptor key
        @OptIn(ExperimentalStdlibApi::class)
        if (key is AbstractCoroutineContextKey<*, *>) {
            return if (key.isSubKey(this.key) && key.tryCast(this) != null) EmptyCoroutineContext else this
        }
        return if (ContinuationInterceptor === key) EmptyCoroutineContext else this
    }
}
```

## Job

><br>
>
>**Job** 在 **Coroutine** 中的重要性可以從這句話看出來：
> A Job instance in the coroutineContext **represents the coroutine itself**.
><br>

也就是說 **Job** 是 **Coroutine** 中的精華。

那他為何那麼特別呢？ 我們來窺探究竟吧。

```kotlin
public interface Job : CoroutineContext.Element {
    public companion object Key : CoroutineContext.Key<Job>

    // check state
    public val isActive: Boolean
    public val isCompleted: Boolean
    public val isCancelled: Boolean

    // start
    public fun start(): Boolean

    // cancelled
    public fun cancel(cause: CancellationException? = null)

    @InternalCoroutinesApi
    public fun getCancellationException(): CancellationException

    // child
    public val children: Sequence<Job>

    @InternalCoroutinesApi
    public fun attachChild(child: ChildJob): ChildHandle

    // state waiting
    public suspend fun join()
    public val onJoin: SelectClause0

    // low-level state-notification
    public fun invokeOnCompletion(handler: CompletionHandler): DisposableHandle
    @InternalCoroutinesApi
    public fun invokeOnCompletion(
        onCancelling: Boolean = false,
        invokeImmediately: Boolean = true,
        handler: CompletionHandler): DisposableHandle


    // useless plus, just like CoroutineDispatcher
    public operator fun plus(other: Job): Job = other
}
```

**Job** 擁有不同的 `state`：

|State|isActive|isCompleted|isCancelled|
|:--|:--:|:--:|:--:|
|**New** (optional initial state)|false|false|false|
|**Active** (default initial state)|true|false|false|
|**Completing** (transient state)|true|false|false|
|**Cancelling** (transient state)|false|false|true|
|**Cancelled** (final state)|false|true|true|
|**Completed** (final state)|false|true|false|


而這些 `state` 可能會出現的順序或流程為以下：
```

+-----+ start  +--------+ complete   +-------------+  finish  +-----------+
| New | -----> | Active | ---------> | Completing  | -------> | Completed |
+-----+        +--------+            +-------------+          +-----------+
                 |  cancel / fail       |
                 |     +----------------+
                 |     |
                 V     V
             +------------+                           finish  +-----------+
             | Cancelling | --------------------------------> | Cancelled |
             +------------+                                   +-----------+
```


從源碼我們只知道 **Job** 有自己的狀態，無法調用 `plus` 但取而代之的是 `children: Sequence<Job>`。

另外，當調用 `attachChild` 時，他會回傳一個 **ChildHandle**。 這類別是用來讓 child 在 cancel 是可以通過 `childCancelled` 通知 parent 的。

想要知道更多關於 **Job** 的作用，我們得看看他的實作與繼承者：
- **CompletableJob**
- **JobSupport**


### CompletableJob

**CompletableJob** 也只是一個介面，但繼承者都是可被 `complete` 的類別：

```kotlin
public interface CompletableJob : Job {
    public fun complete(): Boolean
    public fun completeExceptionally(exception: Throwable): Boolean
}
```

這表示實作 **CompletableJob** 的都是可以被 `complete` 的 **Job**。 唯一實作的類別便是：
- **JobImpl**

### JobSupport
**JobSupport** 扮演著很重要的角色，包括 **Job**、**ChildJob** 、**ParentJob** 與 **SelectClause0** ：

```kotlin
@Deprecated(level = DeprecationLevel.ERROR, message = "This is internal API and may be removed in the future releases")
public open class JobSupport constructor(active: Boolean) : Job, ChildJob, ParentJob, SelectClause0 {
    final override val key: CoroutineContext.Key<*> get() = Job
    /* others  */
}
```

**ChildJob** 是一個可被 `parent` 取消的 **Job**：

```kotlin
@InternalCoroutinesApi
@Deprecated(level = DeprecationLevel.ERROR, message = "This is internal API and may be removed in the future releases")
public interface ChildJob : Job {
    @InternalCoroutinesApi
    public fun parentCancelled(parentJob: ParentJob)
}
```

**ParentJob** 是一個能告訴 **ChildJob** 為何它會被取消的 **Job**

```kotlin
@InternalCoroutinesApi
@Deprecated(level = DeprecationLevel.ERROR, message = "This is internal API and may be removed in the future releases")
public interface ParentJob : Job {
    @InternalCoroutinesApi
    public fun getChildJobCancellationCause(): CancellationException
}
```

**SelectClause0** 是一個能夠註冊 **SelectInstance** 的介面：

```kotlin
public interface SelectClause0 {
    @InternalCoroutinesApi
    public fun <R> registerSelectClause0(select: SelectInstance<R>, block: suspend () -> R)
}
```

所謂的 **SelectInstance** 是一個介面：

```kotlin
@InternalCoroutinesApi // todo: sealed interface https://youtrack.jetbrains.com/issue/KT-22286
public interface SelectInstance<in R> {
    public val isSelected: Boolean
    public fun trySelect(): Boolean
    public fun trySelectOther(otherOp: PrepareOp?): Any? // returns RESUME_TOKEN on success, RETRY_ATOMIC on deadlock
    public fun performAtomicTrySelect(desc: AtomicDesc): Any?

    public val completion: Continuation<R>

    public fun resumeSelectWithException(exception: Throwable)
    public fun disposeOnSelect(handle: DisposableHandle)
}
```

而實作這個介面的是 **SelectBuilderImpl** ：

```kotlin
@PublishedApi
internal class SelectBuilderImpl<in R>(
    private val uCont: Continuation<R> // unintercepted delegate continuation
) : LockFreeLinkedListHead(), SelectBuilder<R>,
    SelectInstance<R>, Continuation<R>, CoroutineStackFrame
```



#### JobImpl

**Job** 的最簡單的創建就是一個 **CompletableJob**，而它的實作則是由 **JobImpl** 完成：

```kotlin
@Suppress("FunctionName")
public fun Job(parent: Job? = null): CompletableJob = JobImpl(parent)
```

**JobImpl** 是直接繼承 **CompletableJob** 的類別，我們來看看他是如何實作的吧。

```kotlin
@PublishedApi // for a custom job in the test module
internal open class JobImpl(parent: Job?) : JobSupport(true), CompletableJob
```

以下便是 `complete` 與 `completeExceptionally` 的實作：

```kotlin
override fun complete() = makeCompleting(Unit)
override fun completeExceptionally(exception: Throwable): Boolean =
    makeCompleting(CompletedExceptionally(exception))
```

它的實作都會將真正的實作導向 **JobSupport** 的 `makeCompleting`：

```kotlin
internal fun makeCompleting(proposedUpdate: Any?): Boolean {
    loopOnState { state ->
        val finalState = tryMakeCompleting(state, proposedUpdate)
        when {
            finalState === COMPLETING_ALREADY -> return false
            finalState === COMPLETING_WAITING_CHILDREN -> return true
            finalState === COMPLETING_RETRY -> return@loopOnState
            else -> {
                afterCompletion(finalState)
                return true
            }
        }
    }
}
```

`makeCompleting` 的行為其實是通過 `loopOnState` 不斷地對 `_state` 進行觀察並將將非 **OpDescriptor** 的 `state` 進行 `block` 的調用：

```kotlin
private inline fun loopOnState(block: (Any?) -> Unit): Nothing {
    while (true) {
        block(state)
    }
}

internal val state: Any? get() {
    _state.loop { state -> // helper loop on state (complete in-progress atomic operations)
        if (state !is OpDescriptor) return state
        state.perform(this)
    }
}

// Note: use shared objects while we have no listeners
private val _state = atomic<Any?>(if (active) EMPTY_ACTIVE else EMPTY_NEW)
```

這裡的 `state` 指的是 **JobSupport** 的 **內部狀態**，之後會提到。

接下來我們來看看 `tryMakeCompleting` 有什麼功能吧：

```kotlin
// Returns one of COMPLETING symbols or final state:
// COMPLETING_ALREADY -- when already complete or completing
// COMPLETING_RETRY -- when need to retry due to interference
// COMPLETING_WAITING_CHILDREN -- when made completing and is waiting for children
// final state -- when completed, for call to afterCompletion
private fun tryMakeCompleting(state: Any?, proposedUpdate: Any?): Any? {
    if (state !is Incomplete)
        return COMPLETING_ALREADY
    /*
     * FAST PATH -- no children to wait for && simple state (no list) && not cancelling => can complete immediately
     * Cancellation (failures) always have to go through Finishing state to serialize exception handling.
     * Otherwise, there can be a race between (completed state -> handled exception and newly attached child/join)
     * which may miss unhandled exception.
     */
    if ((state is Empty || state is JobNode) && state !is ChildHandleNode && proposedUpdate !is CompletedExceptionally) {
        if (tryFinalizeSimpleState(state, proposedUpdate)) {
            // Completed successfully on fast path -- return updated state
            return proposedUpdate
        }
        return COMPLETING_RETRY
    }
    // The separate slow-path function to simplify profiling
    return tryMakeCompletingSlowPath(state, proposedUpdate)
}
```

`tryMakeCompleting` 做了以下是件：
1. 確保 `state` 的狀態為 **Empty** 或非 **ChildHandleNode** 的 **JobNode**
2. 確保 `proposedUpdate` 並非 **CompleteExceptionally**。 因為只有要被取消的 `proposedUpdate` 才會是 **CompleteExceptionally**
3. 若上方條件達成，調用 `tryFinalizeSimpleState` 來嘗試更新 `_state`。 並通過 `completeStateFinalization` 將 `parentHandle` 移除。 最後便通知 **JobNode** 或 **NodeList** 的 **CompletionHandler** 。


```kotlin
// fast-path method to finalize normally completed coroutines without children
// returns true if complete, and afterCompletion(update) shall be called
private fun tryFinalizeSimpleState(state: Incomplete, update: Any?): Boolean {
    assert { state is Empty || state is JobNode } // only simple state without lists where children can concurrently add
    assert { update !is CompletedExceptionally } // only for normal completion
    if (!_state.compareAndSet(state, update.boxIncomplete())) return false
    onCancelling(null) // simple state is not a failure
    onCompletionInternal(update)
    completeStateFinalization(state, update)
    return true
}

// suppressed == true when any exceptions were suppressed while building the final completion cause
private fun completeStateFinalization(state: Incomplete, update: Any?) {
    /*
     * Now the job in THE FINAL state. We need to properly handle the resulting state.
     * Order of various invocations here is important.
     *
     * 1) Unregister from parent job.
     */
    parentHandle?.let {
        it.dispose() // volatile read parentHandle _after_ state was updated
        parentHandle = NonDisposableHandle // release it just in case, to aid GC
    }
    val cause = (update as? CompletedExceptionally)?.cause
    /*
     * 2) Invoke completion handlers: .join(), callbacks etc.
     *    It's important to invoke them only AFTER exception handling and everything else, see #208
     */
    if (state is JobNode) { // SINGLE/SINGLE+ state -- one completion handler (common case)
        try {
            state.invoke(cause)
        } catch (ex: Throwable) {
            handleOnCompletionException(CompletionHandlerException("Exception in completion handler $state for $this", ex))
        }
    } else {
        state.list?.notifyCompletion(cause)
    }
}
```

4. 若上方的條件沒有達成，那便調用 `tryMakeCompletingSlowPath` 來試圖取消或完成當下的 **JobNode**。另外，如果 `state` 是個 **NodeList** 且有 `handle` 並非 **NonDisposableHandle** 時就會回傳 **COMPLETING_WAITING_CHILDREN**










**JobImpl** 中最重要的一個步驟就是 `init`：
```kotlin
init { initParentJob(parent) }
```

通過 `init`， **JobImpl** 會調用 **JobSupport** 中的 `initParentJob`。這個方法會做以下的事：
1. 如果 `parent` 為 `null`，那就將 `parentHandle` 設為 **NonDisposableHandle** 並 return 結束作業
2. 如果 `parent` 不為 `null` ，那就直接讓其 `start` 並自己通過 `attachChild` 加入 **Job** 中。`parentHandle` 也會從 `attachChild` 取得。


由於這裡的 `parent` 為 `null`，所以 **JobImpl** 除了創建 `parentHandle` 外，就沒有任何行為了。

但由於 **JobImpl** 實質是 **CompletableJob**，所以它還有實作了 `complete` 和 `completeExceptionally`：












而 `attachChild` 也是由 **JobSupport** 實作：
```kotlin
@Suppress("OverridingDeprecatedMember")
public final override fun attachChild(child: ChildJob): ChildHandle {
    /*
     * Note: This function attaches a special ChildHandleNode node object. This node object
     * is handled in a special way on completion on the coroutine (we wait for all of them) and
     * is handled specially by invokeOnCompletion itself -- it adds this node to the list even
     * if the job is already cancelling. For cancelling state child is attached under state lock.
     * It's required to properly wait all children before completion and provide linearizable hierarchy view:
     * If child is attached when the job is already being cancelled, such child will receive immediate notification on
     * cancellation, but parent *will* wait for that child before completion and will handle its exception.
     */
    return invokeOnCompletion(onCancelling = true, handler = ChildHandleNode(child).asHandler) as ChildHandle
}
```



```kotlin
@Suppress("OverridingDeprecatedMember")
public final override fun invokeOnCompletion(handler: CompletionHandler): DisposableHandle =
    invokeOnCompletion(onCancelling = false, invokeImmediately = true, handler = handler)

public final override fun invokeOnCompletion(
    onCancelling: Boolean,
    invokeImmediately: Boolean,
    handler: CompletionHandler
): DisposableHandle {
    // Create node upfront -- for common cases it just initializes JobNode.job field,
    // for user-defined handlers it allocates a JobNode object that we might not need, but this is Ok.
    val node: JobNode = makeNode(handler, onCancelling)
    loopOnState { state ->
        when (state) {
            is Empty -> { // EMPTY_X state -- no completion handlers
                if (state.isActive) {
                    // try move to SINGLE state
                    if (_state.compareAndSet(state, node)) return node
                } else
                    promoteEmptyToNodeList(state) // that way we can add listener for non-active coroutine
            }
            is Incomplete -> {
                val list = state.list
                if (list == null) { // SINGLE/SINGLE+
                    promoteSingleToNodeList(state as JobNode)
                } else {
                    var rootCause: Throwable? = null
                    var handle: DisposableHandle = NonDisposableHandle
                    if (onCancelling && state is Finishing) {
                        synchronized(state) {
                            // check if we are installing cancellation handler on job that is being cancelled
                            rootCause = state.rootCause // != null if cancelling job
                            // We add node to the list in two cases --- either the job is not being cancelled
                            // or we are adding a child to a coroutine that is not completing yet
                            if (rootCause == null || handler.isHandlerOf<ChildHandleNode>() && !state.isCompleting) {
                                // Note: add node the list while holding lock on state (make sure it cannot change)
                                if (!addLastAtomic(state, list, node)) return@loopOnState // retry
                                // just return node if we don't have to invoke handler (not cancelling yet)
                                if (rootCause == null) return node
                                // otherwise handler is invoked immediately out of the synchronized section & handle returned
                                handle = node
                            }
                        }
                    }
                    if (rootCause != null) {
                        // Note: attachChild uses invokeImmediately, so it gets invoked when adding to cancelled job
                        if (invokeImmediately) handler.invokeIt(rootCause)
                        return handle
                    } else {
                        if (addLastAtomic(state, list, node)) return node
                    }
                }
            }
            else -> { // is complete
                // :KLUDGE: We have to invoke a handler in platform-specific way via `invokeIt` extension,
                // because we play type tricks on Kotlin/JS and handler is not necessarily a function there
                if (invokeImmediately) handler.invokeIt((state as? CompletedExceptionally)?.cause)
                return NonDisposableHandle
            }
        }
    }
}
```

##### ReportingSupervisorJob

##### SupervisorJob




```kotlin
public final override fun start(): Boolean {
    loopOnState { state ->
        when (startInternal(state)) {
            FALSE -> return false
            TRUE -> return true
        }
    }
}
```


### JobSupport



#### AbstractCoroutine



##### SupervisorJobImpl
```kotlin
private class SupervisorJobImpl(parent: Job?) : JobImpl(parent) {
    override fun childCancelled(cause: Throwable): Boolean = false
}
```

## Deferred
```kotlin
public interface Deferred<out T> : Job {
    public suspend fun await(): T
    public val onAwait: SelectClause1<T>
    @ExperimentalCoroutinesApi
    public fun getCompleted(): T
    @ExperimentalCoroutinesApi
    public fun getCompletionExceptionOrNull(): Throwable?
}
```

- **DeferredCoroutine**
- **CompletableDeferred**

## CompletableDeferred

## DeferredCoroutine
- **LazyDeferredCoroutine**


# CoroutineContext.Key
雖然我們已經看過 **Key** 了，也知道它在 **Coroutine** 中的作用。 但它到底是什麼呢？

原來它只是一個空介面，但是它是有功能的。它的主要功能是確保裡面所放的必須是 **Element**：

```kotlin
public interface Key<E : Element>
```

接下來我們來看一看 **Key** 的唯一繼承者 **AbstractCoroutineContextKey**。

## AbstractCoroutineContextKey

**AbstractCoroutineContextKey** 是一個很特別的 **Key**，它裡面包含著兩種 **Element**：
- **B**
  這是 `baseKey`
- 以及繼承 **B** 的 **E**

```kotlin
@SinceKotlin("1.3")
@ExperimentalStdlibApi
public abstract class AbstractCoroutineContextKey<B : Element, E : B>(
    baseKey: Key<B>,
    private val safeCast: (element: Element) -> E?
) : Key<E> {
    private val topmostKey: Key<*> = if (baseKey is AbstractCoroutineContextKey<*, *>) baseKey.topmostKey else baseKey

    internal fun tryCast(element: Element): E? = safeCast(element)
    internal fun isSubKey(key: Key<*>): Boolean = key === this || topmostKey === key
}
```

我們可以從 `isSubKey` 看出，當我們要檢查 **AbstractCoroutineContextKey** 是否為某 `key` 時，他會檢查 `topmostKey` 與 `this`。 而 `topmostKey` 則會找到最原始的 `baseKey`，也就是最底層的父類別 。

所以， **AbstractCoroutineContextKey** 在檢查 `key` 的行為上相當於在檢查父類別與其繼承者，有點像是協變。

那為什麼要如此設計呢？這種設計又有什麼用呢？

我們試著從以下範例來暸解為何要如此設計吧：

```kotlin
open class BaseElement : CoroutineContext.Element {
    companion object Key : CoroutineContext.Key<BaseElement>
    override val key: CoroutineContext.Key<*> get() = Key
    // It is important to use getPolymorphicKey and minusPolymorphicKey
    override fun <E : CoroutineContext.Element> get(key: CoroutineContext.Key<E>): E? = getPolymorphicElement(key)
    override fun minusKey(key: CoroutineContext.Key<*>): CoroutineContext = minusPolymorphicKey(key)
}

class DerivedElement : BaseElement() {
    companion object Key : AbstractCoroutineContextKey<BaseElement, DerivedElement>(BaseElement, { it as? DerivedElement })
}
```

這範例中很清楚地看出， `BaseElement :> DerivedElement`。 另外一個很重要的細節就是 **BaseElement** 中 `get` 與 `minus` 的實作。

```kotlin
override fun <E : CoroutineContext.Element> get(key: CoroutineContext.Key<E>): E? = getPolymorphicElement(key)
override fun minusKey(key: CoroutineContext.Key<*>): CoroutineContext = minusPolymorphicKey(key)
```

其中的 `getPolymorphicKey` 與 `minusPolymorphicKey` 其實是兩個 **Element** 延伸出來的方法：

```kotlin
@SinceKotlin("1.3")
@ExperimentalStdlibApi
public fun <E : Element> Element.getPolymorphicElement(key: Key<E>): E? {
    if (key is AbstractCoroutineContextKey<*, *>) {
        @Suppress("UNCHECKED_CAST")
        return if (key.isSubKey(this.key)) key.tryCast(this) as? E else null
    }
    @Suppress("UNCHECKED_CAST")
    return if (this.key === key) this as E else null
}

@SinceKotlin("1.3")
@ExperimentalStdlibApi
public fun Element.minusPolymorphicKey(key: Key<*>): CoroutineContext {
    if (key is AbstractCoroutineContextKey<*, *>) {
        return if (key.isSubKey(this.key) && key.tryCast(this) != null) EmptyCoroutineContext else this
    }
    return if (this.key === key) EmptyCoroutineContext else this
}
```

當 `getPolymorphicKey` 與 `minusPolymorphicKey` 遇到 **AbstractCoroutineContextKey** 時，他們會通過 `isSubKey` 來進行檢查。


如此一來，當我們調用 `someContext[BaseElement]` 或 `someContext[DerivedElement]` 時，他們會通過 `isSubKey` 回傳 **BaseElement?** 或 **DerivedElement?**。

有如此設計的類別包括以下：
- **CoroutineDispatcher**
- **ExecutorCoroutineDispatcher**

### CoroutineDispatcher



```kotlin

```



# Continuation
```kotlin
@SinceKotlin("1.3")
public interface Continuation<in T> {
    public val context: CoroutineContext
    public fun resumeWith(result: Result<T>)
}
```


```kotlin
@SinceKotlin("1.3")
@InlineOnly
public inline fun <T> Continuation(
    context: CoroutineContext,
    crossinline resumeWith: (Result<T>) -> Unit
): Continuation<T> =
    object : Continuation<T> {
        override val context: CoroutineContext
            get() = context

        override fun resumeWith(result: Result<T>) =
            resumeWith(result)
    }
```

這裡使用 `crossinline` 是為了避免 `resumeWith` 中有 `return` 的存在。

Android 提供我們幾種創建 **Continuation** 的方法，包括：
- `createSimpleCoroutineForSuspendFunction`
  這方法會
- `createCoroutineFromSuspendFunction`
- `suspend` 方法


```kotlin
private fun <T> createSimpleCoroutineForSuspendFunction(
    completion: Continuation<T>
): Continuation<T> {
    val context = completion.context
    return if (context === EmptyCoroutineContext)
        object : RestrictedContinuationImpl(completion as Continuation<Any?>) {
            override fun invokeSuspend(result: Result<Any?>): Any? {
                return result.getOrThrow()
            }
        }
    else
        object : ContinuationImpl(completion as Continuation<Any?>, context) {
            override fun invokeSuspend(result: Result<Any?>): Any? {
                return result.getOrThrow()
            }
        }
}
```


```kotlin
@SinceKotlin("1.3")
private inline fun <T> createCoroutineFromSuspendFunction(
    completion: Continuation<T>,
    crossinline block: (Continuation<T>) -> Any?
): Continuation<Unit> {
    val context = completion.context
    // label == 0 when coroutine is not started yet (initially) or label == 1 when it was
    return if (context === EmptyCoroutineContext)
        object : RestrictedContinuationImpl(completion as Continuation<Any?>) {
            private var label = 0

            override fun invokeSuspend(result: Result<Any?>): Any? =
                when (label) {
                    0 -> {
                        label = 1
                        result.getOrThrow() // Rethrow exception if trying to start with exception (will be caught by BaseContinuationImpl.resumeWith
                        block(this) // run the block, may return or suspend
                    }
                    1 -> {
                        label = 2
                        result.getOrThrow() // this is the result if the block had suspended
                    }
                    else -> error("This coroutine had already completed")
                }
        }
    else
        object : ContinuationImpl(completion as Continuation<Any?>, context) {
            private var label = 0

            override fun invokeSuspend(result: Result<Any?>): Any? =
                when (label) {
                    0 -> {
                        label = 1
                        result.getOrThrow() // Rethrow exception if trying to start with exception (will be caught by BaseContinuationImpl.resumeWith
                        block(this) // run the block, may return or suspend
                    }
                    1 -> {
                        label = 2
                        result.getOrThrow() // this is the result if the block had suspended
                    }
                    else -> error("This coroutine had already completed")
                }
        }
}
```

```kotlin
@SinceKotlin("1.3")
public actual fun <T> (suspend () -> T).createCoroutineUnintercepted(
    completion: Continuation<T>
): Continuation<Unit> {
    // probeCoroutineCreated's action is replaced by debugger
    // the main purpose is to keep track on the stage continuation is in
    // START, SUSPEND, RUNNING, COMPLETE
    val probeCompletion = probeCoroutineCreated(completion)

    return if (this is BaseContinuationImpl)
        create(probeCompletion) // this will throw error
    else
        createCoroutineFromSuspendFunction(probeCompletion) {
            (this as Function1<Continuation<T>, Any?>).invoke(it)
        }
}

// package kotlin.jvm.functions
public interface Function1<in P1, out R> : Function<R> {
    /** Invokes the function with the specified argument. */
    public operator fun invoke(p1: P1): R
}
```




```kotlin
@SinceKotlin("1.3")
@InlineOnly
public inline fun <T> Continuation<T>.resume(value: T): Unit =
    resumeWith(Result.success(value))
```

```kotlin
@SinceKotlin("1.3")
@InlineOnly
public inline fun <T> Continuation<T>.resumeWithException(exception: Throwable): Unit =
    resumeWith(Result.failure(exception))
```

## CancellableContinuation

## SafeContinuation
```kotlin
// SafeContinuationJVM.kt
@PublishedApi
@SinceKotlin("1.3")
internal actual class SafeContinuation<in T>
internal actual constructor(
    private val delegate: Continuation<T>,
    initialResult: Any?
) : Continuation<T>, CoroutineStackFrame
```

**SafeContinuation** 的主要目的可以從 `resumeWith` 實作看出來：

```kotlin
public actual override fun resumeWith(result: Result<T>) {
    while (true) { // lock-free loop
        val cur = this.result // atomic read
        when {
            cur === UNDECIDED -> if (RESULT.compareAndSet(this, UNDECIDED, result.value)) return
            cur === COROUTINE_SUSPENDED -> if (RESULT.compareAndSet(this, COROUTINE_SUSPENDED, RESUMED)) {
                delegate.resumeWith(result)
                return
            }
            else -> throw IllegalStateException("Already resumed")
        }
    }
}
```

在 `resumeWith` 中，它會不斷地觀察 `result` 的值為何。 當目前的值是 `COROUTINE_SUSPENDED`，他會將 `result` 改為 `RESUMED` 並調用 `delegate: Continuation` 的 `resumeWith` 並告知新的 `result` 值。

源碼中看到的 `UNDECIDED` 、 `RESUMED` 與 `COROUTINE_SUSPENDED` 都是一個 enum 中的值：

```kotlin
// Using enum here ensures two important properties:
//  1. It makes SafeContinuation serializable with all kinds of serialization frameworks (since all of them natively support enums)
//  2. It improves debugging experience, since you clearly see toString() value of those objects and what package they come from
@SinceKotlin("1.3")
@PublishedApi // This class is Published API via serialized representation of SafeContinuation, don't rename/move
internal enum class CoroutineSingletons { COROUTINE_SUSPENDED, UNDECIDED, RESUMED }
```


我們可以從多個地方創造出 **SafeContinuation**，包括：

```kotlin
@SinceKotlin("1.3")
@Suppress("UNCHECKED_CAST")
public fun <T> (suspend () -> T).createCoroutine(
    completion: Continuation<T>
): Continuation<Unit> =
    SafeContinuation(createCoroutineUnintercepted(completion).intercepted(), COROUTINE_SUSPENDED)


```



## BaseContinuationImpl


### RestrictedContinuationImpl
State machines for named restricted suspend functions extend from this class

```kotlin
@SinceKotlin("1.3")
internal abstract class RestrictedContinuationImpl(
    completion: Continuation<Any?>?
) : BaseContinuationImpl(completion) {
    init {
        completion?.let {
            require(it.context === EmptyCoroutineContext) {
                // exception message
                "Coroutines with restricted suspension must have EmptyCoroutineContext"
            }
        }
    }

    public override val context: CoroutineContext
        get() = EmptyCoroutineContext
}
```



# Task / SchedulerTask
**Task** 是一個 **Runnable** 的抽象類別，也就是說繼承 **Task** 的必須要實作 `run` 方法。

**Task** 也被稱為 **SchedulerTask** ：
```kotlin
internal actual typealias SchedulerTask = Task
```

**Task** 與 **Runnable** 的差別在於 **Task** 還包含兩個參數：
- `submissionTime`：這是用在 debug 時知道它是何時被 submit 的
- **TaskContext**

```kotlin
internal abstract class Task(
    @JvmField var submissionTime: Long,
    @JvmField var taskContext: TaskContext
) : Runnable {
    constructor() : this(0, NonBlockingContext)
    inline val mode: Int get() = taskContext.taskMode // TASK_XXX
}
```
**TaskContext** 會記錄 **Task** 的 `taskMode` 或 狀態，以及做完這個 Task 之後該做的事 (`afterTask`)：

```kotlin
internal interface TaskContext {
    val taskMode: Int // TASK_XXX
    fun afterTask()
}

private class TaskContextImpl(override val taskMode: Int): TaskContext {
    override fun afterTask() {
        // Nothing for non-blocking context
    }
}
```

以下是兩個不同的 **TaskContext** ：

```kotlin
@JvmField
internal val NonBlockingContext: TaskContext = TaskContextImpl(TASK_NON_BLOCKING)

@JvmField
internal val BlockingContext: TaskContext = TaskContextImpl(TASK_PROBABLY_BLOCKING)
```

實作 **Task** 的類別包括：
- **DispatchedTask**
- **TaskImpl**

## TaskImpl
**TaskImpl** 是 **Task** 的基本實作類別。 它只是一個 **Runnable** 的 wrapper，並且會在 `finally` 時調用 `afterTask`：

```kotlin
// Non-reusable Task implementation to wrap Runnable instances that do not otherwise implement task
internal class TaskImpl(
    @JvmField val block: Runnable,
    submissionTime: Long,
    taskContext: TaskContext
) : Task(submissionTime, taskContext) {
    override fun run() {
        try {
            block.run()
        } finally {
            taskContext.afterTask()
        }
    }

    override fun toString(): String =
        "Task[${block.classSimpleName}@${block.hexAddress}, $submissionTime, $taskContext]"
}
```

這類別並沒有特別的用處。

## DispatchedTask
><br>
>
>**DispatchedTask** 的作用是用來檢查 **Job** 是否順利完成。這我們可以在 `run` 裡面看出來。
><br>


相較於 **Task**， **DispatchedTask** 多了不少方法與參數。 其中最重要的有：
- **Continuation**
- `resumeMode`
- `run`

我們先看看 `resumeMode` 的作用吧：

```kotlin
internal abstract class DispatchedTask<in T>(
    @JvmField public var resumeMode: Int
) : SchedulerTask()

internal const val MODE_ATOMIC = 0
@PublishedApi
internal const val MODE_CANCELLABLE: Int = 1
internal const val MODE_CANCELLABLE_REUSABLE = 2
internal const val MODE_UNDISPATCHED = 4
internal const val MODE_UNINITIALIZED = -1

internal val Int.isCancellableMode get() = this == MODE_CANCELLABLE || this == MODE_CANCELLABLE_REUSABLE
internal val Int.isReusableMode get() = this == MODE_CANCELLABLE_REUSABLE
```

`resumeMode` 主要是用來確認目前 **DispatchedTask** 的狀態，包括： `MODE_CANCELLABLE`, `MODE_CANCELLABLE_REUSABLE`, `MODE_UNDISPATCHED` 或 `MODE_UNINITIALIZED`。

接下來，我們看看 **DispatchedTask** 中有哪些方法和變數：

```kotlin
internal abstract val delegate: Continuation<T>

internal abstract fun takeState(): Any?

// Called when this task was cancelled while it was being dispatched.
internal open fun cancelCompletedResult(takenState: Any?, cause: Throwable) {}

@Suppress("UNCHECKED_CAST")
internal open fun <T> getSuccessfulResult(state: Any?): T =
        state as T
internal open fun getExceptionalResult(state: Any?): Throwable? =
        (state as? CompletedExceptionally)?.cause

public fun handleFatalException(exception: Throwable?, finallyException: Throwable?) {
    if (exception === null && finallyException === null) return
    if (exception !== null && finallyException !== null) {
        exception.addSuppressedThrowable(finallyException)
    }

    val cause = exception ?: finallyException
    val reason = CoroutinesInternalError("Fatal exception in coroutines machinery for $this. " +
            "Please read KDoc to 'handleFatalException' method and report this incident to maintainers", cause!!)
    handleCoroutineException(this.delegate.context, reason)
}
```

以上的方法和變數主要是解釋了 **DispatchedTask** 是如何處理錯誤的。

但這些東西的重要性並不及以下 `run` 方法的一半：

```kotlin
public final override fun run() {
    assert { resumeMode != MODE_UNINITIALIZED } // should have been set before dispatching
    val taskContext = this.taskContext
    var fatalException: Throwable? = null
    try {
        val delegate = delegate as DispatchedContinuation<T>
        val continuation = delegate.continuation

        // countOrElement is used to count all ThreadContextElement the CoroutineContext
        // in DispatchedContinuation, thanks to Continuation
        withContinuationContext(continuation, delegate.countOrElement) {

            // prior working with this block, withContinuationContext will store the state as follows :
            // 1) update Thread state with countOrElement
            // 2) if there were ThreadContextElement in oldValue, then it will call continuation.updateUndispatchedCompletion and stores the UndispatchedCompletion
            //    - this function will make store the oldValue inside CoroutineStackFrame if this DispatchedTask has not become DispatchedCoroutine
            // 3) run this block
            // 4) at the end restoreThreadContext with oldValue if undispatchedCompletion is available.


            val context = continuation.context

            // 5) takeState acts differently between DispatchedContinuation and CancellableContinuation
            // DispatchedContinuation will return the current state than update it to UNDEFINED
            // CancellableContinuation will simply return the current state
            val state = takeState() // NOTE: Must take state in any case, even if cancelled

            // 6) check if state is CompletedExceptionally. If true, then get the Throwable
            val exception = getExceptionalResult(state)

            /*
             * Check whether continuation was originally resumed with an exception.
             * If so, it dominates cancellation, otherwise the original exception
             * will be silently lost.
             */
            // 7) If there is no exception nor is it in CancellableMode, then find Job in context
            val job = if (exception == null && resumeMode.isCancellableMode) context[Job] else null

            // 8) If there's a job, and it's not active, then call cancelCompletedResult
            //    which called onCancellation if the state is CompletedWithCancellation
            if (job != null && !job.isActive) {
                val cause = job.getCancellationException()
                cancelCompletedResult(state, cause)
                continuation.resumeWithStackTrace(cause)
            } else {

            // 9) If (job is null) or (job is active), then call either resumeWithException or resume
            //    These two method will call `Continuation.resume` which will take either Result.failure or Result.success as parameter.
                if (exception != null) {
                    continuation.resumeWithException(exception)
                } else {
                    continuation.resume(getSuccessfulResult(state))
                }
            }
        }
    } catch (e: Throwable) {
        // This instead of runCatching to have nicer stacktrace and debug experience
        fatalException = e
    } finally {
        val result = runCatching { taskContext.afterTask() }
        handleFatalException(fatalException, result.exceptionOrNull())
    }
}
```

我們來寫一下 `run` 的流程吧：
1. 通過 `withContinuationContext` 進行 state 的保存。 這會保存在 **ThreadContextElement** 中。
2. 進入 Block
3. Block 主要功能是檢查是否有 exception 並通過 `continuation` 將結果回傳
4. 在最後，他會調用 `taskContext` 的 `afterTask` 來做最後的設定。

所以基本上 **DispatchedTask** 並不會與 Thread 有什麼關係，它只是用來檢查結果為何。

知道了 **DispatchedTask** 的用處，我們接下來要看他的繼承者們：
- **DispatchedContinuation**
- **CancellableContinuation**

### CancellableContinuationImpl
**CancellableContinuationImpl** 當中有一個很重要的東西，那就是與 **Job** 的聯繫。
通過 **Job** 它才可以在完成時進行 dispose。

```kotlin
/**
 * @suppress **This is unstable API and it is subject to change.**
 */
@PublishedApi
internal open class CancellableContinuationImpl<in T>(
    final override val delegate: Continuation<T>,
    resumeMode: Int
) : DispatchedTask<T>(resumeMode), CancellableContinuation<T>, CoroutineStackFrame
```



#### AwaitContinuation

**AwaitContinuation** 是 **JobSupport** 的內部類別。

```kotlin
private class AwaitContinuation<T>(
    delegate: Continuation<T>,
    private val job: JobSupport
) : CancellableContinuationImpl<T>(delegate, MODE_CANCELLABLE)
```


### DispatchedContinuation

```kotlin
internal class DispatchedContinuation<in T>(
    @JvmField val dispatcher: CoroutineDispatcher,
    @JvmField val continuation: Continuation<T>
) : DispatchedTask<T>(MODE_UNINITIALIZED), CoroutineStackFrame, Continuation<T> by continuation
```





# viewModelScope

`viewModelScope` 是一個位於 `androidx.lifecycle` 專案中的 **ViewModel** extension：

```kotlin
public val ViewModel.viewModelScope: CoroutineScope
  get() {
      val scope: CoroutineScope? = this.getTag(JOB_KEY)
      if (scope != null) {
          return scope
      }
      return setTagIfAbsent(
          JOB_KEY,
          CloseableCoroutineScope(SupervisorJob() + Dispatchers.Main.immediate)
      )
  }
```

他會通過 **SupervisorJob** 與 **Dispatchers.Main.immediate** 來創建出我們所使用的 **CoroutineScope** / **CloseableCoroutineScope**。

我們先看看什麼是 **Dispatchers** 吧。

## Dispatchers
**Dispatchers** 其實是一個包含多個 **CoroutineDispatcher** 的 Singleton：
```kotlin
public actual object Dispatchers {
  @JvmStatic
  public actual val Default: CoroutineDispatcher = DefaultScheduler

  @JvmStatic
  public actual val Main: MainCoroutineDispatcher get() = MainDispatcherLoader.dispatcher

  @JvmStatic
  public actual val Unconfined: CoroutineDispatcher = kotlinx.coroutines.Unconfined

  @JvmStatic
  public val IO: CoroutineDispatcher = DefaultIoScheduler

  @DelicateCoroutinesApi
  public fun shutdown() {
      DefaultExecutor.shutdown()
      // Also shuts down Dispatchers.IO
      DefaultScheduler.shutdown()
  }
}
```

### CoroutineDispatcher
**CoroutineDispatcher** 則是一個 **AbstractCoroutineContextElement** 的：

```kotlin
public abstract class CoroutineDispatcher :
    AbstractCoroutineContextElement(ContinuationInterceptor), ContinuationInterceptor
```


```kotlin
@SinceKotlin("1.3")
public interface ContinuationInterceptor : CoroutineContext.Element
```







<br><br><br><br><br>
