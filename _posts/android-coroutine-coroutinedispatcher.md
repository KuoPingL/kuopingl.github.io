---
layout: post
title:  Kotlinx.Coroutine - 從 CoroutineDispatcher 來看 Coroutine
date:   2022-08-08 16:15:07 +0800
categories: [coroutine, android, intermediate]
---

# CoroutineDispatcher

**CoroutineDispatcher** 是一個同時繼承 **AbstractCoroutineContextElement** 和 **ContinuationInterceptor** 的抽象類別。

由於 **CoroutineDispatcher** 的 `key` 是 **ContinuationInterceptor**，所以我們可以通過 `context[ContinuationInterceptor]` 取得 **CoroutineDispatcher**。

```kotlin
public abstract class CoroutineDispatcher :
    AbstractCoroutineContextElement(ContinuationInterceptor), ContinuationInterceptor
```

**AbstractCoroutineContextElement** 定義了它的 **CoroutineContext.Element** 特性，。

```kotlin
@SinceKotlin("1.3")
public abstract class AbstractCoroutineContextElement(public override val key: Key<*>) : Element
```

而 **ContinuationInterceptor** 則定義了它的行為：

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

**ContinuationInterceptor** 雖然也是 **CoroutineContext.Element**，但它覆寫了 `operator get` 與 `minusKey` 來處理 **AbstractCoroutineContextKey**。

另外， **ContinuationInterceptor** 還新增了 `interceptContinuation(Continuation)` 並將 `releaseInterceptedContinuation(Continuation)` 預設為沒有行為。

><br>
>
> **ContinuationInterceptor** 的主要目的是為了讓
><br>


而 **CoroutineDispatcher** 的 `interceptContinuation(Continuation)` 則會將 `continuation` 與 `this` 由 **DispatchedContinuation** 包起來：

```kotlin
public final override fun <T> interceptContinuation(continuation: Continuation<T>): Continuation<T> =
    DispatchedContinuation(this, continuation)
```

**DispatchedContinuation** 則是一個 **DispatchedTask** 或 **Runnable** 和 **Continuation**。 但是他的 **Continuation** 行為卻是由 `continuation` 實作：

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








也會通過 `suspendCoroutine` 中創建：
```kotlin
@SinceKotlin("1.3")
@InlineOnly
public suspend inline fun <T> suspendCoroutine(crossinline block: (Continuation<T>) -> Unit): T {
    contract { callsInPlace(block, InvocationKind.EXACTLY_ONCE) }
    return suspendCoroutineUninterceptedOrReturn { c: Continuation<T> ->
        val safe = SafeContinuation(c.intercepted())
        block(safe)
        safe.getOrThrow()
    }
}
```

### DeepRecursiveScopeImpl
```kotlin
@Suppress("UNCHECKED_CAST")
private class DeepRecursiveScopeImpl<T, R>(
    block: suspend DeepRecursiveScope<T, R>.(T) -> R,
    value: T
) : DeepRecursiveScope<T, R>(), Continuation<R>
```


```kotlin

private typealias State = Int

private const val State_NotReady: State = 0
private const val State_ManyNotReady: State = 1
private const val State_ManyReady: State = 2
private const val State_Ready: State = 3
private const val State_Done: State = 4
private const val State_Failed: State = 5

override fun resumeWith(result: Result<R>) {
    this.cont = null
    this.result = result
}
```


### SequenceBuilderIterator
```kotlin
private class SequenceBuilderIterator<T> : SequenceScope<T>(), Iterator<T>, Continuation<Unit>
```

```kotlin
// Completion continuation implementation
override fun resumeWith(result: Result<Unit>) {
    result.getOrThrow() // just rethrow exception if it is there
    state = State_Done
}
```





經由 **Continuation** 的特性，它需要覆寫 `resumeWith`。 而且還附加了幾個 resume 相關的方法： `resumeCancellableWith` 、 `resumeUndispatchedWith` 與 `resumeCancelled`。

