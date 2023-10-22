---
layout: post
title:  Kotlinx.Coroutine - 來談談 Flow
date:   2022-08-08 16:15:07 +0800
categories: [coroutine, android, intermediate]
---

# Flow
```kotlin
public interface Flow<out T> {
    public suspend fun collect(collector: FlowCollector<T>)
}
```

## EmptyFlow
```kotlin
private object EmptyFlow : Flow<Nothing> {
    override suspend fun collect(collector: FlowCollector<Nothing>) = Unit
}


public fun <T> emptyFlow(): Flow<T> = EmptyFlow
```

## CancellableFlow
```kotlin
internal interface CancellableFlow<out T> : Flow<T>
```

## FusibleFlow

```kotlin
@InternalCoroutinesApi
public interface FusibleFlow<T> : Flow<T> {
    /**
     * This function is called by [flowOn] (with context) and [buffer] (with capacity) operators
     * that are applied to this flow. Should not be used with [capacity] of [Channel.CONFLATED]
     * (it shall be desugared to `capacity = 0, onBufferOverflow = DROP_OLDEST`).
     */
    public fun fuse(
        context: CoroutineContext = EmptyCoroutineContext,
        capacity: Int = Channel.OPTIONAL_CHANNEL,
        onBufferOverflow: BufferOverflow = BufferOverflow.SUSPEND
    ): Flow<T>
}
```

### ChannelFlow
```kotlin
@InternalCoroutinesApi
public abstract class ChannelFlow<T>(
    // upstream context
    @JvmField public val context: CoroutineContext,
    // buffer capacity between upstream and downstream context
    @JvmField public val capacity: Int,
    // buffer overflow strategy
    @JvmField public val onBufferOverflow: BufferOverflow
) : FusibleFlow<T>
```

#### ChannelFlowBuilder
```kotlin
// ChannelFlow implementation that is the first in the chain of flow operations and introduces (builds) a flow
private open class ChannelFlowBuilder<T>(
    private val block: suspend ProducerScope<T>.() -> Unit,
    context: CoroutineContext = EmptyCoroutineContext,
    capacity: Int = BUFFERED,
    onBufferOverflow: BufferOverflow = BufferOverflow.SUSPEND
) : ChannelFlow<T>(context, capacity, onBufferOverflow)
```


##### CallbackFlowBuilder

```kotlin
private class CallbackFlowBuilder<T>(
    private val block: suspend ProducerScope<T>.() -> Unit,
    context: CoroutineContext = EmptyCoroutineContext,
    capacity: Int = BUFFERED,
    onBufferOverflow: BufferOverflow = BufferOverflow.SUSPEND
) : ChannelFlowBuilder<T>(block, context, capacity, onBufferOverflow)
```

#### ChannelFlowMerge
```kotlin
internal class ChannelFlowMerge<T>(
    private val flow: Flow<Flow<T>>,
    private val concurrency: Int,
    context: CoroutineContext = EmptyCoroutineContext,
    capacity: Int = Channel.BUFFERED,
    onBufferOverflow: BufferOverflow = BufferOverflow.SUSPEND
) : ChannelFlow<T>(context, capacity, onBufferOverflow)
```

#### ChannelLimitedFlowMerge
```kotlin
internal class ChannelLimitedFlowMerge<T>(
    private val flows: Iterable<Flow<T>>,
    context: CoroutineContext = EmptyCoroutineContext,
    capacity: Int = Channel.BUFFERED,
    onBufferOverflow: BufferOverflow = BufferOverflow.SUSPEND
) : ChannelFlow<T>(context, capacity, onBufferOverflow)
```

#### ChannelFlowOperator

```kotlin
// ChannelFlow implementation that operates on another flow before it
internal abstract class ChannelFlowOperator<S, T>(
    @JvmField protected val flow: Flow<S>,
    context: CoroutineContext,
    capacity: Int,
    onBufferOverflow: BufferOverflow
) : ChannelFlow<T>(context, capacity, onBufferOverflow)
```

