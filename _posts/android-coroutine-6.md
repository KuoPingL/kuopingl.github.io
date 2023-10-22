---
layout: post
title:  Kotlinx.Coroutine - 從 Continuation 來暸解 Coroutine
date:   2022-08-08 16:15:07 +0800
categories: [coroutine, android, intermediate]
---


# Continuation

```kotlin
/**
 * Interface representing a continuation after a suspension point that returns a value of type `T`.
 */
@SinceKotlin("1.3")
public interface Continuation<in T> {
    public val context: CoroutineContext
    public fun resumeWith(result: Result<T>)
}
```

## CancellableContinuation

```kotlin
public interface CancellableContinuation<in T> : Continuation<T> {
    public val isActive: Boolean
    public val isCompleted: Boolean
    public val isCancelled: Boolean

    @InternalCoroutinesApi
    public fun tryResume(value: T, idempotent: Any? = null): Any?

    @InternalCoroutinesApi
    public fun tryResume(value: T, idempotent: Any?, onCancellation: ((cause: Throwable) -> Unit)?): Any?

    @InternalCoroutinesApi
    public fun tryResumeWithException(exception: Throwable): Any?

    @InternalCoroutinesApi
    public fun completeResume(token: Any)

    @InternalCoroutinesApi
    public fun initCancellability()

    public fun cancel(cause: Throwable? = null): Boolean
    public fun invokeOnCancellation(handler: CompletionHandler)

    @ExperimentalCoroutinesApi
    public fun CoroutineDispatcher.resumeUndispatched(value: T)

    @ExperimentalCoroutinesApi
    public fun CoroutineDispatcher.resumeUndispatchedWithException(exception: Throwable)

    @ExperimentalCoroutinesApi // since 1.2.0
    public fun resume(value: T, onCancellation: ((cause: Throwable) -> Unit)?)
}
```

### ContinuationImpl

```kotlin
@PublishedApi
internal open class CancellableContinuationImpl<in T>(
    final override val delegate: Continuation<T>,
    resumeMode: Int
) : DispatchedTask<T>(resumeMode), CancellableContinuation<T>, CoroutineStackFrame
```


**DispatchedTask** 其實是一個 **SchedulerTask**， 而 **SchedulerTask** 本身就完全等於 **Task**。 而 **Task** 又是一個 **Runnable**：
```kotlin
internal abstract class Task(
    @JvmField var submissionTime: Long,
    @JvmField var taskContext: TaskContext
) : Runnable {
    constructor() : this(0, NonBlockingContext)
    inline val mode: Int get() = taskContext.taskMode // TASK_XXX
}

internal actual typealias SchedulerTask = Task

internal abstract class DispatchedTask<in T>(
    @JvmField public var resumeMode: Int
) : SchedulerTask()
```







```kotlin
// Note: Always returns RESUME_TOKEN | null
override fun tryResume(value: T, idempotent: Any?): Any? =
    tryResumeImpl(value, idempotent, onCancellation = null)

override fun tryResume(value: T, idempotent: Any?, onCancellation: ((cause: Throwable) -> Unit)?): Any? = tryResumeImpl(value, idempotent, onCancellation)

override fun tryResumeWithException(exception: Throwable): Any? =
    tryResumeImpl(CompletedExceptionally(exception), idempotent = null, onCancellation = null)


// 這方法會在 dispatchResume 調用
private fun tryResume(): Boolean {
    _decision.loop { decision ->
        when (decision) {
            UNDECIDED -> if (this._decision.compareAndSet(UNDECIDED, RESUMED)) return true
            SUSPENDED -> return false
            else -> error("Already resumed")
        }
    }
}
```

```kotlin
/**
 * Similar to [tryResume], but does not actually completes resume (needs [completeResume] call).
 * Returns [RESUME_TOKEN] when resumed, `null` when it was already resumed or cancelled.
 */
private fun tryResumeImpl(
    proposedUpdate: Any?,
    idempotent: Any?,
    onCancellation: ((cause: Throwable) -> Unit)?
): Symbol? {
    _state.loop { state ->
        when (state) {
            is NotCompleted -> {
                val update = resumedState(state, proposedUpdate, resumeMode, onCancellation, idempotent)
                if (!_state.compareAndSet(state, update)) return@loop // retry on cas failure
                detachChildIfNonResuable()
                return RESUME_TOKEN
            }
            is CompletedContinuation -> {
                return if (idempotent != null && state.idempotentResume === idempotent) {
                    assert { state.result == proposedUpdate } // "Non-idempotent resume"
                    RESUME_TOKEN // resumed with the same token -- ok
                } else {
                    null // resumed with a different token or non-idempotent -- too late
                }
            }
            else -> return null // cannot resume -- not active anymore
        }
    }
}
```