```kotlin
override fun resumeWith(result: Result<T>) {
    val context = continuation.context
    val state = result.toState()

    // 若需要 dispatch 才會調用 dispatcher.dispatch
    if (dispatcher.isDispatchNeeded(context)) {
        _state = state
        resumeMode = MODE_ATOMIC
        dispatcher.dispatch(context, this)
    } else {
        executeUnconfined(state, MODE_ATOMIC) {
            withCoroutineContext(this.context, countOrElement) {
                continuation.resumeWith(result)
            }
        }
    }
}

@Suppress("NOTHING_TO_INLINE")
inline fun resumeCancellableWith(
    result: Result<T>,
    noinline onCancellation: ((cause: Throwable) -> Unit)?
) {
    val state = result.toState(onCancellation)
    if (dispatcher.isDispatchNeeded(context)) {
        _state = state
        resumeMode = MODE_CANCELLABLE
        dispatcher.dispatch(context, this)
    } else {
        executeUnconfined(state, MODE_CANCELLABLE) {
            if (!resumeCancelled(state)) {
                resumeUndispatchedWith(result)
            }
        }
    }
}

@Suppress("NOTHING_TO_INLINE") // we need it inline to save us an entry on the stack
inline fun resumeUndispatchedWith(result: Result<T>) {
    withContinuationContext(continuation, countOrElement) {
        continuation.resumeWith(result)
    }
}

@Suppress("NOTHING_TO_INLINE")
inline fun resumeCancelled(state: Any?): Boolean {
    val job = context[Job]
    if (job != null && !job.isActive) {
        val cause = job.getCancellationException()
        cancelCompletedResult(state, cause)
        resumeWithException(cause)
        return true
    }
    return false
}
```





```kotlin
public abstract class CoroutineDispatcher :
    AbstractCoroutineContextElement(ContinuationInterceptor), ContinuationInterceptor {

    @ExperimentalStdlibApi
    public companion object Key : AbstractCoroutineContextKey<ContinuationInterceptor, CoroutineDispatcher>(
        ContinuationInterceptor,
        { it as? CoroutineDispatcher })

    public open fun isDispatchNeeded(context: CoroutineContext): Boolean = true

    @ExperimentalCoroutinesApi
    public open fun limitedParallelism(parallelism: Int): CoroutineDispatcher {
        parallelism.checkParallelism()
        return LimitedDispatcher(this, parallelism)
    }

    public abstract fun dispatch(context: CoroutineContext, block: Runnable)

    @InternalCoroutinesApi
    public open fun dispatchYield(context: CoroutineContext, block: Runnable): Unit = dispatch(context, block)

    public final override fun <T> interceptContinuation(continuation: Continuation<T>): Continuation<T> =
        DispatchedContinuation(this, continuation)

    public final override fun releaseInterceptedContinuation(continuation: Continuation<*>) {
        val dispatched = continuation as DispatchedContinuation<*>
        dispatched.release()
    }

    override fun toString(): String = "$classSimpleName@$hexAddress"
}
```

- **EventLoop**
- **ExecutorCoroutineDispatcher**
- **MainCoroutineDispatcher**
-

