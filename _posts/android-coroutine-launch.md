---
layout: post
title:  Kotlinx.Coroutine - 從 launch 來談 Coroutine
date:   2022-08-08 16:15:07 +0800
categories: [coroutine, android, intermediate]
---

# launch
```kotlin
public fun CoroutineScope.launch(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> Unit
): Job {
    val newContext = newCoroutineContext(context)
    val coroutine = if (start.isLazy)
        LazyStandaloneCoroutine(newContext, block) else
        StandaloneCoroutine(newContext, active = true)
    coroutine.start(start, coroutine, block)
    return coroutine
}
```



```kotlin
// AbstractCoroutine

// 1. 此時 receiver 與 this 一樣，都會是 StandaloneCoroutine 或 LazyStandaloneCoroutine
public fun <R> start(start: CoroutineStart, receiver: R, block: suspend R.() -> T) {
    start(block, receiver, this)
}
```

此時的 `start` 其實是 **CoroutineStart**，而 `start(block, receiver, this)` 其實是調用 **CoroutineStart** 的 `invoke`：

```kotlin
@InternalCoroutinesApi
public operator fun <R, T> invoke(block: suspend R.() -> T, receiver: R, completion: Continuation<T>): Unit =
    when (this) {

        // 2. receiver 與 completion 皆為相同的 Coroutine
        DEFAULT -> block.startCoroutineCancellable(receiver, completion)

        ATOMIC -> block.startCoroutine(receiver, completion)
        UNDISPATCHED -> block.startCoroutineUndispatched(receiver, completion)
        LAZY -> Unit // will start lazily
    }
```

由於 `this` 此時是 **DEFAULT**，所以會調用 `block.startCoroutineCancellable(receiver, completion)`。

```kotlin
/**
 * Use this function to start coroutine in a cancellable way, so that it can be cancelled
 * while waiting to be dispatched.
 */

 // 3. 此時 R: StandaloneCoroutine 或 LazyStandaloneCoroutine
 //    由於 completion 也是 StandaloneCoroutine 或 LazyStandaloneCoroutine 所以 T 為 **Unit**
 //    這表示此時 startCoroutineCancellable 回傳的是 Unit
internal fun <R, T> (suspend (R) -> T).startCoroutineCancellable(
    receiver: R, completion: Continuation<T>,
    onCancellation: ((cause: Throwable) -> Unit)? = null
) =
    runSafely(completion) {
        createCoroutineUnintercepted(receiver, completion)
          .intercepted()
          .resumeCancellableWith(Result.success(Unit), onCancellation)
    }
```

`runSafely` 主要會調用 `block` 並使用 `completion` 來接 **Throwable**。 此時的 `completion` 會是 **LazyStandaloneCoroutine** 或 **StandaloneCoroutine**：

```kotlin
private inline fun runSafely(completion: Continuation<*>, block: () -> Unit) {
    try {
        block()
    } catch (e: Throwable) {
        dispatcherFailure(completion, e)
    }
}
```

`createCoroutineUnintercepted` 便會回傳 **Continuation**：

```kotlin
@SinceKotlin("1.3")
public actual fun <R, T> (suspend R.() -> T).createCoroutineUnintercepted(
    receiver: R,
    completion: Continuation<T>
): Continuation<Unit>
```

通過 `Continuation.interrupted()` 我們會調用 **CoroutineDispatcher** 的 `interceptContinuation` 並取得 **DispatchedContinuation**：

```kotlin
@SinceKotlin("1.3")
public actual fun <T> Continuation<T>.intercepted(): Continuation<T> =
    (this as? ContinuationImpl)?.intercepted() ?: this

// ContinuationImpl
public fun intercepted(): Continuation<Any?> =
    intercepted
        ?: (context[ContinuationInterceptor]?.interceptContinuation(this) ?: this)
            .also { intercepted = it }

// DispatchedContinuation
public final override fun <T> interceptContinuation(continuation: Continuation<T>): Continuation<T> =
    DispatchedContinuation(this, continuation)
```