#### AwaitContinuation

```kotlin
private class AwaitContinuation<T>(
    delegate: Continuation<T>,
    private val job: JobSupport
) : CancellableContinuationImpl<T>(delegate, MODE_CANCELLABLE) {
    override fun getContinuationCancellationCause(parent: Job): Throwable {
        val state = job.state
        /*
         * When the job we are waiting for had already completely completed exceptionally or
         * is failing, we shall use its root/completion cause for await's result.
         */
        if (state is Finishing) state.rootCause?.let { return it }
        if (state is CompletedExceptionally) return state.cause
        return parent.getCancellationException()
    }

    protected override fun nameString(): String =
        "AwaitContinuation"
}
```


### state
```kotlin
State                             isActive    isCompleted   isCancelled
Active (initial state)            true        false         false
Resumed (final completed state)   false       true          false
Canceled (final completed state)  false       true          true

Invocation of cancel transitions this continuation from active to cancelled state, while invocation of Continuation.resume or Continuation.resumeWithException transitions it from active to resumed state.
A cancelled continuation implies that it is completed.
Invocation of Continuation.resume or Continuation.resumeWithException in resumed state produces an IllegalStateException, but is ignored in cancelled state.
    +-----------+   resume    +---------+
    |  Active   | ----------> | Resumed |
    +-----------+             +---------+
          |
          | cancel
          V
    +-----------+
    | Cancelled |
    +-----------+

```



# Task

## TaskImpl
**TaskImpl**
```kotlin
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

**CoroutineScheduler**
```kotlin
fun createTask(block: Runnable, taskContext: TaskContext): Task {
    val nanoTime = schedulerTimeSource.nanoTime()
    if (block is Task) {
        block.submissionTime = nanoTime
        block.taskContext = taskContext
        return block
    }
    return TaskImpl(block, nanoTime, taskContext)
}
```

```kotlin
fun dispatch(block: Runnable, taskContext: TaskContext = NonBlockingContext, tailDispatch: Boolean = false) {
    trackTask() // this is needed for virtual time support
    val task = createTask(block, taskContext)
    // try to submit the task to the local queue and act depending on the result
    val currentWorker = currentWorker()
    val notAdded = currentWorker.submitToLocalQueue(task, tailDispatch)
    if (notAdded != null) {
        if (!addToGlobalQueue(notAdded)) {
            // Global queue is closed in the last step of close/shutdown -- no more tasks should be accepted
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

```kotlin
override fun execute(command: Runnable) = dispatch(command)
```


## SchedulerTask

### DispatchedTask
```kotlin
internal abstract class DispatchedTask<in T>(
    @JvmField public var resumeMode: Int
) : SchedulerTask()
```


```kotlin
public final override fun run() {
    assert { resumeMode != MODE_UNINITIALIZED } // should have been set before dispatching
    val taskContext = this.taskContext
    var fatalException: Throwable? = null
    try {
        val delegate = delegate as DispatchedContinuation<T>
        val continuation = delegate.continuation
        withContinuationContext(continuation, delegate.countOrElement) {
            val context = continuation.context
            val state = takeState() // NOTE: Must take state in any case, even if cancelled
            val exception = getExceptionalResult(state)
            /*
             * Check whether continuation was originally resumed with an exception.
             * If so, it dominates cancellation, otherwise the original exception
             * will be silently lost.
             */
            val job = if (exception == null && resumeMode.isCancellableMode) context[Job] else null
            if (job != null && !job.isActive) {
                val cause = job.getCancellationException()
                cancelCompletedResult(state, cause)
                continuation.resumeWithStackTrace(cause)
            } else {
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


#### AwaitContinuation

#### CancellableContinuationImpl


#### DispatchedContinuation

```kotlin
internal class DispatchedContinuation<in T>(
    @JvmField val dispatcher: CoroutineDispatcher,
    @JvmField val continuation: Continuation<T>
) : DispatchedTask<T>(MODE_UNINITIALIZED), CoroutineStackFrame, Continuation<T> by continuation
```







<br><br><br><br><br><br><br><br>