## EventLoop
```kotlin
internal abstract class EventLoop : CoroutineDispatcher() {
    /**
     * Counts the number of nested `runBlocking` and [Dispatchers.Unconfined] that use this event loop.
     */
    private var useCount = 0L

    /**
     * Set to true on any use by `runBlocking`, because it potentially leaks this loop to other threads, so
     * this instance must be properly shutdown. We don't need to shutdown event loop that was used solely
     * by [Dispatchers.Unconfined] -- it can be left as [ThreadLocalEventLoop] and reused next time.
     */
    private var shared = false

    /**
     * Queue used by [Dispatchers.Unconfined] tasks.
     * These tasks are thread-local for performance and take precedence over the rest of the queue.
     */
    private var unconfinedQueue: ArrayQueue<DispatchedTask<*>>? = null

    /**
     * Processes next event in this event loop.
     *
     * The result of this function is to be interpreted like this:
     * * `<= 0` -- there are potentially more events for immediate processing;
     * * `> 0` -- a number of nanoseconds to wait for next scheduled event;
     * * [Long.MAX_VALUE] -- no more events.
     *
     * **NOTE**: Must be invoked only from the event loop's thread
     *          (no check for performance reasons, may be added in the future).
     */
    public open fun processNextEvent(): Long {
        if (!processUnconfinedEvent()) return Long.MAX_VALUE
        return 0
    }

    protected open val isEmpty: Boolean get() = isUnconfinedQueueEmpty

    protected open val nextTime: Long
        get() {
            val queue = unconfinedQueue ?: return Long.MAX_VALUE
            return if (queue.isEmpty) Long.MAX_VALUE else 0L
        }

    public fun processUnconfinedEvent(): Boolean {
        val queue = unconfinedQueue ?: return false
        val task = queue.removeFirstOrNull() ?: return false
        task.run()
        return true
    }
    /**
     * Returns `true` if the invoking `runBlocking(context) { ... }` that was passed this event loop in its context
     * parameter should call [processNextEvent] for this event loop (otherwise, it will process thread-local one).
     * By default, event loop implementation is thread-local and should not processed in the context
     * (current thread's event loop should be processed instead).
     */
    public open fun shouldBeProcessedFromContext(): Boolean = false

    /**
     * Dispatches task whose dispatcher returned `false` from [CoroutineDispatcher.isDispatchNeeded]
     * into the current event loop.
     */
    public fun dispatchUnconfined(task: DispatchedTask<*>) {
        val queue = unconfinedQueue ?:
            ArrayQueue<DispatchedTask<*>>().also { unconfinedQueue = it }
        queue.addLast(task)
    }

    public val isActive: Boolean
        get() = useCount > 0

    public val isUnconfinedLoopActive: Boolean
        get() = useCount >= delta(unconfined = true)

    // May only be used from the event loop's thread
    public val isUnconfinedQueueEmpty: Boolean
        get() = unconfinedQueue?.isEmpty ?: true

    private fun delta(unconfined: Boolean) =
        if (unconfined) (1L shl 32) else 1L

    fun incrementUseCount(unconfined: Boolean = false) {
        useCount += delta(unconfined)
        if (!unconfined) shared = true
    }

    fun decrementUseCount(unconfined: Boolean = false) {
        useCount -= delta(unconfined)
        if (useCount > 0) return
        assert { useCount == 0L } // "Extra decrementUseCount"
        if (shared) {
            // shut it down and remove from ThreadLocalEventLoop
            shutdown()
        }
    }

    final override fun limitedParallelism(parallelism: Int): CoroutineDispatcher {
        parallelism.checkParallelism()
        return this
    }

    open fun shutdown() {}
}

@ThreadLocal
internal object ThreadLocalEventLoop {
    private val ref = CommonThreadLocal<EventLoop?>()

    internal val eventLoop: EventLoop
        get() = ref.get() ?: createEventLoop().also { ref.set(it) }

    internal fun currentOrNull(): EventLoop? =
        ref.get()

    internal fun resetEventLoop() {
        ref.set(null)
    }

    internal fun setEventLoop(eventLoop: EventLoop) {
        ref.set(eventLoop)
    }
}
```

### EventLoopImplPlatform
```kotlin
internal actual abstract class EventLoopImplPlatform: EventLoop() {
    protected abstract val thread: Thread

    protected actual fun unpark() {
        val thread = thread // atomic read
        if (Thread.currentThread() !== thread)
            unpark(thread)
    }

    protected actual open fun reschedule(now: Long, delayedTask: EventLoopImplBase.DelayedTask) {
        DefaultExecutor.schedule(now, delayedTask)
    }
}
```

### EventLoopImplBase

```kotlin
internal abstract class EventLoopImplBase: EventLoopImplPlatform(), Delay
```



### BlockingEventLoop