##### ChannelFlowOperatorImpl
```kotlin
/**
 * Simple channel flow operator: [flowOn], [buffer], or their fused combination.
 */
internal class ChannelFlowOperatorImpl<T>(
    flow: Flow<T>,
    context: CoroutineContext = EmptyCoroutineContext,
    capacity: Int = Channel.OPTIONAL_CHANNEL,
    onBufferOverflow: BufferOverflow = BufferOverflow.SUSPEND
) : ChannelFlowOperator<T, T>(flow, context, capacity, onBufferOverflow)
```


##### ChannelFlowTransformLatest
```kotlin
internal class ChannelFlowTransformLatest<T, R>(
    private val transform: suspend FlowCollector<R>.(value: T) -> Unit,
    flow: Flow<T>,
    context: CoroutineContext = EmptyCoroutineContext,
    capacity: Int = Channel.BUFFERED,
    onBufferOverflow: BufferOverflow = BufferOverflow.SUSPEND
) : ChannelFlowOperator<T, R>(flow, context, capacity, onBufferOverflow)
```


#### ChannelAsFlow
```kotlin
private class ChannelAsFlow<T>(
    private val channel: ReceiveChannel<T>,
    private val consume: Boolean,
    context: CoroutineContext = EmptyCoroutineContext,
    capacity: Int = Channel.OPTIONAL_CHANNEL,
    onBufferOverflow: BufferOverflow = BufferOverflow.SUSPEND
) : ChannelFlow<T>(context, capacity, onBufferOverflow)
```



## AbstractFlow
```kotlin
@FlowPreview
public abstract class AbstractFlow<T> : Flow<T>, CancellableFlow<T> {

    public final override suspend fun collect(collector: FlowCollector<T>) {
        val safeCollector = SafeCollector(collector, coroutineContext)
        try {
            collectSafely(safeCollector)
        } finally {
            safeCollector.releaseIntercepted()
        }
    }

    public abstract suspend fun collectSafely(collector: FlowCollector<T>)
}
```

### SafeFlow
```kotlin
private class SafeFlow<T>(private val block: suspend FlowCollector<T>.() -> Unit) : AbstractFlow<T>() {
    override suspend fun collectSafely(collector: FlowCollector<T>) {
        collector.block()
    }
}
```





# FlowCollector
```kotlin
public fun interface FlowCollector<in T> {
    public suspend fun emit(value: T)
}
```






# Flow 的用法

## flow
```kotlin
public fun <T> flow(@BuilderInference block: suspend FlowCollector<T>.() -> Unit): Flow<T> = SafeFlow(block)
```

### asFlow (cold)
```kotlin
@FlowPreview
public fun <T> (() -> T).asFlow(): Flow<T> = flow {
    emit(invoke())
}

@FlowPreview
public fun <T> (suspend () -> T).asFlow(): Flow<T> = flow {
    emit(invoke())
}

public fun <T> Iterable<T>.asFlow(): Flow<T> = flow {
    forEach { value ->
        emit(value)
    }
}

public fun <T> Iterator<T>.asFlow(): Flow<T> = flow {
    forEach { value ->
        emit(value)
    }
}

public fun <T> Sequence<T>.asFlow(): Flow<T> = flow {
    forEach { value ->
        emit(value)
    }
}

public fun <T> Array<T>.asFlow(): Flow<T> = flow {
    forEach { value ->
        emit(value)
    }
}

public fun IntArray.asFlow(): Flow<Int> = flow {
    forEach { value ->
        emit(value)
    }
}

public fun LongArray.asFlow(): Flow<Long> = flow {
    forEach { value ->
        emit(value)
    }
}

public fun IntRange.asFlow(): Flow<Int> = flow {
    forEach { value ->
        emit(value)
    }
}

public fun LongRange.asFlow(): Flow<Long> = flow {
    forEach { value ->
        emit(value)
    }
}
```

### flowOf
```kotlin
public fun <T> flowOf(vararg elements: T): Flow<T> = flow {
    for (element in elements) {
        emit(element)
    }
}

public fun <T> flowOf(value: T): Flow<T> = flow {
    emit(value)
}
```

### channelFlow
```kotlin
public fun <T> channelFlow(@BuilderInference block: suspend ProducerScope<T>.() -> Unit): Flow<T> =
    ChannelFlowBuilder(block)
```


# BufferOverflow




<br><br><br><br><br><br><br><br><br>