最後便會在 `(suspend (R) -> T).startCoroutineCancellable` 中取得 **DispatchedContinuation**，最後便會調用 `resumeCancellableWith(Result.success(Unit), onCancellation)`：

```kotlin
// 1. 此時傳入的 Result 是 Result.success(Unit)
@InternalCoroutinesApi
public fun <T> Continuation<T>.resumeCancellableWith(
    result: Result<T>,
    onCancellation: ((cause: Throwable) -> Unit)? = null
): Unit = when (this) {
    is DispatchedContinuation -> resumeCancellableWith(result, onCancellation)
    else -> resumeWith(result)
}
```

由於 `this` 會是 **DispatchedContinuation**，所以 `resumeCancellableWith` 會是：

```kotlin
// We inline it to save an entry on the stack in cases where it shows (unconfined dispatcher)
// It is used only in Continuation<T>.resumeCancellableWith
@Suppress("NOTHING_TO_INLINE")
inline fun resumeCancellableWith(
    result: Result<T>,
    noinline onCancellation: ((cause: Throwable) -> Unit)?
) {
    // 1. 將 result 加上 onSuccess 與 onFailure 的行為
    //    至於要調用哪個方法就要通過 exceptionOrNull() 是否為 null 了
    val state = result.toState(onCancellation)

    // 2. isDispatchNeeded 預設為 true
    if (dispatcher.isDispatchNeeded(context)) {
        _state = state
        resumeMode = MODE_CANCELLABLE

        // 3. 調用 dispatcher 的 dispatch
        //    這要看 CoroutineDispatcher 的實作了
        dispatcher.dispatch(context, this)
    } else {
        executeUnconfined(state, MODE_CANCELLABLE) {
            if (!resumeCancelled(state)) {
                resumeUndispatchedWith(result)
            }
        }
    }
}

// 1. Any? 指的是 CompletedWithCancellation, CompleteExceptionally 或 it， onSuccess 的結果
//    此時 onCancellation == null，所以 onSuccess 時只會回傳 it
// 2. onCancellation 會在 DispatchedCoroutine 的 cancelCompletedResult 調用
//    而 onCancellation 的實質傳入會在 AbstractChannel 中以
//    resumeOnCancellationFun(value: E): ((Throwable) -> Unit)? 帶入。
internal fun <T> Result<T>.toState(
    onCancellation: ((cause: Throwable) -> Unit)? = null
): Any? = fold(
    onSuccess = { if (onCancellation != null) CompletedWithCancellation(it, onCancellation) else it },
    onFailure = { CompletedExceptionally(it) }
)
```


# CoroutineDispatcher

**CoroutineDispatcher** 是一個 **AbstractCoroutineContextElement** 也是一個 **ContinuationInterceptor**。

```kotlin
public abstract class CoroutineDispatcher :
    AbstractCoroutineContextElement(ContinuationInterceptor), ContinuationInterceptor
```

**AbstractCoroutineContextElement** 只是一個 **CoroutineContext.Element** 的 base class ：

```kotlin
@SinceKotlin("1.3")
public abstract class AbstractCoroutineContextElement(public override val key: Key<*>) : Element
```

我們可以從 **CoroutineDispatcher** 的建構子看見 **AbstractCoroutineContextElement** 的 `key` 被設為 `ContinuationInterceptor`。 這裡的 `ContinuationInterceptor` 是 **ContinuationInterceptor** 的 `key`。

**ContinuationInterceptor** 也是 **CoroutineContext.Element**，但它覆寫了 `operator get` 與 `minusKey` 來處理 **AbstractCoroutineContextKey**。 另外， **ContinuationInterceptor** 還新增了 `interceptContinuation(Continuation)` 並將 `releaseInterceptedContinuation(Continuation)` 預設為沒有行為：

