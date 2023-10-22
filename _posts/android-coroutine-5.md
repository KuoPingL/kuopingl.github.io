---
layout: post
title:  Kotlinx.Coroutine - Channel
date:   2022-08-08 16:15:07 +0800
categories: [coroutine, android, intermediate]
---


# Channel

**Channel**
```kotlin
public interface Channel<E> : SendChannel<E>, ReceiveChannel<E>
```

**Channel** 其實是繼承了 **SendChannel** 與 **ReceiveChannel** 的介面。 而在 **Channel** 中還有一個 **Channel.Factory**。 其中定義了一些常數，包括創建 **Channel** 時所需要的 `capacity`：

```kotlin
public companion object Factory {
    public const val UNLIMITED: Int = Int.MAX_VALUE
    public const val RENDEZVOUS: Int = 0
    public const val CONFLATED: Int = -1

    /**
     * Requests a buffered channel with the default buffer capacity in the `Channel(...)` factory function.
     * The default capacity for a channel that [suspends][BufferOverflow.SUSPEND] on overflow
     * is 64 and can be overridden by setting [DEFAULT_BUFFER_PROPERTY_NAME] on JVM.
     * For non-suspending channels, a buffer of capacity 1 is used.
     */
    public const val BUFFERED: Int = -2

    // only for internal use, cannot be used with Channel(...)
    internal const val OPTIONAL_CHANNEL = -3

    /**
     * Name of the property that defines the default channel capacity when
     * [BUFFERED] is used as parameter in `Channel(...)` factory function.
     */
    public const val DEFAULT_BUFFER_PROPERTY_NAME: String = "kotlinx.coroutines.channels.defaultBuffer"

    internal val CHANNEL_DEFAULT_CAPACITY = systemProp(DEFAULT_BUFFER_PROPERTY_NAME,
        64, 1, UNLIMITED - 1
    )
}
```


- **AbstractChannel**
- **ChannelCoroutine**
-



## AbstractChannel

- **ArrayChannel**
- **ConflatedChannel**
- **LinkedListChannel**
- **RendezvousChannel**

### ArrayChannel


### ConflatedChannel


### LinkedListChannel


### RendezvousChannel

**RendezvousChannel** 主要特性就是沒有任何 **Buffer**，為此它的 `isBuffer{State}` 都會回傳 `true`：

```kotlin
internal open class RendezvousChannel<E>(onUndeliveredElement: OnUndeliveredElement<E>?) : AbstractChannel<E>(onUndeliveredElement) {
    protected final override val isBufferAlwaysEmpty: Boolean get() = true
    protected final override val isBufferEmpty: Boolean get() = true
    protected final override val isBufferAlwaysFull: Boolean get() = true
    protected final override val isBufferFull: Boolean get() = true
}

internal typealias OnUndeliveredElement<E> = (E) -> Unit
```

### AbstractChannel 與 Channel 的創建

**Channel\<E\>** 可以通過

```kotlin
public fun <E> Channel(
    capacity: Int = RENDEZVOUS,
    onBufferOverflow: BufferOverflow = BufferOverflow.SUSPEND,
    onUndeliveredElement: ((E) -> Unit)? = null
): Channel<E>
```

<br>

```kotlin
when (capacity) {
    RENDEZVOUS -> {
        if (onBufferOverflow == BufferOverflow.SUSPEND)
            RendezvousChannel(onUndeliveredElement) // an efficient implementation of rendezvous channel
        else
            ArrayChannel(1, onBufferOverflow, onUndeliveredElement) // support buffer overflow with buffered channel
    }
    CONFLATED -> {
        require(onBufferOverflow == BufferOverflow.SUSPEND) {
            "CONFLATED capacity cannot be used with non-default onBufferOverflow"
        }
        ConflatedChannel(onUndeliveredElement)
    }
    UNLIMITED -> LinkedListChannel(onUndeliveredElement) // ignores onBufferOverflow: it has buffer, but it never overflows
    BUFFERED -> ArrayChannel( // uses default capacity with SUSPEND
        if (onBufferOverflow == BufferOverflow.SUSPEND) CHANNEL_DEFAULT_CAPACITY else 1,
        onBufferOverflow, onUndeliveredElement
    )
    else -> {
        if (capacity == 1 && onBufferOverflow == BufferOverflow.DROP_OLDEST)
            ConflatedChannel(onUndeliveredElement) // conflated implementation is more efficient but appears to work in the same way
        else
            ArrayChannel(capacity, onBufferOverflow, onUndeliveredElement)
    }
}
```