```kotlin
internal class BlockingEventLoop(
    override val thread: Thread
) : EventLoopImplBase()
```

```kotlin
internal actual fun createEventLoop(): EventLoop = BlockingEventLoop(Thread.currentThread())
```


```kotlin
@ThreadLocal
internal object ThreadLocalEventLoop {

    // 這是 ThreadLocal，所以 eventLoop 會在這個 Thread 中使用
    private val ref = CommonThreadLocal<EventLoop?>()

    internal val eventLoop: EventLoop
        get() = ref.get() ?: createEventLoop().also { ref.set(it) }

    internal fun currentOrNull(): EventLoop? =
        ref.get()

    internal fun resetEventLoop() {
        ref.set(null)
    }

    internal fun setEventLoop(eventLoop: EventLoop) {
        ref.set(eventLoop)
    }
}
```

- **DispatchedContinuation.executeUnconfined()**
- **DispatchedTask.resumeUnConfined()**
- **Builder.runBlocking**




```kotlin
private inline fun DispatchedContinuation<*>.executeUnconfined(
    contState: Any?, mode: Int, doYield: Boolean = false,
    block: () -> Unit
): Boolean {
    assert { mode != MODE_UNINITIALIZED } // invalid execution mode
    val eventLoop = ThreadLocalEventLoop.eventLoop
    // If we are yielding and unconfined queue is empty, we can bail out as part of fast path
    if (doYield && eventLoop.isUnconfinedQueueEmpty) return false
    return if (eventLoop.isUnconfinedLoopActive) {
        // When unconfined loop is active -- dispatch continuation for execution to avoid stack overflow
        _state = contState
        resumeMode = mode
        eventLoop.dispatchUnconfined(this)
        true // queued into the active loop
    } else {
        // Was not active -- run event loop until all unconfined tasks are executed
        runUnconfinedEventLoop(eventLoop, block = block)
        false
    }
}
```


```kotlin
private fun DispatchedTask<*>.resumeUnconfined() {
    val eventLoop = ThreadLocalEventLoop.eventLoop
    if (eventLoop.isUnconfinedLoopActive) {
        // When unconfined loop is active -- dispatch continuation for execution to avoid stack overflow
        eventLoop.dispatchUnconfined(this)
    } else {
        // Was not active -- run event loop until all unconfined tasks are executed
        runUnconfinedEventLoop(eventLoop) {
            resume(delegate, undispatched = true)
        }
    }
}
```


```kotlin
@Throws(InterruptedException::class)
public actual fun <T> runBlocking(context: CoroutineContext, block: suspend CoroutineScope.() -> T): T {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    val currentThread = Thread.currentThread()
    val contextInterceptor = context[ContinuationInterceptor]
    val eventLoop: EventLoop?
    val newContext: CoroutineContext
    if (contextInterceptor == null) {
        // create or use private event loop if no dispatcher is specified
        eventLoop = ThreadLocalEventLoop.eventLoop
        newContext = GlobalScope.newCoroutineContext(context + eventLoop)
    } else {
        // See if context's interceptor is an event loop that we shall use (to support TestContext)
        // or take an existing thread-local event loop if present to avoid blocking it (but don't create one)
        eventLoop = (contextInterceptor as? EventLoop)?.takeIf { it.shouldBeProcessedFromContext() }
            ?: ThreadLocalEventLoop.currentOrNull()
        newContext = GlobalScope.newCoroutineContext(context)
    }
    val coroutine = BlockingCoroutine<T>(newContext, currentThread, eventLoop)
    coroutine.start(CoroutineStart.DEFAULT, coroutine, block)
    return coroutine.joinBlocking()
}
```



### DefaultExecutor


## ExecutorCoroutineDispatcher

- **CommonPool**
- **DefaultIoScheduler**

### ExecutorCoroutineDispatcherImpl

### DefaultIoScheduler

### SchedulerCoroutineDispatcher

#### DefaultScheduler

### LimitingDispatcher



## MainCoroutineDispatcher