```kotlin
@SinceKotlin("1.3")
public interface ContinuationInterceptor : CoroutineContext.Element {
    companion object Key : CoroutineContext.Key<ContinuationInterceptor>

    public fun <T> interceptContinuation(continuation: Continuation<T>): Continuation<T>
    public fun releaseInterceptedContinuation(continuation: Continuation<*>) {}

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

**CoroutineDispatcher** 會通過 **ContinuationInterceptor** 的 `interceptContinuation(Continuation)` 將 `continuation` 與 `this` 由 **DispatchedContinuation** 包起來：

```kotlin
public final override fun <T> interceptContinuation(continuation: Continuation<T>): Continuation<T> =
    DispatchedContinuation(this, continuation)
```

**DispatchedContinuation** 則是一個 **DispatchedTask** 或 **Runnable** 和 **Continuation**：

```kotlin
internal class DispatchedContinuation<in T>(
    @JvmField val dispatcher: CoroutineDispatcher,
    @JvmField val continuation: Continuation<T>
) : DispatchedTask<T>(MODE_UNINITIALIZED), CoroutineStackFrame, Continuation<T> by continuation
```

由於 **DispatchedContinuation** 是一個 **DispatchedTask** 所以我們還可以調用 `run`。 但這裡的行為則是 **DispatchedTask** 定義的：

```kotlin
public final override fun run() {
    assert { resumeMode != MODE_UNINITIALIZED } // should have been set before dispatching
    val taskContext = this.taskContext
    var fatalException: Throwable? = null
    try {
        val delegate = delegate as DispatchedContinuation<T>
        val continuation = delegate.continuation

        // 1. 存放 ThreadState 或 ThreadLocalElement 並調用 block()
        withContinuationContext(continuation, delegate.countOrElement) {

            // 2. 進入 block()
            val context = continuation.context

            // 3. takeState() 在不同類別會有不同行為
            //    - 若為 DispatchedContinuation 便會更新 _state 為 UNDEFINED 並回傳舊的 _state
            //    - 若為 CancellableContinuationImpl 便直接回傳 state
            val state = takeState() // NOTE: Must take state in any case, even if cancelled
            val exception = getExceptionalResult(state)

            /*
             * 4. Check whether continuation was originally resumed with an exception.
             * If so, it dominates cancellation, otherwise the original exception
             * will be silently lost.
             */
            val job = if (exception == null && resumeMode.isCancellableMode) context[Job] else null
            if (job != null && !job.isActive) {

                // 5. 若 job 沒在動，這表示它可能已經完成或被取消

                val cause = job.getCancellationException()

                // 6. cancelCompletedResult 會有兩種行為
                //    -> DispatchedContinuation - 如果 state 是 CompletedWithCancellation 那就會調用 state.onCancellation(cause) 進行指定行為。
                //                                 而 onCancellation: (cause: Throwable) -> Unit 會通過 internal fun <T> Result<T>.toState 傳入
                //    -> CancelledContinuationImpl - 若 state 是 CompletedContinuation 那就會更新 _state 並調用 state.invokeHandlers(this, cause) 來調用 CancellableContinuationImpl 的 callCancelHandler 與 callOnCancellation。
                cancelCompletedResult(state, cause)

                // 7. 調用 resumeWith(Result.failure(recoverStackTrace(exception, this)))
                //    這行為主要是用來記錄 debug
                continuation.resumeWithStackTrace(cause)
            } else {

                // 8. 若 job 為 null 或 active
                if (exception != null) {

                    // 9. 若 exception != null，那便調用 resumeWithException
                    continuation.resumeWithException(exception)
                } else {
                    // 10. 否則就調用 resume
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

從 `run` 源碼，我們會發現 `continuation` 是從 `delegate: DispatchedContinuation` 取得。

又因為 **DispatchedContinuation** 只有在 `CoroutineDispatcher.interceptContinuation(Continuation)` 中才會創建。 所以我們需要知道 `interceptContinuation(Continuation)` 是被誰調動的。

最後我們會發現，只有 **ContinuationImpl** 的 `interrupted()` 才會調動：

```kotlin
public fun intercepted(): Continuation<Any?> =
    intercepted
        ?: (context[ContinuationInterceptor]?.interceptContinuation(this) ?: this)
            .also { intercepted = it }
```

也就是說 `run` 中的 `continuation` 其實是 **ContinuationImpl**。 因此 `resumeWithException` 與 `resumed` 的行為是由 **Continuation** 定義：

```kotlin
@SinceKotlin("1.3")
@InlineOnly
public inline fun <T> Continuation<T>.resume(value: T): Unit =
    resumeWith(Result.success(value))

@SinceKotlin("1.3")
@InlineOnly
public inline fun <T> Continuation<T>.resumeWithException(exception: Throwable): Unit =
    resumeWith(Result.failure(exception))
```

`resumeWith` 的實作要看是哪個類別了。


## resumeWith 的行為

源碼中有實作 `resumeWith` 的有：
- **BaseContinuationImpl**
  這抽象類別中的 `resumeWith` 也會與它的子類別共用。 子類別包括： **ContinuationImpl**, **SafeCollector**, **SuspendLambda**, **RestrictedSuspendLambda** 與 **RestrictedContinuationImpl**。
- **SafeContinuation**
- **DeepRecursiveScope**
- **SequenceBuilderIterator**

### **BaseContinuationImpl**：

```kotlin
@SinceKotlin("1.3")
internal abstract class BaseContinuationImpl(
    public val completion: Continuation<Any?>?
) : Continuation<Any?>, CoroutineStackFrame, Serializable
```

**BaseContinuationImpl** 的 `resumeWith` 是最複雜的

```kotlin
public final override fun resumeWith(result: Result<Any?>) {
    // This loop unrolls recursion in current.resumeWith(param) to make saner and shorter stack traces on resume
    var current = this
    var param = result

    // 1. 進行無限監聽
    while (true) {
        // Invoke "resume" debug probe on every resumed continuation, so that a debugging library infrastructure
        // can precisely track what part of suspended callstack was already resumed
        probeCoroutineResumed(current)
        with(current) {
            val completion = completion!! // fail fast when trying to resume continuation without completion
            val outcome: Result<Any?> =
                try {

                    // 2. 調用 invokeSuspend，這個方法只有在 SafeCollector 才有實作。
                    //    其他的包括 SuspendLambda、 ContinuationImpl ... 都是抽象類別，而 SuspendLambda 的 invokeSuspend 則會由編輯器實作。
                    val outcome = invokeSuspend(param)

                    // 3. 如果是由 SafeCollector 定義，那 outcome 就一定是 COROUTINE_SUSPENDED
                    if (outcome === COROUTINE_SUSPENDED) return

                    // 4. 若 outcome !== COROUTINE_SUSPENDED 就表示完成了
                    //    所以就將 `outcome` 設為 Result.success
                    Result.success(outcome)
                } catch (exception: Throwable) {

                    // 5. 有其他問題就回傳 Result.failure
                    Result.failure(exception)
                }

            // 6. releaseIntercepted 在 CoroutineDispatcher 會調用 dispatched.release()
            //    dispatched 是 DispatchedContinuation，而 release() 則會調用 reusableCancellableContinuation?.detachChild()
            //    將自己從 parentHandle 移除
            releaseIntercepted() // this state machine instance is terminating

            if (completion is BaseContinuationImpl) {
                // unrolling recursion via loop
                // 7. 若 completion 是 BaseContinuationImpl 這表示還沒結束
                //    所以會更新 current 與 param 並繼續整個 loop
                current = completion
                param = outcome
            } else {
                // top-level completion reached -- invoke and return
                // 8. 若不是 BaseContinuationImpl 那就調用他的 resumeWith
                //    此時的 completion 可以是 SafeContinuation, DeepRecursiveScope 或 SequenceBuilderIterator
                completion.resumeWith(outcome)
                return
            }
        }
    }
}
```

### **SafeContinuation**

```kotlin
@PublishedApi
@SinceKotlin("1.3")
internal actual class SafeContinuation<in T>
internal actual constructor(
    private val delegate: Continuation<T>,
    initialResult: Any?
) : Continuation<T>, CoroutineStackFrame {
    @Volatile
    private var result: Any? = initialResult

    // ...
}
```

有趣的是 **SafeContinuation** 創建時會傳入一個 `delegate: Continuation`。 而這個 `delegate` 則會在 `resumeWith` 用到。

在 `resumeWith` 中會有以下行為：
1. 無限監控 `result: Any?`
   `result` 預設為 `initialResult`
2. 若 `this.result` 為 **UNDECIDED**，就更新成 `result.value` 並 return。
3. 若 `this.result` 為 **COROUTINE_SUSPENDED**，就更新成 **RESUMED** 並調用 `delegate` 的 `resumeWith`， 最後 return。
4. 否則，就拋出 **IllegalStateException**，因為此 **SafeContinuation** 必定是已 **RESUMED**。

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

想要知道 `delegate` 是哪個 **Continuation** 就要看 **SafeContinuation** 是如何被創建的。

#### SafeContinuation 的創建

**SafeContinuation** 會在三個地方被創建：
1. `(suspend () -> T).createCoroutine`
2. `(suspend R.() -> T).createCoroutine`
3. `suspendCoroutine`

前兩個方法其實很類似：

```kotlin
@SinceKotlin("1.3")
@Suppress("UNCHECKED_CAST")
public fun <T> (suspend () -> T).createCoroutine(
    completion: Continuation<T>
): Continuation<Unit> =
    SafeContinuation(createCoroutineUnintercepted(completion).intercepted(), COROUTINE_SUSPENDED)

@SinceKotlin("1.3")
@Suppress("UNCHECKED_CAST")
public fun <R, T> (suspend R.() -> T).createCoroutine(
    receiver: R,
    completion: Continuation<T>
): Continuation<Unit> =
    SafeContinuation(createCoroutineUnintercepted(receiver, completion).intercepted(), COROUTINE_SUSPENDED)
```

這兩個方法都會將 **SafeContinuation** 的 `initialResult` 設為 **COROUTINE_SUSPENDED**。 而它所需要的 `delegate` 則會由兩種方法取得：

```kotlin
createCoroutineUnintercepted(completion).intercepted()
createCoroutineUnintercepted(receiver, completion).intercepted()
```

這方法我們可以從 **IntrinsicsJvm.kt** 中找到：

```kotlin
/*
Creates unintercepted coroutine without receiver and with result type T.

This function creates a new, fresh instance of suspendable computation every time it is invoked.

- To start executing the created coroutine, invoke resume(Unit) on the returned Continuation instance.
- The completion continuation is invoked when coroutine completes with result or exception.

This function returns unintercepted continuation.

Invocation of resume(Unit) starts coroutine immediately in the invoker's call stack without going through the ContinuationInterceptor that might be present in the completion's CoroutineContext.

  It is the invoker's responsibility to ensure that a proper invocation context is established.

  Note that completion of this function may get invoked in an arbitrary context.

  Continuation.intercepted can be used to acquire the intercepted continuation.

Invocation of resume(Unit) on intercepted continuation guarantees that execution of both the coroutine and completion happens in the invocation context established by ContinuationInterceptor.

  Repeated invocation of any resume function on the resulting continuation corrupts the state machine of the coroutine and may result in arbitrary behaviour or exception.
*/


@SinceKotlin("1.3")
public actual fun <T> (suspend () -> T).createCoroutineUnintercepted(
    completion: Continuation<T>
): Continuation<Unit> {
    val probeCompletion = probeCoroutineCreated(completion)
    return if (this is BaseContinuationImpl)
        create(probeCompletion)
    else
        createCoroutineFromSuspendFunction(probeCompletion) {
            (this as Function1<Continuation<T>, Any?>).invoke(it)
        }
}

// Creates unintercepted coroutine with receiver type R and result type T.
@SinceKotlin("1.3")
public actual fun <R, T> (suspend R.() -> T).createCoroutineUnintercepted(
    receiver: R,
    completion: Continuation<T>
): Continuation<Unit> {
    val probeCompletion = probeCoroutineCreated(completion)
    return if (this is BaseContinuationImpl)
        create(receiver, probeCompletion)
    else {
        createCoroutineFromSuspendFunction(probeCompletion) {
            (this as Function2<R, Continuation<T>, Any?>).invoke(receiver, it)
        }
    }
}
```

`createCoroutineUnintercepted` 會有以下行為：
1. 通過 `probeCoroutineCreated` 來進行 debug 的監控
2. 當 `(suspend R.() -> T)` 或 `(suspend () -> T)` 是 **BaseContinuationImpl** 就會調用它的 `create(Any?, Continuation)` 或 `create(Continuation)`。
   之所以可能是 **BaseContinuationImpl** 是因為 `suspend ()` 或 `suspend R.()` 其實都會傳入 **Continuation**。 所以他們另一個寫法是：

   ```kotlin
   suspend () -> T ==> (this, Continuation) -> T
   suspend R.() -> T ==> (this, R, Continuation) -> T
   ```

   所以我們才可以調用 `this`。
3. 反之，便調用 `createCoroutineFromSuspendFunction` 將 `probeCompletion` 由 **RestrictedContinuationImpl** 或 **ContinuationImpl** 包裹起來。

   <br>
   而其覆寫的 `invokeSuspend` 則會調用以下：
   <br>

   ```kotlin
   (this as Function1<Continuation<T>, Any?>).invoke(it)
   // 或
   (this as Function2<R, Continuation<T>, Any?>).invoke(receiver, it)
   ```

## dispatch 的行為

- **DefaultIoScheduler** \[Singleton\]
  這是 **Dispatchers.IO**。

  它本身是一個 **ExecutorCoroutineDispatcher** 且 **Executor**。

  它的 `dispatch` 與 `dispatchYield` 行為是通過 **UnlimitedIoScheduler** 來完成的。

- **UnlimitedIoScheduler** \[Singleton\]
  雖然這是 **CoroutineDispatcher**，但它的 `dispatch` 與 `dispatchYield` 都是由 **DefaultScheduler** 完成的。

- **DefaultScheduler** \[Singleton\]
  這是 **Dispatches.Default**。

  它繼承了 **SchedulerCoroutineDispatcher**，且全部行為皆由 **SchedulerCoroutineDispatcher** 完成。

  但它還多了 `shutdown` 方法 並 覆寫了 `close` 方法來確保不會被關閉。

- **SchedulerCoroutineDispatcher**
  這是一個可以讓我們選擇 `corePoolSize`、 `maxPoolSize`、 `idleWorkerKeepAliveNs` 與 `schedulerName` 的 **ExecutorCoroutineDispatcher**。

  其實這些參數都是用來建立 **CoroutineScheduler**， 一個 **Executor** 。

  而它的 `dispatch`、 `dispatchYield`、 `close` 與 `dispatchWithContext` 都由 `coroutineScheduler` 執行。



- **Unconfined**

- **EventLoopImplBase**
- **ExecutorCoroutineDispatcherImpl**
- **ExperimentalCoroutineDispatcher**
- **HandlerContext**
- **LimitedDispatcher**
- **LimitingDispatcher**
- **MissingMainCoroutineDispatcher**
- **PausingDispatcher**








# Executor

```kotlin
public interface Executor {
    void execute(Runnable command);
}
```

我們只看與 **CoroutineDispatcher** 相關的 **Executor**。

## CoroutineScheduler
```kotlin
@Suppress("NOTHING_TO_INLINE")
internal class CoroutineScheduler(
    @JvmField val corePoolSize: Int,
    @JvmField val maxPoolSize: Int,
    @JvmField val idleWorkerKeepAliveNs: Long = IDLE_WORKER_KEEP_ALIVE_NS,
    @JvmField val schedulerName: String = DEFAULT_SCHEDULER_NAME // "DefaultDispatcher"
) : Executor, Closeable
```

由於 **SchedulerCoroutineDispatcher** 的 `dispatch`、 `dispatchYield` 與 `dispatchWithContext` 都是由 **CoroutineScheduler** 實作，所以我們就看看 **CoroutineScheduler** 的 `dispatch`：

```kotlin
override fun dispatch(context: CoroutineContext, block: Runnable): Unit = coroutineScheduler.dispatch(block)

override fun dispatchYield(context: CoroutineContext, block: Runnable): Unit =
    coroutineScheduler.dispatch(block, tailDispatch = true)

internal fun dispatchWithContext(block: Runnable, context: TaskContext, tailDispatch: Boolean) {
    coroutineScheduler.dispatch(block, context, tailDispatch)
}
```

另外，**TaskContext** 有兩種：

```kotlin
@JvmField
internal val NonBlockingContext: TaskContext = TaskContextImpl(TASK_NON_BLOCKING)

@JvmField
internal val BlockingContext: TaskContext = TaskContextImpl(TASK_PROBABLY_BLOCKING)
```

### dispatch

```kotlin
fun dispatch(block: Runnable, taskContext: TaskContext = NonBlockingContext, tailDispatch: Boolean = false) {
    trackTask() // this is needed for virtual time support

    // 1. 將 block: Runnable 包裹在 Task 中
    val task = createTask(block, taskContext)

    // try to submit the task to the local queue and act depending on the result
    // 2. 若 currentThread 不是 Worker，那就會回傳 null
    val currentWorker = currentWorker()

    // 3. 嘗試將 task 放入 Worker 的 localQueue: WorkQueue
    //    如果 state 是 WorkerState.TERMINATED 或
    //    task.mode == TASK_NON_BLOCKING && state === WorkerState.BLOCKING (CPU 在忙) 時，
    //    那就回傳 task
    //    否則，就將 task 放入 localQueue 便成功時回傳 null 否則回傳 task
    val notAdded = currentWorker.submitToLocalQueue(task, tailDispatch)

    // 4. 若加入失敗，便嘗試將 task 加入 GlobalQueue。
    //    若 task.isBlocking，那 task 就會放入 globalBlockingQueue : GlobalQueue
    //    否則就放入 globalCpuQueue: GlobalQueue
    //    GlobalQueue : LockFreeTaskQueue<Task>(singleConsumer = false)
    if (notAdded != null) {
        if (!addToGlobalQueue(notAdded)) {
            // Global queue is closed in the last step of close/shutdown -- no more tasks should be accepted
            // 5. 若沒有成功，就拋出 RejectedExecutionException
            throw RejectedExecutionException("$schedulerName was terminated")
        }
    }

    val skipUnpark = tailDispatch && currentWorker != null
    // Checking 'task' instead of 'notAdded' is completely okay
    if (task.mode == TASK_NON_BLOCKING) {
        if (skipUnpark) return
        signalCpuWork()
    } else {
        // Increment blocking tasks anyway
        signalBlockingWork(skipUnpark = skipUnpark)
    }
}
```

### execute
```kotlin
override fun execute(command: Runnable) = dispatch(command)
```












<br><br><br><br><br><br><br><br><br>