這個方法會在以下方法中調用：
- `CoroutineScope.actor`
- `CoroutineScope.produce`
- `FlowCollector.combineInternal`
- `CoroutinesRoom.createFlow`



```kotlin
/*
  CoroutineScope.actor
*/

@ObsoleteCoroutinesApi
public fun <E> CoroutineScope.actor(
    context: CoroutineContext = EmptyCoroutineContext,
    capacity: Int = 0, // todo: Maybe Channel.DEFAULT here?
    start: CoroutineStart = CoroutineStart.DEFAULT,
    onCompletion: CompletionHandler? = null,
    block: suspend ActorScope<E>.() -> Unit
): SendChannel<E> {
    val newContext = newCoroutineContext(context)
    val channel = Channel<E>(capacity)
    val coroutine = if (start.isLazy)
        LazyActorCoroutine(newContext, channel, block) else
        ActorCoroutine(newContext, channel, active = true)
    if (onCompletion != null) coroutine.invokeOnCompletion(handler = onCompletion)
    coroutine.start(start, coroutine, block)
    return coroutine
}
```

<br>

```kotlin
/* CoroutineScope.produce */
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

<br>

```kotlin
/* FlowCollector.combineInternal */

@PublishedApi
internal suspend fun <R, T> FlowCollector<R>.combineInternal(
    flows: Array<out Flow<T>>,
    arrayFactory: () -> Array<T?>?, // Array factory is required to workaround array typing on JVM
    transform: suspend FlowCollector<R>.(Array<T>) -> Unit
): Unit = flowScope { // flow scope so any cancellation within the source flow will cancel the whole scope
    val size = flows.size
    if (size == 0) return@flowScope // bail-out for empty input
    val latestValues = arrayOfNulls<Any?>(size)
    latestValues.fill(UNINITIALIZED) // Smaller bytecode & faster than Array(size) { UNINITIALIZED }
    val resultChannel = Channel<Update>(size)
    val nonClosed = LocalAtomicInt(size)
    var remainingAbsentValues = size
    for (i in 0 until size) {
        // Coroutine per flow that keeps track of its value and sends result to downstream
        launch {
            try {
                flows[i].collect { value ->
                    resultChannel.send(Update(i, value))
                    yield() // Emulate fairness, giving each flow chance to emit
                }
            } finally {
                // Close the channel when there is no more flows
                if (nonClosed.decrementAndGet() == 0) {
                    resultChannel.close()
                }
            }
        }
    }

    /*
     * Batch-receive optimization: read updates in batches, but bail-out
     * as soon as we encountered two values from the same source
     */
    val lastReceivedEpoch = ByteArray(size)
    var currentEpoch: Byte = 0
    while (true) {
        ++currentEpoch
        // Start batch
        // The very first receive in epoch should be suspending
        var element = resultChannel.receiveCatching().getOrNull() ?: break // Channel is closed, nothing to do here
        while (true) {
            val index = element.index
            // Update values
            val previous = latestValues[index]
            latestValues[index] = element.value
            if (previous === UNINITIALIZED) --remainingAbsentValues
            // Check epoch
            // Received the second value from the same flow in the same epoch -- bail out
            if (lastReceivedEpoch[index] == currentEpoch) break
            lastReceivedEpoch[index] = currentEpoch
            element = resultChannel.tryReceive().getOrNull() ?: break
        }

        // Process batch result if there is enough data
        if (remainingAbsentValues == 0) {
            /*
             * If arrayFactory returns null, then we can avoid array copy because
             * it's our own safe transformer that immediately deconstructs the array
             */
            val results = arrayFactory()
            if (results == null) {
                transform(latestValues as Array<T>)
            } else {
                (latestValues as Array<T?>).copyInto(results)
                transform(results as Array<T>)
            }
        }
    }
}
```

<br>

```kotlin
/* CoroutinesRoom.createFlow */

