---
layout: post
title:  Kotlinx.Coroutine - 從 CoroutineScope 來暸解 Coroutine
date:   2022-08-08 16:15:07 +0800
categories: [coroutine, android, intermediate]
---
# 序文


# CoroutineScope

```kotlin
public interface CoroutineScope {
    public val coroutineContext: CoroutineContext
}
```


- **AbstractCoroutine**
- **ActorScope**


```kotlin
@Suppress("FunctionName")
public fun MainScope(): CoroutineScope = ContextScope(SupervisorJob() + Dispatchers.Main)
```




## GlobalScope

```kotlin
@DelicateCoroutinesApi
public object GlobalScope : CoroutineScope {
    override val coroutineContext: CoroutineContext
        get() = EmptyCoroutineContext
}
```

以下是 **GlobalScope** 的講解：

```kotlin
/*
A global CoroutineScope not bound to any job.

Global scope is used to launch top-level coroutines which are operating on the whole application lifetime and are not cancelled prematurely.

Active coroutines launched in GlobalScope do not keep the process alive. They are like daemon threads.
[ daemon thread == background thread ]

This is a delicate API. It is easy to accidentally create resource or memory leaks when GlobalScope is used.

A coroutine launched in GlobalScope is not subject to the principle of structured concurrency, so if it hangs or gets delayed due to a problem (e.g. due to a slow network), it will stay working and consuming resources. For example, consider the following code:
*/

fun loadConfiguration() {
    GlobalScope.launch {
        val config = fetchConfigFromServer() // network request
        updateConfiguration(config)
    }
}

/*
A call to loadConfiguration creates a coroutine in the GlobalScope that works in background without any provision to cancel it or to wait for its completion. If a network is slow, it keeps waiting in background, consuming resources. Repeated calls to loadConfiguration will consume more and more resources.
Possible replacements
In many cases uses of GlobalScope should be removed, marking the containing operation with suspend, for example:
*/

suspend fun loadConfiguration() {
    val config = fetchConfigFromServer() // network request
    updateConfiguration(config)
}

/*
In cases when GlobalScope.launch was used to launch multiple concurrent operations, the corresponding operations shall be grouped with coroutineScope instead:
*/

// concurrently load configuration and data
suspend fun loadConfigurationAndData() {
    coroutineScope {
        launch { loadConfiguration() }
        launch { loadData() }
    }
}

/*
In top-level code, when launching a concurrent operation from a non-suspending context, an appropriately confined instance of CoroutineScope shall be used instead of a GlobalScope. See docs on CoroutineScope for details.
GlobalScope vs custom scope

Do not replace GlobalScope.launch { ... } with CoroutineScope().launch { ... } constructor function call. The latter has the same pitfalls as GlobalScope. See CoroutineScope documentation on the intended usage of CoroutineScope() constructor function.
Legitimate use-cases
There are limited circumstances under which GlobalScope can be legitimately and safely used, such as top-level background processes that must stay active for the whole duration of the application's lifetime. Because of that, any use of GlobalScope requires an explicit opt-in with @OptIn(DelicateCoroutinesApi::class), like this:
*/
// A global coroutine to log statistics every second, must be always active
@OptIn(DelicateCoroutinesApi::class)
val globalScopeReporter = GlobalScope.launch {
    while (true) {
        delay(1000)
        logStatistics()
    }
}
```

在源碼中使用到 **GlobalScope** 的包括：
- **


## AbstractCoroutine

### ChannelCoroutine

### BlockingCoroutine

### StandaloneCoroutine
```kotlin
private open class StandaloneCoroutine(
    parentContext: CoroutineContext,
    active: Boolean
) : AbstractCoroutine<Unit>(parentContext, initParentJob = true, active = active) {
    override fun handleJobException(exception: Throwable): Boolean {
        handleCoroutineException(context, exception)
        return true
    }
}
```

#### LazyStandaloneCoroutine
```kotlin
private class LazyStandaloneCoroutine(
    parentContext: CoroutineContext,
    block: suspend CoroutineScope.() -> Unit
) : StandaloneCoroutine(parentContext, active = false) {
    private val continuation = block.createCoroutineUnintercepted(this, this)

    override fun onStart() {
        continuation.startCoroutineCancellable(this)
    }
}
```