## Unconfined
```kotlin
internal object Unconfined : CoroutineDispatcher() {
    @ExperimentalCoroutinesApi
    override fun limitedParallelism(parallelism: Int): CoroutineDispatcher {
        throw UnsupportedOperationException("limitedParallelism is not supported for Dispatchers.Unconfined")
    }

    override fun isDispatchNeeded(context: CoroutineContext): Boolean = false
}
```

```kotlin
override fun dispatch(context: CoroutineContext, block: Runnable) {
    /** It can only be called by the [yield] function. See also code of [yield] function. */
    val yieldContext = context[YieldContext]
    if (yieldContext != null) {
        // report to "yield" that it is an unconfined dispatcher and don't call "block.run()"
        yieldContext.dispatcherWasUnconfined = true
        return
    }
    throw UnsupportedOperationException("Dispatchers.Unconfined.dispatch function can only be used by the yield function. " +
        "If you wrap Unconfined dispatcher in your code, make sure you properly delegate " +
        "isDispatchNeeded and dispatch calls.")
}
```

```kotlin
@PublishedApi
internal class YieldContext : AbstractCoroutineContextElement(Key) {
    companion object Key : CoroutineContext.Key<YieldContext>

    @JvmField
    var dispatcherWasUnconfined = false
}
```


# Dispatchers
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

## launch
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

        // 3. 調用 dispatch
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



## withContext

`withContext` 會按不同的情況創建出三種不同 Coroutine：
- 若 suspend 傳入的 **Continuation** 的 `uCont.context` 與 傳入的 `context` 相同
  建立 **ScopeCoroutine**
- 若兩個 `context` 中的 **ContinuationInterceptor** 相同，表示 **Dispatcher** 保持一樣。
  建立 **UndispatchedCoroutine**
- 若都不同
  建立 **DispatchedCoroutine**


```kotlin
internal open class ScopeCoroutine<in T>(
    context: CoroutineContext,
    @JvmField val uCont: Continuation<T> // unintercepted continuation
) : AbstractCoroutine<T>(context, true, true), CoroutineStackFrame

// Used by withContext when context changes, but dispatcher stays the same
internal actual class UndispatchedCoroutine<in T>actual constructor (
    context: CoroutineContext,
    uCont: Continuation<T>
) : ScopeCoroutine<T>(if (context[UndispatchedMarker] == null) context + UndispatchedMarker else context, uCont)

// Used by withContext when context dispatcher changes
internal class DispatchedCoroutine<in T>(
    context: CoroutineContext,
    uCont: Continuation<T>
) : ScopeCoroutine<T>(context, uCont)
```


```kotlin
// Builder.common.kt
public suspend fun <T> withContext(
    context: CoroutineContext,
    block: suspend CoroutineScope.() -> T
): T {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return suspendCoroutineUninterceptedOrReturn sc@ { uCont ->

        // 1. 進入 block

        // compute new context
        val oldContext = uCont.context

        // 2. 通過 foldCopies 將 context 放在 oldContext 的 copyable coroutineContext 後面
        // Copy CopyableThreadContextElement if necessary
        val newContext = oldContext.newCoroutineContext(context)

        // 3. 確定 newContext 為 active
        // always check for cancellation of new context
        newContext.ensureActive()

        // 4. 若 context 沒變，
        //      那就創建 ScopeCoroutine
        //    並將 coroutine.startUndispatchedOrReturn 的結果回傳
        // FAST PATH #1 -- new context is the same as the old one
        if (newContext === oldContext) {
            val coroutine = ScopeCoroutine(newContext, uCont)
            return@sc coroutine.startUndispatchedOrReturn(coroutine, block)
        }

        // 5. 若兩個 context 共用一個 ContinuationInterceptor，
        //      那就創建 UndispatchedCoroutine
        //    並調用 withCoroutineContext 來更新當下 context
        //    最後就會傳回 block() 的結果
        // FAST PATH #2 -- the new dispatcher is the same as the old one (something else changed)
        // `equals` is used by design (see equals implementation is wrapper context like ExecutorCoroutineDispatcher)
        if (newContext[ContinuationInterceptor] == oldContext[ContinuationInterceptor]) {
            val coroutine = UndispatchedCoroutine(newContext, uCont)
            // There are changes in the context, so this thread needs to be updated
            withCoroutineContext(newContext, null) {
                return@sc coroutine.startUndispatchedOrReturn(coroutine, block)
            }
        }

        // 6. 否則，就創建 DispatchedCoroutine
        //      並通過 `block.startCoroutineCancellable`
        // SLOW PATH -- use new dispatcher
        val coroutine = DispatchedCoroutine(newContext, uCont)
        block.startCoroutineCancellable(coroutine, coroutine)
        coroutine.getResult()
    }
}
```