@JvmStatic
public fun <R> createFlow(
    db: RoomDatabase,
    inTransaction: Boolean,
    tableNames: Array<String>,
    callable: Callable<R>
): Flow<@JvmSuppressWildcards R> = flow {
    coroutineScope {
        // Observer channel receives signals from the invalidation tracker to emit queries.
        val observerChannel = Channel<Unit>(Channel.CONFLATED)
        val observer = object : InvalidationTracker.Observer(tableNames) {
            override fun onInvalidated(tables: Set<String>) {
                observerChannel.trySend(Unit)
            }
        }
        observerChannel.trySend(Unit) // Initial signal to perform first query.
        val queryContext = coroutineContext[TransactionElement]?.transactionDispatcher
            ?: if (inTransaction) db.transactionDispatcher else db.getQueryDispatcher()
        val resultChannel = Channel<R>()
        launch(queryContext) {
            db.invalidationTracker.addObserver(observer)
            try {
                // Iterate until cancelled, transforming observer signals to query results
                // to be emitted to the flow.
                for (signal in observerChannel) {
                    val result = callable.call()
                    resultChannel.send(result)
                }
            } finally {
                db.invalidationTracker.removeObserver(observer)
            }
        }

        emitAll(resultChannel)
    }
}
```

## ChannelCoroutine


- **ActorCoroutine**, **LazyActorCoroutine**
- **ProducerCoroutine**, **FlowProduceCoroutine**


### ActorCoroutine

**ActorCoroutine** 蠻有趣的，因為雖然 `initParentJob == false`，但它卻會在 `init` 時調用 `initParentJob(Job)`。 這表示它依舊擁有 `parentHandle: ChildHandle`，可以用來進行 GC ：

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

#### LazyActorCoroutine

與 **ActorCoroutine** 相比， **LazyActorCoroutine** 還實作了 **SelectClause2**：

```kotlin
private class LazyActorCoroutine<E>(
    parentContext: CoroutineContext,
    channel: Channel<E>,
    block: suspend ActorScope<E>.() -> Unit
) : ActorCoroutine<E>(parentContext, channel, active = false),
    SelectClause2<E, SendChannel<E>> {

    private var continuation = block.createCoroutineUnintercepted(this, this)

    override fun onStart() {
        continuation.startCoroutineCancellable(this)
    }

    override suspend fun send(element: E) {
        start()
        return super.send(element)
    }

    @Suppress("DEPRECATION", "DEPRECATION_ERROR")
    override fun offer(element: E): Boolean {
        start()
        return super.offer(element)
    }

    override fun trySend(element: E): ChannelResult<Unit> {
        start()
        return super.trySend(element)
    }

    override fun close(cause: Throwable?): Boolean {
        // close the channel _first_
        val closed = super.close(cause)
        // then start the coroutine (it will promptly fail if it was not started yet)
        start()
        return closed
    }

    override val onSend: SelectClause2<E, SendChannel<E>>
        get() = this

    // registerSelectSend
    override fun <R> registerSelectClause2(select: SelectInstance<R>, param: E, block: suspend (SendChannel<E>) -> R) {
        start()
        super.onSend.registerSelectClause2(select, param, block)
    }
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

#### FlowProduceCoroutine
```kotlin
private class FlowProduceCoroutine<T>(
    parentContext: CoroutineContext,
    channel: Channel<T>
) : ProducerCoroutine<T>(parentContext, channel) {
    public override fun childCancelled(cause: Throwable): Boolean {
        if (cause is ChildCancelledException) return true
        return cancelImpl(cause)
    }
}
```



# **SendChannel**
```kotlin
public interface SendChannel<in E> {
    @ExperimentalCoroutinesApi
    public val isClosedForSend: Boolean

    public suspend fun send(element: E)
    public val onSend: SelectClause2<E, SendChannel<E>>
    public fun trySend(element: E): ChannelResult<Unit>
    public fun close(cause: Throwable? = null): Boolean

    @ExperimentalCoroutinesApi
    public fun invokeOnClose(handler: (cause: Throwable?) -> Unit)
}
```

我們來看看這些方法是如何實作的吧。

## 方法實作
### isClosedForSend

`isClosedForSend` 只有在 **AbstractChannel** 與 **ConflatedBroadcastChannel** 覆寫：

```kotlin
/* AbstractChannel */
protected val closedForSend: Closed<*>? get() = (queue.prevNode as? Closed<*>)?.also { helpClose(it) }
public final override val isClosedForSend: Boolean get() = closedForSend != null

/* ConflatedBroadcastChannel */
private val _state = atomic<Any>(INITIAL_STATE) // State | Closed
public override val isClosedForSend: Boolean get() = _state.value is Closed
```

而 **BroadcastCoroutine** 與 **ChannelCoroutine** 則會由傳入的 `_channel` 實作：

```kotlin
/* BroadcastCoroutine */
private open class BroadcastCoroutine<E>(
    parentContext: CoroutineContext,
    protected val _channel: BroadcastChannel<E>,
    active: Boolean
) : AbstractCoroutine<Unit>(parentContext, initParentJob = false, active = active),
    ProducerScope<E>, BroadcastChannel<E> by _channel

/* ChannelCoroutine */
@Suppress("DEPRECATION")
internal open class ChannelCoroutine<E>(
    parentContext: CoroutineContext,
    protected val _channel: Channel<E>,
    initParentJob: Boolean,
    active: Boolean
) : AbstractCoroutine<Unit>(parentContext, initParentJob, active), Channel<E> by _channel

```

### send(E)

```kotlin
/* AbstractCoroutine */
public final override suspend fun send(element: E) {
    // fast path -- try offer non-blocking
    if (offerInternal(element) === OFFER_SUCCESS) return
    // slow-path does suspend or throws exception
    return sendSuspend(element)
}


```


## AbstractSendChannel

```kotlin
internal abstract class AbstractSendChannel<E>(
    @JvmField protected val onUndeliveredElement: OnUndeliveredElement<E>?
) : SendChannel<E> {

    protected abstract val isBufferAlwaysFull: Boolean
    protected abstract val isBufferFull: Boolean

    protected val queue = LockFreeLinkedListHead()
    private val onCloseHandler = atomic<Any?>(null)
}
```


```kotlin
protected open fun offerInternal(element: E): Any {
    while (true) {
        val receive = takeFirstReceiveOrPeekClosed() ?: return OFFER_FAILED
        val token = receive.tryResumeReceive(element, null)
        if (token != null) {
            assert { token === RESUME_TOKEN }
            receive.completeResumeReceive(element)
            return receive.offerResult
        }
    }
}

// 讀取第一個 Closed 且 ReceiveOrClosed 的 Node 否則就回傳 null
protected open fun takeFirstReceiveOrPeekClosed(): ReceiveOrClosed<E>? =
        queue.removeFirstIfIsInstanceOfOrPeekIf<ReceiveOrClosed<E>>({ it is Closed<*> })
```

**ReceiveOrClosed** 有

```kotlin
internal interface ReceiveOrClosed<in E> {
    val offerResult: Any // OFFER_SUCCESS | Closed
    // Returns: null - failure,
    //          RETRY_ATOMIC for retry (only when otherOp != null),
    //          RESUME_TOKEN on success (call completeResumeReceive)
    // Must call otherOp?.finishPrepare() after deciding on result other than RETRY_ATOMIC
    fun tryResumeReceive(value: E, otherOp: PrepareOp?): Symbol?
    fun completeResumeReceive(value: E)
}
```

```kotlin
internal class Closed<in E>(
    @JvmField val closeCause: Throwable?
) : Send(), ReceiveOrClosed<E> {
    val sendException: Throwable get() = closeCause ?: ClosedSendChannelException(DEFAULT_CLOSE_MESSAGE)
    val receiveException: Throwable get() = closeCause ?: ClosedReceiveChannelException(DEFAULT_CLOSE_MESSAGE)

    override val offerResult get() = this
    override val pollResult get() = this
    override fun tryResumeSend(otherOp: PrepareOp?): Symbol = RESUME_TOKEN.also { otherOp?.finishPrepare() }
    override fun completeResumeSend() {}
    override fun tryResumeReceive(value: E, otherOp: PrepareOp?): Symbol = RESUME_TOKEN.also { otherOp?.finishPrepare() }
    override fun completeResumeReceive(value: E) {}
    override fun resumeSendClosed(closed: Closed<*>) = assert { false } // "Should be never invoked"
    override fun toString(): String = "Closed@$hexAddress[$closeCause]"
}
```

```kotlin
internal abstract class Receive<in E> : LockFreeLinkedListNode(), ReceiveOrClosed<E> {
    override val offerResult get() = OFFER_SUCCESS
    abstract fun resumeReceiveClosed(closed: Closed<*>)
    open fun resumeOnCancellationFun(value: E): ((Throwable) -> Unit)? = null
}
```












# **ReceiveChannel**
```kotlin
public interface ReceiveChannel<out E> {
    @ExperimentalCoroutinesApi
    public val isClosedForReceive: Boolean
    @ExperimentalCoroutinesApi
    public val isEmpty: Boolean

    public suspend fun receive(): E
    public val onReceive: SelectClause1<E>

    public suspend fun receiveCatching(): ChannelResult<E>
    public val onReceiveCatching: SelectClause1<ChannelResult<E>>

    public fun tryReceive(): ChannelResult<E>

    public operator fun iterator(): ChannelIterator<E>
    public fun cancel(cause: CancellationException? = null)
}
```









<br><br><br><br><br><br><br><br><br>