#### CoroutineScope.launch
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




### DeferredCoroutine


#### LazyDeferredCoroutine

#### CoroutineScope.async

```kotlin
public fun <T> CoroutineScope.async(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> T
): Deferred<T> {
    val newContext = newCoroutineContext(context)
    val coroutine = if (start.isLazy)
        LazyDeferredCoroutine(newContext, block) else
        DeferredCoroutine<T>(newContext, active = true)
    coroutine.start(start, coroutine, block)
    return coroutine
}
```


### ScopeCoroutine

```kotlin
/**
 * This is a coroutine instance that is created by [coroutineScope] builder.
 */
internal open class ScopeCoroutine<in T>(
    context: CoroutineContext,
    @JvmField val uCont: Continuation<T> // unintercepted continuation
) : AbstractCoroutine<T>(context, true, true), CoroutineStackFrame {

    final override val callerFrame: CoroutineStackFrame? get() = uCont as? CoroutineStackFrame
    final override fun getStackTraceElement(): StackTraceElement? = null

    final override val isScopedCoroutine: Boolean get() = true
    internal val parent: Job? get() = parentHandle?.parent

    override fun afterCompletion(state: Any?) {
        // Resume in a cancellable way by default when resuming from another context
        uCont.intercepted().resumeCancellableWith(recoverResult(state, uCont))
    }

    override fun afterResume(state: Any?) {
        // Resume direct because scope is already in the correct context
        uCont.resumeWith(recoverResult(state, uCont))
    }
}
```

#### UndispatchedCoroutine
```kotlin
// Used by withContext when context changes, but dispatcher stays the same
internal actual class UndispatchedCoroutine<in T>actual constructor (
    context: CoroutineContext,
    uCont: Continuation<T>
) : ScopeCoroutine<T>(if (context[UndispatchedMarker] == null) context + UndispatchedMarker else context, uCont) {
    /*
    * The state is thread-local because this coroutine can be used concurrently.
    * Scenario of usage (withContinuationContext):
    * val state = saveThreadContext(ctx)
    * try {
    *     invokeSmthWithThisCoroutineAsCompletion() // Completion implies that 'afterResume' will be called
    *     // COROUTINE_SUSPENDED is returned
    * } finally {
    *     thisCoroutine().clearThreadContext() // Concurrently the "smth" could've been already resumed on a different thread
    *     // and it also calls saveThreadContext and clearThreadContext
    * }
    */
   private var threadStateToRecover = ThreadLocal<Pair<CoroutineContext, Any?>>()

   init {
       /*
        * This is a hack for a very specific case in #2930 unless #3253 is implemented.
        * 'ThreadLocalStressTest' covers this change properly.
        *
        * The scenario this change covers is the following:
        * 1) The coroutine is being started as plain non kotlinx.coroutines related suspend function,
        *    e.g. `suspend fun main` or, more importantly, Ktor `SuspendFunGun`, that is invoking
        *    `withContext(tlElement)` which creates `UndispatchedCoroutine`.
        * 2) It (original continuation) is then not wrapped into `DispatchedContinuation` via `intercept()`
        *    and goes neither through `DC.run` nor through `resumeUndispatchedWith` that both
        *    do thread context element tracking.
        * 3) So thread locals never got chance to get properly set up via `saveThreadContext`,
        *    but when `withContext` finishes, it attempts to recover thread locals in its `afterResume`.
        *
        * Here we detect precisely this situation and properly setup context to recover later.
        *
        */
       if (uCont.context[ContinuationInterceptor] !is CoroutineDispatcher) {
           /*
            * We cannot just "read" the elements as there is no such API,
            * so we update-restore it immediately and use the intermediate value
            * as the initial state, leveraging the fact that thread context element
            * is idempotent and such situations are increasingly rare.
            */
           val values = updateThreadContext(context, null)
           restoreThreadContext(context, values)
           saveThreadContext(context, values)
       }
   }

   fun saveThreadContext(context: CoroutineContext, oldValue: Any?) {
       threadStateToRecover.set(context to oldValue)
   }

   fun clearThreadContext(): Boolean {
       if (threadStateToRecover.get() == null) return false
       threadStateToRecover.set(null)
       return true
   }

   override fun afterResume(state: Any?) {
       threadStateToRecover.get()?.let { (ctx, value) ->
           restoreThreadContext(ctx, value)
           threadStateToRecover.set(null)
       }
       // resume undispatched -- update context but stay on the same dispatcher
       val result = recoverResult(state, uCont)
       withContinuationContext(uCont, null) {
           uCont.resumeWith(result)
       }
   }
}
```