`suspendCoroutineUninterceptedOrReturn`

```kotlin
@SinceKotlin("1.3")
@InlineOnly
@Suppress("UNUSED_PARAMETER", "RedundantSuspendModifier")
public suspend inline fun <T> suspendCoroutineUninterceptedOrReturn(crossinline block: (Continuation<T>) -> Any?): T {
    contract { callsInPlace(block, InvocationKind.EXACTLY_ONCE) }
    throw NotImplementedError("Implementation of suspendCoroutineUninterceptedOrReturn is intrinsic")
}
```

`withCoroutineContext` 只是更新 context 並調用 `block`：
```kotlin
/**
 * Executes a block using a given coroutine context.
 */
internal actual inline fun <T> withCoroutineContext(context: CoroutineContext, countOrElement: Any?, block: () -> T): T {
    val oldValue = updateThreadContext(context, countOrElement)
    try {
        return block()
    } finally {
        restoreThreadContext(context, oldValue)
    }
}
```

`startCoroutineCancellable`

```kotlin
internal fun <R, T> (suspend (R) -> T).startCoroutineCancellable(
    receiver: R, completion: Continuation<T>,
    onCancellation: ((cause: Throwable) -> Unit)? = null
) =
    runSafely(completion) {
        createCoroutineUnintercepted(receiver, completion)
          .intercepted()
          .resumeCancellableWith(Result.success(Unit), onCancellation)
    }


private inline fun runSafely(completion: Continuation<*>, block: () -> Unit) {
    try {
        block()
    } catch (e: Throwable) {
        dispatcherFailure(completion, e)
    }
}

@InternalCoroutinesApi
public fun <T> Continuation<T>.resumeCancellableWith(
    result: Result<T>,
    onCancellation: ((cause: Throwable) -> Unit)? = null
): Unit = when (this) {
    is DispatchedContinuation -> resumeCancellableWith(result, onCancellation)
    else -> resumeWith(result)
}
```


## runBlocking
```kotlin
@Throws(InterruptedException::class)
public actual fun <T> runBlocking(context: CoroutineContext, block: suspend CoroutineScope.() -> T): T {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    val currentThread = Thread.currentThread()
    val contextInterceptor = context[ContinuationInterceptor]
    val eventLoop: EventLoop?
    val newContext: CoroutineContext

    if (contextInterceptor == null) {
        // create or use private event loop if no dispatcher is specified
        eventLoop = ThreadLocalEventLoop.eventLoop
        newContext = GlobalScope.newCoroutineContext(context + eventLoop)
    } else {
        // See if context's interceptor is an event loop that we shall use (to support TestContext)
        // or take an existing thread-local event loop if present to avoid blocking it (but don't create one)
        eventLoop = (contextInterceptor as? EventLoop)?.takeIf { it.shouldBeProcessedFromContext() }
            ?: ThreadLocalEventLoop.currentOrNull()
        newContext = GlobalScope.newCoroutineContext(context)
    }
    val coroutine = BlockingCoroutine<T>(newContext, currentThread, eventLoop)
    coroutine.start(CoroutineStart.DEFAULT, coroutine, block)
    return coroutine.joinBlocking()
}
```















<br><br><br><br><br><br><br><br><br>