## ActorScope

```kotlin
@ObsoleteCoroutinesApi
public interface ActorScope<E> : CoroutineScope, ReceiveChannel<E> {
    public val channel: Channel<E>
}
```

### ActorCoroutine
```kotlin
private open class ActorCoroutine<E>(
    parentContext: CoroutineContext,
    channel: Channel<E>,
    active: Boolean
) : ChannelCoroutine<E>(parentContext, channel, initParentJob = false, active = active), ActorScope<E> {

    init {
        initParentJob(parentContext[Job])
    }

    override fun onCancelling(cause: Throwable?) {
        _channel.cancel(cause?.let {
            it as? CancellationException ?: CancellationException("$classSimpleName was cancelled", it)
        })
    }

    override fun handleJobException(exception: Throwable): Boolean {
        handleCoroutineException(context, exception)
        return true
    }
}
```


## ProducerScope
```kotlin
public interface ProducerScope<in E> : CoroutineScope, SendChannel<E> {
    /**
     * A reference to the channel this coroutine [sends][send] elements to.
     * It is provided for convenience, so that the code in the coroutine can refer
     * to the channel as `channel` as opposed to `this`.
     * All the [SendChannel] functions on this interface delegate to
     * the channel instance returned by this property.
     */
    public val channel: SendChannel<E>
}
```

### ProducerCoroutine
```kotlin
private class ProducerCoroutine<E>(
    parentContext: CoroutineContext, channel: Channel<E>
) : ChannelCoroutine<E>(parentContext, channel, true, active = true), ProducerScope<E> {
    override val isActive: Boolean
        get() = super.isActive

    override fun onCompleted(value: Unit) {
        _channel.close()
    }

    override fun onCancelled(cause: Throwable, handled: Boolean) {
        val processed = _channel.close(cause)
        if (!processed && !handled) handleCoroutineException(context, cause)
    }
}
```







### BroadcastCoroutine

#### LazyBroadcastCoroutine


# 相關方法

## CoroutineScope
### CoroutineScope.cancel
```kotlin
public fun CoroutineScope.cancel(cause: CancellationException? = null) {
    val job = coroutineContext[Job] ?: error("Scope cannot be cancelled because it does not have a job: $this")
    job.cancel(cause)
}
```


### CoroutineScope.newCoroutineContext
```kotlin
@ExperimentalCoroutinesApi
public actual fun CoroutineScope.newCoroutineContext(context: CoroutineContext): CoroutineContext {
    val combined = foldCopies(coroutineContext, context, true)
    val debug = if (DEBUG) combined + CoroutineId(COROUTINE_ID.incrementAndGet()) else combined
    return if (combined !== Dispatchers.Default && combined[ContinuationInterceptor] == null)
        debug + Dispatchers.Default else debug
}

@InternalCoroutinesApi
public actual fun CoroutineContext.newCoroutineContext(addedContext: CoroutineContext): CoroutineContext {
    /*
     * Fast-path: we only have to copy/merge if 'addedContext' (which typically has one or two elements)
     * contains copyable elements.
     */
    if (!addedContext.hasCopyableElements()) return this + addedContext
    return foldCopies(this, addedContext, false)
}
```

### CoroutineDispatcher.invoke
```kotlin
public suspend inline operator fun <T> CoroutineDispatcher.invoke(
    noinline block: suspend CoroutineScope.() -> T
): T = withContext(this, block)
```

### CoroutineScope.produce
```kotlin
@ExperimentalCoroutinesApi
public fun <E> CoroutineScope.produce(
    context: CoroutineContext = EmptyCoroutineContext,
    capacity: Int = 0,
    @BuilderInference block: suspend ProducerScope<E>.() -> Unit
): ReceiveChannel<E> =
    produce(context, capacity, BufferOverflow.SUSPEND, CoroutineStart.DEFAULT, onCompletion = null, block = block)

@InternalCoroutinesApi
public fun <E> CoroutineScope.produce(
    context: CoroutineContext = EmptyCoroutineContext,
    capacity: Int = 0,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    onCompletion: CompletionHandler? = null,
    @BuilderInference block: suspend ProducerScope<E>.() -> Unit
): ReceiveChannel<E> =
    produce(context, capacity, BufferOverflow.SUSPEND, start, onCompletion, block)

// Internal version of produce that is maximally flexible, but is not exposed through public API (too many params)
internal fun <E> CoroutineScope.produce(
    context: CoroutineContext = EmptyCoroutineContext,
    capacity: Int = 0,
    onBufferOverflow: BufferOverflow = BufferOverflow.SUSPEND,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    onCompletion: CompletionHandler? = null,
    @BuilderInference block: suspend ProducerScope<E>.() -> Unit
): ReceiveChannel<E> {
    val channel = Channel<E>(capacity, onBufferOverflow)
    val newContext = newCoroutineContext(context)
    val coroutine = ProducerCoroutine(newContext, channel)
    if (onCompletion != null) coroutine.invokeOnCompletion(handler = onCompletion)
    coroutine.start(start, coroutine, block)
    return coroutine
}
```

### MainScope
```kotlin
/*
 * Creates the main [CoroutineScope] for UI components.
 *
 * Example of use:
 */
 class MyAndroidActivity {
     private val scope = MainScope()

     override fun onDestroy() {
         super.onDestroy()
         scope.cancel()
     }
 }

/ *
 * The resulting scope has [SupervisorJob] and [Dispatchers.Main] context elements.
 * If you want to append additional elements to the main scope, use [CoroutineScope.plus] operator:
 */
 val scope = MainScope() + CoroutineName("MyActivity")

@Suppress("FunctionName")
public fun MainScope(): CoroutineScope = ContextScope(SupervisorJob() + Dispatchers.Main)
```


## ProducerScope
### ProducerScope.awaitClose
```kotlin
public suspend fun ProducerScope<*>.awaitClose(block: () -> Unit = {}) {
    check(kotlin.coroutines.coroutineContext[Job] === this) { "awaitClose() can only be invoked from the producer context" }
    try {
        suspendCancellableCoroutine<Unit> { cont ->
            invokeOnClose {
                cont.resume(Unit)
            }
        }
    } finally {
        block()
    }
}
```




## 其他



### withContext
```kotlin
public suspend fun <T> withContext(
    context: CoroutineContext,
    block: suspend CoroutineScope.() -> T
): T {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return suspendCoroutineUninterceptedOrReturn sc@ { uCont ->
        // compute new context
        val oldContext = uCont.context
        // Copy CopyableThreadContextElement if necessary
        val newContext = oldContext.newCoroutineContext(context)
        // always check for cancellation of new context
        newContext.ensureActive()
        // FAST PATH #1 -- new context is the same as the old one
        if (newContext === oldContext) {
            val coroutine = ScopeCoroutine(newContext, uCont)
            return@sc coroutine.startUndispatchedOrReturn(coroutine, block)
        }
        // FAST PATH #2 -- the new dispatcher is the same as the old one (something else changed)
        // `equals` is used by design (see equals implementation is wrapper context like ExecutorCoroutineDispatcher)
        if (newContext[ContinuationInterceptor] == oldContext[ContinuationInterceptor]) {
            val coroutine = UndispatchedCoroutine(newContext, uCont)
            // There are changes in the context, so this thread needs to be updated
            withCoroutineContext(newContext, null) {
                return@sc coroutine.startUndispatchedOrReturn(coroutine, block)
            }
        }
        // SLOW PATH -- use new dispatcher
        val coroutine = DispatchedCoroutine(newContext, uCont)
        block.startCoroutineCancellable(coroutine, coroutine)
        coroutine.getResult()
    }
}
```

### runBlocking


<br><br><br><br><br><br><br><br><br>
