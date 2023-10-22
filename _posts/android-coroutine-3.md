---
layout: post
title:  Kotlinx.Coroutine - Job
date:   2022-08-08 16:15:07 +0800
categories: [coroutine, android, intermediate]
---
# 序文
對 **Coroutine** 有興趣的小夥伴們可以去看看 **Coroutine** 的首篇。

我們這篇主要是要講解 **Job** 的整個家族


# Job
><br>
>
>A **Job** instance in the coroutineContext **represents the coroutine itself**.
><br>

從這句話，我們就知道 **Job** 在 **Coroutine** 中的重要性。但其實 **Job** 是一個介面：

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

**Job** 的主要功能為：
- `start`
- `cancel`
- `join`
- `invokeOnCompletion`

通過這些方法 **Job** 的 **state** 也會跟著改變：

|State|isActive|isCompleted|isCancelled|
|:--|:--:|:--:|:--:|
|**New** (optional initial state)|false|false|false|
|**Active** (default initial state)|true|false|false|
|**Completing** (transient state)|true|false|false|
|**Cancelling** (transient state)|false|false|true|
|**Cancelled** (final state)|false|true|true|
|**Completed** (final state)|false|true|false|


而這些 **state** 可能會出現的順序或流程為以下：
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

><br>
>
> 這便是 **Job** 的基本生命週期。 但他的 **state** 又是從何而來呢？
><br>


這就要看 **JobSupport** 了， 因為它一個 **Job** 的完整實作者。

## CompletableJob

**CompletableJob** 是一個 **Job** 的介面，但多了可以進行「完成」的功能：

```kotlin
public interface CompletableJob : Job {
  public fun complete(): Boolean
  public fun completeExceptionally(exception: Throwable): Boolean
}
```

# JobSupport
**JobSupport** 並非簡單的 **Job**，它還可以是 **ChildJob** 或 **ParentJob**。另外，它還是一個可以註冊 **SelectInstance** 的 **SelectClause**。

```kotlin
@Deprecated(level = DeprecationLevel.ERROR, message = "This is internal API and may be removed in the future releases")
public open class JobSupport constructor(active: Boolean) : Job, ChildJob, ParentJob, SelectClause0 {
    final override val key: CoroutineContext.Key<*> get() = Job
}
```

**ChildJob** 與 **ParentJob** 分別是兩個介面。 他們各自定義了 ：
- **ChildJob** 的 **可被取消特性** 以及
- **ParentJob** 在 **ChildJob** 被取消後必須要給理由的職責

```kotlin
@InternalCoroutinesApi
@Deprecated(level = DeprecationLevel.ERROR, message = "This is internal API and may be removed in the future releases")
public interface ChildJob : Job {
    // Parent is cancelling its child by invoking this method. Child finds the cancellation cause using
    @InternalCoroutinesApi
    public fun parentCancelled(parentJob: ParentJob)
}

@InternalCoroutinesApi
@Deprecated(level = DeprecationLevel.ERROR, message = "This is internal API and may be removed in the future releases")
public interface ParentJob : Job {
    // Child job is using this method to learn its cancellation cause when the parent cancels it with ChildJob.parentCancelled.
    // This method is invoked only if the child was not already being cancelle
    @InternalCoroutinesApi
    public fun getChildJobCancellationCause(): CancellationException
}
```

當中比較怪的就是 **SelectClause0** 了。 這也是一個介面，它定義了 **JobSupport** 該如何針對註冊 **SelectInstance** 時的反應：

```kotlin
/**
 * Clause for [select] expression without additional parameters that does not select any value.
 */
public interface SelectClause0 {
    /**
     * Registers this clause with the specified [select] instance and [block] of code.
     * @suppress **This is unstable API and it is subject to change.**
     */
    @InternalCoroutinesApi
    public fun <R> registerSelectClause0(select: SelectInstance<R>, block: suspend () -> R)
}
```

所謂的 **SelectInstance** 又是另一個介面：

```kotlin
/**
 * Internal representation of select instance. This instance is called _selected_ when
 * the clause to execute is already picked.
 *
 * @suppress **This is unstable API and it is subject to change.**
 */
@InternalCoroutinesApi // todo: sealed interface https://youtrack.jetbrains.com/issue/KT-22286
public interface SelectInstance<in R> {

    public val isSelected: Boolean
    public fun trySelect(): Boolean

    /**
     * Tries to select this instance. Returns:
     * * [RESUME_TOKEN] on success,
     * * [RETRY_ATOMIC] on deadlock (needs retry, it is only possible when [otherOp] is not `null`)
     * * `null` on failure to select (already selected).
     * [otherOp] is not null when trying to rendezvous with this select from inside of another select.
     * In this case, [PrepareOp.finishPrepare] must be called before deciding on any value other than [RETRY_ATOMIC].
     *
     * Note, that this method's actual return type is `Symbol?` but we cannot declare it as such, because this
     * member is public, but [Symbol] is internal. When [SelectInstance] becomes a `sealed interface`
     * (see KT-222860) we can declare this method as internal.
     */
    public fun trySelectOther(otherOp: PrepareOp?): Any?
    public fun performAtomicTrySelect(desc: AtomicDesc): Any?
    public val completion: Continuation<R>
    public fun resumeSelectWithException(exception: Throwable)

    public fun disposeOnSelect(handle: DisposableHandle)
}
```

**SelectInstance** 主要功能就是在被 `select` 時執行某些行為。 這我們之後會談。


所以，我們可以知道 **JobSupport** 除了是一個 **Job** 之外，還可以是 **ChildJob** 也可以是 **ParentJob**。並且負責應對註冊 **SelectInstance** 時的行為。

其中最重要的方法是 `initParentJob(Job?)`。 這方法是主要功能是設定 `parentHandle: ChildHandle` ：

```kotlin
/**
 * Initializes parent job.
 * It shall be invoked at most once after construction after all other initialization.
 */
protected fun initParentJob(parent: Job?) {
    assert { parentHandle == null }
    if (parent == null) {
        parentHandle = NonDisposableHandle
        return
    }
    parent.start() // make sure the parent is started
    @Suppress("DEPRECATION")
    val handle = parent.attachChild(this)
    parentHandle = handle
    // now check our state _after_ registering (see tryFinalizeSimpleState order of actions)
    if (isCompleted) {
        handle.dispose()
        parentHandle = NonDisposableHandle // release it just in case, to aid GC
    }
}
```

而且 **JobSupport** 也承擔更新 `_state` 的功能。 至於有哪些 **state**，我們就來看看吧。

## state

**JobSupport** 中其實會定義了 `_state`。它是 `atomic<Any?>` 表示他的行為皆為 atomic，不會在 OS 進行優化記憶體時被拆散。

```kotlin
// Note: use shared objects while we have no listeners
private val _state = atomic<Any?>(if (active) EMPTY_ACTIVE else EMPTY_NEW)
```

其中我們可以看到他的預設值為 **EMPTY_ACTIVE** 或 **EMPTY_NEW**，這便是 **JobSupport** 剛被創建時以及的 **state**。

當然，他的 **state** 也不只如此。 我們可以從官方提供的 **JobSupport** `state` ：

```kotlin
/*
       === Internal states ===

       name       state class              public state  description
       ------     ------------             ------------  -----------
       EMPTY_N    EmptyNew               : New           no listeners
       EMPTY_A    EmptyActive            : Active        no listeners
       SINGLE     JobNode                : Active        a single listener
       SINGLE+    JobNode                : Active        a single listener + NodeList added as its next
       LIST_N     InactiveNodeList       : New           a list of listeners (promoted once, does not got back to EmptyNew)
       LIST_A     NodeList               : Active        a list of listeners (promoted once, does not got back to JobNode/EmptyActive)
       COMPLETING Finishing              : Completing    has a list of listeners (promoted once from LIST_*)
       CANCELLING Finishing              : Cancelling    -- " --
       FINAL_C    Cancelled              : Cancelled     Cancelled (final state)
       FINAL_R    <any>                  : Completed     produced some result


       + Job object is created
       ## NEW: state == EMPTY_NEW | is InactiveNodeList
       ## ACTIVE: state == EMPTY_ACTIVE | is JobNode | is NodeList
       ## CANCELLING: state is Finishing, state.rootCause != null
       ## COMPLETING: state is Finishing, state.isCompleting == true
       ## SEALED: state is Finishing, state.isSealed == true
       ## COMPLETE: state !is Incomplete (CompletedExceptionally | Cancelled)
```

從圖表中，我們可以看到原來不同的 **state class** 會有不同的 **public state**。

所謂的 **state class** 其實就是不同的類別，包括：
- **EMPTY_NEW**
- **EMPTY_ACTIVE**
- **JobNode**
- **InactiveNodeList**
- **NodeList**
- **Finishing**
- **Cancelled** 這其實是 **CompletedExceptionally**
- `<Any>`


接下來我們來看看這些 **state class** 吧。

### Incomplete
**Incomplete** 是最基本的介面。 裡面包含著 `isActive` 與 **NodeList**：

```kotlin
internal interface Incomplete {
    val isActive: Boolean
    val list: NodeList? // is null only for Empty and JobNode incomplete state objects
}
```

#### Empty
**Empty** 可以帶入 `isActive` 並將 **NodeList** 設為 `null`：

```kotlin
private class Empty(override val isActive: Boolean) : Incomplete {
    override val list: NodeList? get() = null
    override fun toString(): String = "Empty{${if (isActive) "Active" else "New" }}"
}
```

**Empty** 也是 **EMPTY_NEW** 與 **EMPTY_ACTIVE** 的類別：

```kotlin
@SharedImmutable
private val EMPTY_NEW = Empty(false)
@SharedImmutable
private val EMPTY_ACTIVE = Empty(true)
```

#### NodeList
**NodeList** 原來也是 **Incomplete** 的實作者。 除此之外，它還是一個 **LockFreeLinkedListHead**。這表示它必須是 **LinkedList** 的 *head*，而且對它的操作都會是 atomic：

```kotlin
internal class NodeList : LockFreeLinkedListHead(), Incomplete {
    override val isActive: Boolean get() = true
    override val list: NodeList get() = this

    fun getString(state: String) = buildString {
        append("List{")
        append(state)
        append("}[")
        var first = true
        this@NodeList.forEach<JobNode> { node ->
            if (first) first = false else append(", ")
            append(node)
        }
        append("]")
    }

    override fun toString(): String =
        if (DEBUG) getString("Active") else super.toString()
}
```

比較有趣的是它的 `list` 回傳的是 `this`。 這是因為我們可以通過 **LockFreeLinkedListHead** 的 `next` 取得其他的值。另外，它的 `isActive` 預設為 `true`。

#### InactiveNodeList

**InactiveNodeList**

與 **NodeList** 不同， **InactiveNodeList** 的 `isActive` 是 `false`。 而且它會帶入一個 **NodeList**：

```kotlin
internal class InactiveNodeList(
    override val list: NodeList
) : Incomplete {
    override val isActive: Boolean get() = false
    override fun toString(): String = if (DEBUG) list.getString("New") else super.toString()
}
```

他會在 **JobSupport** 的 `promoteEmptyToNodeList` 中被創建：

```kotlin
private fun promoteEmptyToNodeList(state: Empty) {
    // try to promote it to LIST state with the corresponding state
    val list = NodeList()
    val update = if (state.isActive) list else InactiveNodeList(list)
    _state.compareAndSet(state, update)
}
```

而 `promoteEmptyToNodeList` 則會在 `invokeOnCompletion` 中監控 `_state` 狀態期間，當 _state 為 inactive 且 Empty 時才會調用。 由於 `_state` 在 **JobSupport** 中的預設值便是 **Empty**，所以理論上會在初次調用 `invokeOnCompletion` 時創建出來。


#### Finishing
**Finishing** 是在 **JobSupport** 生命週期最後時，包括 `tryMakeCancelling` 與 `tryMakeCompleting`， 創建。

**Finishing** 除了是 **Incomplete** 還是一個 **SynchronizedObject**。 這類別中還需要帶入 **NodeList**、 `isCompleted` 及 `rootCause`：

```kotlin
// Completing & Cancelling states,
// All updates are guarded by synchronized(this), reads are volatile
@Suppress("UNCHECKED_CAST")
private class Finishing(
    override val list: NodeList,
    isCompleting: Boolean,
    rootCause: Throwable?
) : SynchronizedObject(), Incomplete
```

其中 **SynchronizedObject** 其實是 **Any**：
```kotlin
@InternalCoroutinesApi
public actual typealias SynchronizedObject = Any
```

對 **Finishing** 而言，我們在對他操作時，我們可以通過 `synchronized` 來進行修改，而讀取則是 **volatile**：

```kotlin
@InternalCoroutinesApi
public actual inline fun <T> synchronized(lock: SynchronizedObject, block: () -> T): T =
    kotlin.synchronized(lock, block)

 /**
 * Executes the given function [block] while holding the monitor of the given object [lock].
 */
@kotlin.internal.InlineOnly
public inline fun <R> synchronized(lock: Any, block: () -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }

    // Force the lock object into a local and use that local for monitor enter/exit.
    // This ensures that the JVM can prove that locking is balanced which is a
    // prerequisite for using fast locking implementations. See KT-48367 for details.
    val lockLocal = lock

    @Suppress("NON_PUBLIC_CALL_FROM_PUBLIC_INLINE", "INVISIBLE_MEMBER", "INVISIBLE_REFERENCE")
    monitorEnter(lockLocal)
    try {
        return block()
    }
    finally {
        @Suppress("NON_PUBLIC_CALL_FROM_PUBLIC_INLINE", "INVISIBLE_MEMBER", "INVISIBLE_REFERENCE")
        monitorExit(lockLocal)
    }
}
```

通過 `synchronized(obj)` 編輯器會調用 `monitorEnter` 與 `monitorExit` [stackoverflow](https://stackoverflow.com/a/9117177/18597115)：
```java
public class Sync {

    public void f() {
        synchronized (this) {
        }
    }

    public synchronized void g() {
    }

}

public void f();
  Code:
   0:   aload_0
   1:   dup
   2:   monitorenter
   3:   monitorexit
   4:   return

public synchronized void g();
  Code:
   0:   return
```

**Finishing** 中會有以下 `atomic` 參數：
```kotlin
private val _isCompleting = atomic(isCompleting)
private val _rootCause = atomic(rootCause)  // 若 != null 則為 isCancelled = true, 反之 isActive = true
private val _exceptionsHolder = atomic<Any?>(null) // 決定是否為 SEALED
```

**SEALED** 是一個 **Symbol**：
```kotlin
@SharedImmutable
private val SEALED = Symbol("SEALED")

/**
 * A symbol class that is used to define unique constants that are self-explanatory in debugger.
 *
 * @suppress **This is unstable API and it is subject to change.**
 */
internal class Symbol(@JvmField val symbol: String) {
    override fun toString(): String = "<$symbol>"

    @Suppress("UNCHECKED_CAST", "NOTHING_TO_INLINE")
    inline fun <T> unbox(value: Any?): T = if (value === this) null as T else value as T
}
```

**Finishing** 主要有兩種方法：
- `sealLocked`
  這方法主要是建立一個 **ArrayList<Throwable>** 並把 `rootCause` 放在 array 首位。
  <br>
  ```kotlin
  // Seals current state and returns list of exceptions
  // guarded by `synchronized(this)`
  fun sealLocked(proposedException: Throwable?): List<Throwable> {
      val list = when(val eh = exceptionsHolder) { // volatile read
          null -> allocateList()
          is Throwable -> allocateList().also { it.add(eh) }
          is ArrayList<*> -> eh as ArrayList<Throwable>
          else -> error("State is $eh") // already sealed -- cannot happen
      }
      val rootCause = this.rootCause // volatile read
      rootCause?.let { list.add(0, it) } // note -- rootCause goes to the beginning
      if (proposedException != null && proposedException != rootCause) list.add(proposedException)
      exceptionsHolder = SEALED
      return list
  }

  private fun allocateList() = ArrayList<Throwable>(4)
  ```

- `addExceptionLocked`
  這方法主要是更新 exceptionsHolder。 其中可能會將 `rootCause` 與 `exception` 放入其中。
  ```kotlin
  // guarded by `synchronized(this)`
  fun addExceptionLocked(exception: Throwable) {
      val rootCause = this.rootCause // volatile read
      if (rootCause == null) {
          this.rootCause = exception
          return
      }
      if (exception === rootCause) return // nothing to do
      when (val eh = exceptionsHolder) { // volatile read
          null -> exceptionsHolder = exception
          is Throwable -> {
              if (exception === eh) return // nothing to do
              exceptionsHolder = allocateList().apply {
                  add(eh)
                  add(exception)

              }
          }
          is ArrayList<*> -> (eh as ArrayList<Throwable>).add(exception)
          else -> error("State is $eh") // already sealed -- cannot happen
      }
  }
  ```

**Finishing** 會在 **JobSupport** 中的兩個方法建立：

- `tryMakeCancelling`
  這方法會創建 `Finishing(list, false, rootCause)`
  <br>
  ```kotlin
  // try make new Cancelling state on the condition that we're still in the expected state
  private fun tryMakeCancelling(state: Incomplete, rootCause: Throwable): Boolean {
      assert { state !is Finishing } // only for non-finishing states
      assert { state.isActive } // only for active states
      // get state's list or else promote to list to correctly operate on child lists
      val list = getOrPromoteCancellingList(state) ?: return false
      // Create cancelling state (with rootCause!)
      val cancelling = Finishing(list, false, rootCause)
      if (!_state.compareAndSet(state, cancelling)) return false
      // Notify listeners
      notifyCancelling(list, rootCause)
      return true
  }
  ```
- `tryMakeCompletingSlowPath`
  這方法會建立 `Finishing(list, false, null)`。所以 completing 時， `rootCause` 會被設為 `null`
  <br>
  ```kotlin
  // Returns one of COMPLETING symbols or final state:
  // COMPLETING_ALREADY -- when already complete or completing
  // COMPLETING_RETRY -- when need to retry due to interference
  // COMPLETING_WAITING_CHILDREN -- when made completing and is waiting for children
  // final state -- when completed, for call to afterCompletion
  private fun tryMakeCompletingSlowPath(state: Incomplete, proposedUpdate: Any?): Any? {
      // get state's list or else promote to list to correctly operate on child lists
      val list = getOrPromoteCancellingList(state) ?: return COMPLETING_RETRY
      // promote to Finishing state if we are not in it yet
      // This promotion has to be atomic w.r.t to state change, so that a coroutine that is not active yet
      // atomically transition to finishing & completing state
      val finishing = state as? Finishing ?: Finishing(list, false, null)
      // must synchronize updates to finishing state
      var notifyRootCause: Throwable? = null
      synchronized(finishing) {
          // check if this state is already completing
          if (finishing.isCompleting) return COMPLETING_ALREADY
          // mark as completing
          finishing.isCompleting = true
          // if we need to promote to finishing then atomically do it here.
          // We do it as early is possible while still holding the lock. This ensures that we cancelImpl asap
          // (if somebody else is faster) and we synchronize all the threads on this finishing lock asap.
          if (finishing !== state) {
              if (!_state.compareAndSet(state, finishing)) return COMPLETING_RETRY
          }
          // ## IMPORTANT INVARIANT: Only one thread (that had set isCompleting) can go past this point
          assert { !finishing.isSealed } // cannot be sealed
          // add new proposed exception to the finishing state
          val wasCancelling = finishing.isCancelling
          (proposedUpdate as? CompletedExceptionally)?.let { finishing.addExceptionLocked(it.cause) }
          // If it just becomes cancelling --> must process cancelling notifications
          notifyRootCause = finishing.rootCause.takeIf { !wasCancelling }
      }
      // process cancelling notification here -- it cancels all the children _before_ we start to to wait them (sic!!!)
      notifyRootCause?.let { notifyCancelling(list, it) }
      // now wait for children
      val child = firstChild(state)
      if (child != null && tryWaitForChild(finishing, child, proposedUpdate))
          return COMPLETING_WAITING_CHILDREN
      // otherwise -- we have not children left (all were already cancelled?)
      return finalizeFinishingState(finishing, proposedUpdate)
  }
  ```

## transition

我們現在知道 **state** 有哪些，以及何時被創建。 接下來我們來探討這些 **state** 是如何轉換的吧。

首先附上官方的轉換圖表：

```kotlin
/*
=== Transitions ===

    New states      Active states       Inactive states

   +---------+       +---------+                          }
   | EMPTY_N | ----> | EMPTY_A | ----+                    } Empty states
   +---------+       +---------+     |                    }
        |  |           |     ^       |    +----------+
        |  |           |     |       +--> |  FINAL_* |
        |  |           V     |       |    +----------+
        |  |         +---------+     |                    }
        |  |         | SINGLE  | ----+                    } JobNode states
        |  |         +---------+     |                    }
        |  |              |          |                    }
        |  |              V          |                    }
        |  |         +---------+     |                    }
        |  +-------> | SINGLE+ | ----+                    }
        |            +---------+     |                    }
        |                 |          |
        V                 V          |
   +---------+       +---------+     |                    }
   | LIST_N  | ----> | LIST_A  | ----+                    } [Inactive]NodeList states
   +---------+       +---------+     |                    }
      |   |             |   |        |
      |   |    +--------+   |        |
      |   |    |            V        |
      |   |    |    +------------+   |   +------------+   }
      |   +-------> | COMPLETING | --+-- | CANCELLING |   } Finishing states
      |        |    +------------+       +------------+   }
      |        |         |                    ^
      |        |         |                    |
      +--------+---------+--------------------+

      === Internal states ===

      name       state class              public state  description
      ------     ------------             ------------  -----------
      EMPTY_N    EmptyNew               : New           no listeners
      EMPTY_A    EmptyActive            : Active        no listeners
      SINGLE     JobNode                : Active        a single listener
      SINGLE+    JobNode                : Active        a single listener + NodeList added as its next
      LIST_N     InactiveNodeList       : New           a list of listeners (promoted once, does not got back to EmptyNew)
      LIST_A     NodeList               : Active        a list of listeners (promoted once, does not got back to JobNode/EmptyActive)
      COMPLETING Finishing              : Completing    has a list of listeners (promoted once from LIST_*)
      CANCELLING Finishing              : Cancelling    -- " --
      FINAL_C    Cancelled              : Cancelled     Cancelled (final state)
      FINAL_R    <any>                  : Completed     produced some result

      + Job object is created
      ## NEW: state == EMPTY_NEW | is InactiveNodeList
      ## ACTIVE: state == EMPTY_ACTIVE | is JobNode | is NodeList
      ## CANCELLING: state is Finishing, state.rootCause != null
      ## COMPLETING: state is Finishing, state.isCompleting == true
      ## SEALED: state is Finishing, state.isSealed == true
      ## COMPLETE: state !is Incomplete (CompletedExceptionally | Cancelled)
*/
```

### **EMPTY_NEW** 或 **EMPTY_N**

|name|state class |public state|description|
|:--|:--|:--|:--|
| **EMPTY_N**  |  **EmptyNew** | New  | no listeners  |

這個狀態只會在 **JobSupport** 初始化且 `active` 為 `false` 才會出現，包括以下 **JobSupport** 的子類別：
- **LazyActorCoroutine**
- **LazyStandaloneCoroutine**
- **LazyDeferredCoroutine**
- **LazyBroadcastCoroutine**


### **EMPTY_ACTIVE** 或 **EMPTY_A**

|name|state class |public state|description|
|:--|:--|:--|:--|
| **EMPTY_A**  |  **EmptyActive** | Active  | no listeners  |

想要從 **EMPTY_N** 到 **EMPTY_A** 我們需要將 **JobSupport** 的 `isActive` 改為 `true`。
但想要將 **JobSupport** 的 `active` 設為 `true 只有在創建 **JobSupport** 時設定。以下便是有此設定的 **JobSupport** 子類別：

- **CompletableDeferredImpl**
- **JobImpl** :> **SupervisorJobImpl** 和 **ReportingSupervisorJob**
- **BlockingCoroutine**
- **TestScopeImpl**
- **TestBodyCoroutine**
- **ProduceCoroutine**
- `CoroutineScope.actor` 創建 **ActorCoroutine**
- `CoroutineScope.launch` 創建 **StandaloneCoroutine**
- `CoroutineScope.async` 創建 **DeferredCoroutine**
- `CoroutineScope.broadcast` 創建 **BroadcastCoroutine**


### **SINGLE**

|name|state class |public state|description|
|:--|:--|:--|:--|
| **SINGLE**  |  **JobNode** | Active  | a single listener ( **JobSupport** )  |


此時的 `_state` 會是個 **JobNode**。 他會通過 `makeNode` 建立傳入：

```kotlin
// val node: JobNode = makeNode(handler, onCancelling)

private fun makeNode(handler: CompletionHandler, onCancelling: Boolean): JobNode {
    val node = if (onCancelling) {
        // JobNode :> JobCancellingNode
        (handler as? JobCancellingNode)
            ?: InvokeOnCancelling(handler)
    } else {
        (handler as? JobNode)
            ?.also { assert { it !is JobCancellingNode } }
            ?: InvokeOnCompletion(handler)
    }
    node.job = this
    return node
}
```

`makeNode` 在確定 `handle` 不為 **JobNode** 或 **JobCancellingNode** 後創建 **InvokeOnCancelling** 或 **InvokeOnCompletion** 並將 `handle` 包裹其中：

```kotlin
private class InvokeOnCancelling(
    private val handler: CompletionHandler
) : JobCancellingNode()  {
    // delegate handler shall be invoked at most once, so here is an additional flag
    private val _invoked = atomic(0) // todo: replace with atomic boolean after migration to recent atomicFu
    override fun invoke(cause: Throwable?) {
        // make sure it only invoke once
        if (_invoked.compareAndSet(0, 1)) handler.invoke(cause)
    }
}

private class InvokeOnCompletion(
    private val handler: CompletionHandler
) : JobNode()  {
    override fun invoke(cause: Throwable?) = handler.invoke(cause)
}
```

<u>值得注意的是， **InvokeOnCancelling** 的 `invoke` 只可以被調用 **一次**。</u>


那什麼情況之下 `handle` 會是一個 **JobNode** 呢？如果我們繼續順藤摸瓜，便會發現 **CompletionHandler** 會在以下地方以 **JobCancellingNode** 或 **JobNode** 類型帶入其中：
- **CancellableContinuationImpl** 中， `handle` 會以 **ChildContinuation** 從 `installParentHandle` 中通過 `invokeOnCompletion` 帶入其中。  ( **JobCancellingNode** :> **ChildContinuation** )
- **JobSupport** 也會通過 `tryWaitForChild` 與 `attachChild` 傳入 **ChildContinuation**
- **SelectBuilderImpl** 會通過 `initCancellability` 創建 **SelectOnCancelling** 再次從 `invokeOnCompletion` 傳入。 ( **JobCancellingNode** :> **SelectOnCancelling** )

當然， **CompletionHandler** 還可以從其他類別傳入，包括：
- **ThreadState**
- **DisposableHandle**

**JobNode** 會晚點再深談。我們現在只需要知道它是如何被創建以及它與 **SINGLE** 的關係。

<u>那我們要如何從 **SINGLE** 回到 **EMPTY_A** 呢？</u>

我們只需要通過 `jobNode.dispose()` 便可將自己從 **JobSupport** 移除。 因為 `jobNode.dispose()` 會直接調用 **JobSupport** 的 `removeNode`：

```kotlin
/**
 * @suppress **This is unstable API and it is subject to change.**
 */
internal fun removeNode(node: JobNode) {
    // remove logic depends on the state of the job
    loopOnState { state ->
        when (state) {
            is JobNode -> { // SINGE/SINGLE+ state -- one completion handler
                if (state !== node) return // a different job node --> we were already removed
                // try remove and revert back to empty state
                if (_state.compareAndSet(state, EMPTY_ACTIVE)) return
            }
            is Incomplete -> { // may have a list of completion handlers
                // remove node from the list if there is a list
                if (state.list != null) node.remove()
                return
            }
            else -> return // it is complete and does not have any completion handlers
        }
    }
}
```

### **SINGLE+**

|name|state class |public state|description|
|:--|:--|:--|:--|
| **SINGLE+**  |  **JobNode** | Active  | a single listener + NodeList added as its next  |


此時的 `_state` 依舊是 **JobNode**，但 **NodeList** 不會是 `null`。

<u>那我們該如何將預設為 `null` 的 `list` 變成非 `null` 呢？</u>

我們只需要通過 **JobNode** 的 `promoteSingleToNodeList` 進行：

```kotlin
private fun promoteSingleToNodeList(state: JobNode) {
    // try to promote it to list (SINGLE+ state)
    state.addOneIfEmpty(NodeList())
    // it must be in SINGLE+ state or state has changed (node could have need removed from state)
    val list = state.nextNode // either our NodeList or somebody else won the race, updated state
    // just attempt converting it to list if state is still the same, then we'll continue lock-free loop
    _state.compareAndSet(state, list)
}
```

如此一來 `state` 就會多了一個空的 **NodeList** 了。

有趣的一件事就是 **NodeList** 其實是一個 **LockFreeLinkedListHead**：

```kotlin
internal class NodeList : LockFreeLinkedListHead(), Incomplete
```

**這表示它必定會放在 LinkedList 的前面。**


而傳入的 **JobNode** 其實就是 **LockFreeLinkedListNode**， 因為它繼承了 **CompletionHandlerBase**。 所以它才可以通過 **LockFreeLinkedListNode** 的 `addOneIfEmpty` 將 `_state` 轉換成 **SINGLE+**：

```kotlin
public actual fun addOneIfEmpty(node: Node): Boolean {
    node._prev.lazySet(this)
    node._next.lazySet(this)

    /*
    *                +-----------------------+
    *          this  |         node          v  next (this)
    *          +---+---+     +---+---+     +---+---+
    *  ... <-- | P | N | <== | P | N |     | P | N | --> ....
    *          +---+---+     +---+---+     +---+---+
    *              ^                         |
    *              |                         |
    *              +-------------------------+
    * `this` == JobNode
    * `node` == NodeList()
    * `next` == `this` if state == SINGLE
    */

    while (true) {
        val next = next

        if (next !== this) return false // this is not an empty list!

        if (_next.compareAndSet(this, node)) {

          /*
          *
          *          this            node             next (this)
          *          +---+---+     +---+---+     +---+---+
          *  ... <-- | P | N | --> | P | N | --> | P | N |
          *          +---+---+     +---+---+     +---+---+
          *              ^ ^         |             |
          *              | +---------+             |
          *              +-------------------------+
          * `this` == JobNode
          * `node` == NodeList()
          * `next` == `this` if state == SINGLE
          */

            // added successfully (linearized add) -- fixup the list
            node.finishAdd(this)

            /*
            *
            *          this            node             next (this)
            *          +---+---+     +---+---+     +---+---+
            *  ... <-- | P | N | --> | P | N | --> | P | N | --> ....
            *          +---+---+     +---+---+     +---+---+
            *                ^         |   ^         |
            *                +---------+   +---------+
            *
            * `this` == JobNode
            * `node` == NodeList()
            * `next` == `this` if state == SINGLE
            * this forms a circular loop
            */

            return true
        }
    }
}

/**
 * Given:
 * ```
 *
 *          prev            this             next
 *          +---+---+     +---+---+     +---+---+
 *  ... <-- | P | N | --> | P | N | --> | P | N | --> ....
 *          +---+---+     +---+---+     +---+---+
 *              ^ ^         |             |
 *              | +---------+             |
 *              +-------------------------+
 *
 * ```
 * Produces:
 * ```
 *          prev            this             next
 *          +---+---+     +---+---+     +---+---+
 *  ... <-- | P | N | --> | P | N | --> | P | N | --> ....
 *          +---+---+     +---+---+     +---+---+
 *                ^         |   ^         |
 *                +---------+   +---------+
 * ```
 *
 */
private fun finishAdd(next: Node) {
    next._prev.loop { nextPrev ->
        if (this.next !== next) return // this or next was removed or another node added, remover/adder fixes up links
        if (next._prev.compareAndSet(nextPrev, this)) {
            // This newly added node could have been removed, and the above CAS would have added it physically again.
            // Let us double-check for this situation and correct if needed
            if (isRemoved) next.correctPrev(null)
            return
        }
    }
}
```

所以通過 `addOneIfEmpty` **JobNode** 與 **NodeList** 會變成一個 circular linkedList。

```
*          JobNode       NodeList
*          +---+---+     +---+---+
*          | P | N | --> | P | N |
*          +---+---+     +---+---+
*            ^                 |
*            +-----------------+
```

從這裡可以理解 **JobNode** 一個用來存放 **JobSupport** 的 node。 而 **NodeList** 則是這個 **JobSupport**  負責的資料 ( **LockFreeLinkedListNode** )。


所謂的資料其實是 **NodeList** 或 **LockFreeLinkedListHead** 後面所連接的 **LockFreeLinkedListNode**。 而繼承 **LockFreeLinkedListNode** 的類別則包括 **JobNode** 的實作者們：

|File|Class|Super|
|:--|:--|:--|
|Await.kt   | **AwaitAll.AwaitAllNode**  |**JobNode**|
|AbstractChannel   | **Receive**\n**Send**\n**SendElement**\n**Closed**  |  **LockFreeLinkedListNode** |
|CompletionHandler.kt   |  **CompletionHandlerBase** |  **LockFreeLinkedListNode** |
|Future.kt   | **CancelFutureOnCompletion**   |**JobNode**|
|JobSupport.kt   | **JobSupport.ChildCompletion**\n**ChildHandleNode**\n**ChildContinuation**\n**DisposeOnCompletion**\n****SelectJoinOnCompletion**\n**SelectAwaitOnCompletion**\n**InvokeOnCancelling**\n**InvokeOnCompletion** \n**ResumeOnCompletion**\n**ResumeAwaitOnCompletion**|**JobNode**|
|Select.kt   | **SelectBuilderImpl.DisposeNode**  | **LockFreeLinkedListNode**  |
|**Mutex.kt**   | **MutexImpl.LockWaiter**\n**MutexImpl.LockSelect**\n**MutexImpl.LockCont**  | **LockFreeLinkedListNode**  |


><br>
>
> 值得注意的是， 能成為 **LockFreeLinkedListHead** 除了 **NodeList** 外，還包括了：
> - **SelectBuilderImpl**
> - **MutexImpl.LockedQueue**
><br>

### **LIST_A**

|name|state class |public state|description|
|:--|:--|:--|:--|
| **LIST_A**  |  **NodeList** | Active  | a list of listeners (promoted once, does not got back to JobNode/EmptyActive)  |

想要將 **SINGLE+** 變成 **LIST_A** 我們只需要將更多的 **JobNode** 連接到 **NodeList** 之後即可。由於 **JobNode** 中的 **JobSupport** 可被視為 listener，所以我們就會有很多的 listeners 了。


而新增的方法就需要看 **LockFreeLinkedListNode** 中的以下方法：
- `addOneIfEmpty`
  這方法負責將 **SINGLE** 轉換成 **SINGLE+**，所以這裡部會討論。
- `addNext`
  這方法主要是將 `node` 插入 `this` 與 `next` 之間。 這方法會在 `addLast` 與 `addLastIf` 調用。

  ```kotlin
  // ------ addXXX util ------

  /**
   * Given:
   * ```
   *                +-----------------------+
   *          this  |         node          V  next
   *          +---+---+     +---+---+     +---+---+
   *  ... <-- | P | N |     | P | N |     | P | N | --> ....
   *          +---+---+     +---+---+     +---+---+
   *                ^                       |
   *                +-----------------------+
   * ```
   * Produces:
   * ```
   *          this            node             next
   *          +---+---+     +---+---+     +---+---+
   *  ... <-- | P | N | ==> | P | N | --> | P | N | --> ....
   *          +---+---+     +---+---+     +---+---+
   *                ^         |   ^         |
   *                +---------+   +---------+
   * ```
   *  Where `==>` denotes linearization point.
   *  Returns `false` if `next` was not following `this` node.
   */
  @PublishedApi
  internal fun addNext(node: Node, next: Node): Boolean {
      node._prev.lazySet(this)
      node._next.lazySet(next)

      /*
      *                +-----------------------+
      *          this  |         node          V  next
      *          +---+---+     +---+---+     +---+---+
      *  ... <-- | P | N |     | P | N | --> | P | N | --> ....
      *          +---+---+     +---+---+     +---+---+
      *                ^         |             |
      *                +---------+             |
      *                +-----------------------+
      */

      if (!_next.compareAndSet(next, node)) return false  // [this] <===> [node] circular linkedList

      /*
      *
      *          this             node            next
      *          +---+---+     +---+---+     +---+---+
      *  ... <-- | P | N | --> | P | N | --> | P | N | --> ....
      *          +---+---+     +---+---+     +---+---+
      *                ^         |             |
      *                +---------+             |
      *                +-----------------------+
      */

      // added successfully (linearized add) -- fixup the list
      node.finishAdd(next)
      return true
  }
  ```
- `finishAdd`
  這方法會在 `addOneIfEmpty` 、 `addNext` 、 **CondAddOp** 與 **AddLastDesc** 中調用。
  它的主要功能是將 `next._prev` 指向正確的位置 `this`：

  ```kotlin
  // ------ other helpers ------
  /**
   * Given:
   * ```
   *
   *          prev            this             next
   *          +---+---+     +---+---+     +---+---+
   *  ... <-- | P | N | --> | P | N | --> | P | N | --> ....
   *          +---+---+     +---+---+     +---+---+
   *              ^ ^         |             |
   *              | +---------+             |
   *              +-------------------------+
   * ```
   * Produces:
   * ```
   *          prev            this             next
   *          +---+---+     +---+---+     +---+---+
   *  ... <-- | P | N | --> | P | N | --> | P | N | --> ....
   *          +---+---+     +---+---+     +---+---+
   *                ^         |   ^         |
   *                +---------+   +---------+
   * ```
   */
  private fun finishAdd(next: Node) {
      next._prev.loop { nextPrev ->
          if (this.next !== next) return // this or next was removed or another node added, remover/adder fixes up links
          if (next._prev.compareAndSet(nextPrev, this)) {
              // This newly added node could have been removed, and the above CAS would have added it physically again.
              // Let us double-check for this situation and correct if needed
              if (isRemoved) next.correctPrev(null)
              return
          }
      }
  }
  ```
- `addLast`
  這方法會將 `node` 插入 `prevNode` 與 `this` 之間：

  ```kotlin
  /**
   * Adds last item to this list.
   * This will add node between prevNode and this
   *
   *          prevNode        node             this
   *          +---+---+     +---+---+     +---+---+
   *  ... <-- | P | N | ==> | P | N | --> | P | N | --> ....
   *          +---+---+     +---+---+     +---+---+
   *                ^         |   ^         |
   *                +---------+   +---------+
   */
  public actual fun addLast(node: Node) {
      while (true) { // lock-free loop on prev.next
          if (prevNode.addNext(node, this)) return
      }
  }
  ```

- `addLastIf`
  這方法與 `addLast` 很類似，但中間會由 **CondAddOp**，一個 **AtomicOp<Node>**， 增加 Condition ( 條件 )。

  由於這方法只會被 **JobSupport** 的 `addLastAtomic` 通過 `invokeOnCompletion` 調用，所以他的 Condition 只會是：

  ```kotlin
  private fun addLastAtomic(expect: Any, list: NodeList, node: JobNode) =
        list.addLastIf(node) { this.state === expect }
  ```
  <br>

  也就是所謂的條件其實就是確保 `state` 不變。

  接下來我們來深入暸解 `addLastIf` 的流程吧。

  **CondAddOp** 會通過 `makeCondAddOp` 創建，並將 `newNode` 設為 `node` 。這個 **AtomicOp** 會陪同 `node` 以及 **LockFreeLinkedListNode** 傳入 `tryCondAddNext`。

  <br>

  ```kotlin
  private typealias Node = LockFreeLinkedListNode

  /**
   * Adds last item to this list atomically if the [condition] is true.
   */
  public actual inline fun addLastIf(node: Node, crossinline condition: () -> Boolean): Boolean {
      val condAdd = makeCondAddOp(node, condition)
      while (true) { // lock-free loop on prev.next
          val prev = prevNode // sentinel node is never removed, so prev is always defined
          when (prev.tryCondAddNext(node, this, condAdd)) {
              SUCCESS -> return true
              FAILURE -> return false
          }
      }
  }

  @PublishedApi
  internal inline fun makeCondAddOp(node: Node, crossinline condition: () -> Boolean): CondAddOp =
      object : CondAddOp(node) {
          override fun prepare(affected: Node): Any? = if (condition()) null else CONDITION_FALSE
      }
  ```

  `tryCondAddNext` 主要目的是將 `condAdd` 加入 **LinkedList** 之中，然後調用 `condAdd.perform` 並會由 **AtomicOp<Node>** 實作：

  <br>
  ```kotlin
  // LockFreeLinkedListNode
  // returns UNDECIDED, SUCCESS or FAILURE
  // prev.tryCondAddNext(node, this, condAdd)
  //    this : NodeList ... LockFreeLinkedListHead
  @PublishedApi
  internal fun tryCondAddNext(node: Node, next: Node, condAdd: CondAddOp): Int {
      node._prev.lazySet(this)
      node._next.lazySet(next)

      /*
      * condAdd.newNode = affected, condAdd.oldNode = null
      *
      *                +-----------------------+
      *          this  |         node          V  next
      *          +---+---+     +---+---+     +---+---+
      *  ... <-- | P | N |     | P | N | --> | P | N | --> ....
      *          +---+---+     +---+---+     +---+---+
      *                ^         |             |
      *                +---------+             |
      *                +-----------------------+
      */

      condAdd.oldNext = next

      if (!_next.compareAndSet(next, condAdd)) return UNDECIDED

      /*
      * condAdd.newNode = affected, condAdd.oldNode = next
      *
      *                              +------ condAdd
      *          this            node|            next
      *          +---+---+     +---+---+     +---+---+
      *  ... <-- | P | N |     | P | N |     | P | N | --> ....
      *          +---+---+     +---+---+     +---+---+
      *                ^         |             |
      *                +---------+             |
      *                +-----------------------+
      */

      // added operation successfully (linearized) -- complete it & fixup the list
      return if (condAdd.perform(this) == null) SUCCESS else FAILURE
  }

  @PublishedApi
  internal abstract class CondAddOp(
      @JvmField val newNode: Node
  ) : AtomicOp<Node>() {
      @JvmField var oldNext: Node? = null

      override fun complete(affected: Node, failure: Any?) {
          val success = failure == null
          val update = if (success) newNode else oldNext
          if (update != null && affected._next.compareAndSet( this, update)) {
              // only the thread the makes this update actually finishes add operation
              if (success) newNode.finishAdd(oldNext!!)
          }
      }
  }
  ```
  <br>
  在調用 `tryCondAddNext` 時，由於 **CondAddOp** 才剛被創建，所以它的 `_consensus` 會是 **UNDECIDED**。 因此 `perform` 會調用 `decide(prepare(affected as T))` 來取得新的 `decision`。

  <br>
  ```kotlin
  // AtomicOp<T>
  private val _consensus = atomic<Any?>(NO_DECISION)

  @Suppress("UNCHECKED_CAST")
  final override fun perform(affected: Any?): Any? {
      // make decision on status
      var decision = this._consensus.value
      if (decision === NO_DECISION) {
          decision = decide(prepare(affected as T))
      }
      // complete operation
      complete(affected as T, decision)
      return decision
  }
  ```
  <br>

  `prepare` 的實作就是 `makeCondAddOp` 中看到的：

  <br>

  ```kotlin
  override fun prepare(affected: Node): Any? = if (condition()) null else CONDITION_FALSE
  ```
  <br>

  而 `condition` 則會由 `addLastIf` 傳入。 調用 `addLastIf` 的地方就是 **JobSupport** 的 `addLastAtomic`：

  ```kotlin
  private fun addLastAtomic(expect: Any, list: NodeList, node: JobNode) =
        list.addLastIf(node) { this.state === expect }
  ```
  <br>

  `addLastAtomic` 則會在 **JobSupport** 的 `invokeOnCompletion` 時被調用。 它的主要功能其實是確保 `state` 在沒有被改變的情況下進行需要的行為。

  在這裡， 若 `state` 沒改變， **CondAddOp** 的 `prepare` 便會回傳 `null` 反之則回傳 `CONDITION_FALSE`。 而這個結果便會回傳至 **Atomic<Node>** 的 `decide` 之中。
  <br>

  ```kotlin
  fun decide(decision: Any?): Any? {
      assert { decision !== NO_DECISION }
      val current = _consensus.value
      if (current !== NO_DECISION) return current
      if (_consensus.compareAndSet(NO_DECISION, decision)) return decision
      return _consensus.value
  }
  ```

  `decide` 主要是將 `_consensus` 的值更新為 `decision`，並進行回傳。 回到 `perform` 之後， 就會調用 `complete` 並將 **CondAddOp** 與 `decision` 帶入：

  <br>

  ```kotlin
  complete(affected as T, decision)

  // CondAddOp
  override fun complete(affected: Node, failure: Any?) {
      val success = failure == null
      val update = if (success) newNode else oldNext

      /*
      * condAdd.newNode = affected, condAdd.oldNode = next
      *
      *                              +------ condAdd
      *          prev        affected|            next
      *          +---+---+     +---+---+     +---+---+
      *  ... <-- | P | N |     | P | N |     | P | N | --> ....
      *          +---+---+     +---+---+     +---+---+
      *                ^         |             |
      *                +---------+             |
      *                +-----------------------+
      */

      if (update != null && affected._next.compareAndSet( this, update)) {

        /*
        * success == true
        * condAdd.newNode = affected, condAdd.oldNode = next
        *
        *
        *          this           affected          next
        *          +---+---+     +---+---+     +---+---+
        *  ... <-- | P | N |     | P | N | --> | P | N | --> ....
        *          +---+---+     +---+---+     +---+---+
        *                ^         |             |
        *                +---------+             |
        *                +-----------------------+
        * success == false
        *
        *          this           affected -+       next
        *          +---+---+     +---+---+  |  +---+---+
        *  ... <-- | P | N |     | P | N | -+  | P | N | --> ....
        *          +---+---+     +---+---+     +---+---+
        *                ^         |             |
        *                +---------+             |
        *                +-----------------------+
        */



          // only the thread the makes this update actually finishes add operation
          if (success) newNode.finishAdd(oldNext!!)
      }
  }
  ```

  <br>

  之後 `perform` 便會回傳 `decision`，可以是 `null` 或 `CONDITION_FALSE`。這些值又會回傳到 `tryCondAddNext`，若結果為 `null` 便會回傳 `SUCCESS` 否則則回傳 `FAILURE`。最後，`addLastIf` 就會對應著 `SUCCESS` 或 `FAILURE` 回傳 `true` 或 `false`。


- `addLastIfPrev`
  ```kotlin
  public actual inline fun addLastIfPrev(node: Node, predicate: (Node) -> Boolean): Boolean {
      while (true) { // lock-free loop on prev.next
          val prev = prevNode // sentinel node is never removed, so prev is always defined
          if (!predicate(prev)) return false
          if (prev.addNext(node, this)) return true
      }
  }
  ```
  <br>

  這方法會由 **AbstractSendChannel** 的 `queue` **LockFreeLinkedListHead** 在以下方法 `sendBuffered`、 `enqueueSend` 與 `close` 中調用。 另外，它還會在 **AbstractChannel** 的 `enqueueReceiveInternal` 中通過 `queue` 調用。

  而他的主要目的就是在 `predicate` 為 `true` 時將 `node` 通過 `addNext` 插入在 `prev` 與 `this` 之間。

  接下來我們看看 `addLastIfPrev` 的流程吧。

  以下是 `predicate` 中的條件設定：

  <br>

  ```kotlin
  // AbstractSendChannel
  protected fun sendBuffered(element: E): ReceiveOrClosed<*>? {
      queue.addLastIfPrev(SendBuffered(element)) { prev ->
          if (prev is ReceiveOrClosed<*>) return@sendBuffered prev
          true
      }
      return null
  }

  protected open fun enqueueSend(send: Send): Any? {
      if (isBufferAlwaysFull) {
          queue.addLastIfPrev(send) { prev ->
              if (prev is ReceiveOrClosed<*>) return@enqueueSend prev
              true
          }
      } else {
          if (!queue.addLastIfPrevAndIf(send, { prev ->
              if (prev is ReceiveOrClosed<*>) return@enqueueSend prev
              true
          }, { isBufferFull }))
              return ENQUEUE_FAILED
      }
      return null
  }

  public override fun close(cause: Throwable?): Boolean {
      val closed = Closed<E>(cause)
      /*
       * Try to commit close by adding a close token to the end of the queue.
       * Successful -> we're now responsible for closing receivers
       * Not successful -> help closing pending receivers to maintain invariant
       * "if (!close()) next send will throw"
       */
      val closeAdded = queue.addLastIfPrev(closed) { it !is Closed<*> }
      val actuallyClosed = if (closeAdded) closed else queue.prevNode as Closed<*>
      helpClose(actuallyClosed)
      if (closeAdded) invokeOnCloseHandler(cause)
      return closeAdded // true if we have closed
  }

  // AbstractChannel
  protected open fun enqueueReceiveInternal(receive: Receive<E>): Boolean = if (isBufferAlwaysEmpty)
      queue.addLastIfPrev(receive) { it !is Send } else
      queue.addLastIfPrevAndIf(receive, { it !is Send }, { isBufferEmpty })
  ```


- `addLastIfPrevAndIf`
  `addLastIfPrevAndIf` 與 `addLastIf` 非常相近。 主要差別就是在 `tryCondAddNext` 之前它還會通過 `predicate` 進行第二次的確認。
  <br>
  ```kotlin
  public actual inline fun addLastIfPrevAndIf(
          node: Node,
          predicate: (Node) -> Boolean, // prev node predicate
          crossinline condition: () -> Boolean // atomically checked condition
  ): Boolean {
      val condAdd = makeCondAddOp(node, condition)
      while (true) { // lock-free loop on prev.next
          val prev = prevNode // sentinel node is never removed, so prev is always defined
          if (!predicate(prev)) return false
          when (prev.tryCondAddNext(node, this, condAdd)) {
              SUCCESS -> return true
              FAILURE -> return false
          }
      }
  }
  ```
  <br>

  調用這方法的地方只有兩處：

  ```kotlin
  // AbstractChannel
  protected open fun enqueueReceiveInternal(receive: Receive<E>): Boolean = if (isBufferAlwaysEmpty)
      queue.addLastIfPrev(receive) { it !is Send } else
      queue.addLastIfPrevAndIf(receive, { it !is Send }, { isBufferEmpty })

  // AbstractSendChannel
  protected open fun enqueueSend(send: Send): Any? {
      if (isBufferAlwaysFull) {
          queue.addLastIfPrev(send) { prev ->
              if (prev is ReceiveOrClosed<*>) return@enqueueSend prev
              true
          }
      } else {
          if (!queue.addLastIfPrevAndIf(send, { prev ->
              if (prev is ReceiveOrClosed<*>) return@enqueueSend prev
              true
          }, { isBufferFull }))
              return ENQUEUE_FAILED
      }
      return null
  }
  ```

- `tryCondAddNext`
  這個方法整個流程已經在上面提過了，因為它只會在 `addLastIf` 與 `addLastIfPrevAndIf` 調用。


複習一次， 若想要新增 `node` 我們就需要通過以下方法將其插入或加入：`addNext`、 `finishAdd`、 `addLast`、 `addLastIf`、 `addLastIfPrev`、 `addLastIfPrevAndIf` 與 `tryCondAddNext`


### **LIST_N**

|name|state class |public state|description|
|:--|:--|:--|:--|
| **LIST_N**  |  **InactiveNodeList** | New  | a list of listeners (promoted once, does not got back to EmptyNew)  |

想要從 **EMPTY_N** 轉換成 **LIST_N**，我們便會經過調用 `invokeOnCompletion` 進行。 在 `_state` 為 `isActive == false` 且 為 **Empty** 時，通過 `promoteEmptyToNodeList` 將 `_state` 改為 **NodeList**：


```kotlin
private fun promoteEmptyToNodeList(state: Empty) {
    // try to promote it to LIST state with the corresponding state
    val list = NodeList()
    val update = if (state.isActive) list else InactiveNodeList(list)
    _state.compareAndSet(state, update)
}
```
<br>

### **COMPLETING**

|name|state class |public state|description|
|:--|:--|:--|:--|
| **COMPLETING**  |  **Finishing** | Completing  | has a list of listeners (promoted once from LIST_*)  |

```kotlin
// Completing & Cancelling states,
// All updates are guarded by synchronized(this), reads are volatile
@Suppress("UNCHECKED_CAST")
private class Finishing(
    override val list: NodeList,
    isCompleting: Boolean,
    rootCause: Throwable?
) : SynchronizedObject(), Incomplete
```

**Finishing** 也是個 **Incomplete**，雖然它繼承了 **Incomplete** 但我們可以通過 `isCompleting` 來得知 **Job** 是否正在完成。

除此之外，**Finishing** 所代表的是 **COMPLETING** 與 **CANCELLING** 狀態。

想要創造 **Finishing** 就只能通過 `tryMakeCancelling` 與 `tryMakeCompletingSlowPath` 中創建。

但在 **COMPLETING** 狀態下，理應會調用 `tryMakeCompletingSlowPath`。 而 `tryMakeCompletingSlowPath` 中會在 `state` 不是 **Finishing** 時創建 **Finishing** 並更新 `state` 值：

<br>

```kotlin
private fun tryMakeCompletingSlowPath(state: Incomplete, proposedUpdate: Any?): Any? {
    // get state's list or else promote to list to correctly operate on child lists
    val list = getOrPromoteCancellingList(state) ?: return COMPLETING_RETRY
    // promote to Finishing state if we are not in it yet
    // This promotion has to be atomic w.r.t to state change, so that a coroutine that is not active yet
    // atomically transition to finishing & completing state
    val finishing = state as? Finishing ?: Finishing(list, false, null)
    // must synchronize updates to finishing state
    var notifyRootCause: Throwable? = null
    synchronized(finishing) {
        // check if this state is already completing
        if (finishing.isCompleting) return COMPLETING_ALREADY
        // mark as completing
        finishing.isCompleting = true
        // if we need to promote to finishing then atomically do it here.
        // We do it as early is possible while still holding the lock. This ensures that we cancelImpl asap
        // (if somebody else is faster) and we synchronize all the threads on this finishing lock asap.
        if (finishing !== state) {
            if (!_state.compareAndSet(state, finishing)) return COMPLETING_RETRY
        }
        // ## IMPORTANT INVARIANT: Only one thread (that had set isCompleting) can go past this point
        assert { !finishing.isSealed } // cannot be sealed
        // add new proposed exception to the finishing state
        val wasCancelling = finishing.isCancelling
        (proposedUpdate as? CompletedExceptionally)?.let { finishing.addExceptionLocked(it.cause) }
        // If it just becomes cancelling --> must process cancelling notifications
        notifyRootCause = finishing.rootCause.takeIf { !wasCancelling }
    }
    // process cancelling notification here -- it cancels all the children _before_ we start to to wait them (sic!!!)
    notifyRootCause?.let { notifyCancelling(list, it) }
    // now wait for children
    val child = firstChild(state)
    if (child != null && tryWaitForChild(finishing, child, proposedUpdate))
        return COMPLETING_WAITING_CHILDREN
    // otherwise -- we have not children left (all were already cancelled?)
    return finalizeFinishingState(finishing, proposedUpdate)
}
```


### **CANCELLING**

|name|state class |public state|description|
|:--|:--|:--|:--|
| **CANCELLING**  |  **Finishing** | Cancelling  | -- " --  |


這時會調用 `tryMakeCancelling`。 它所創建的 **Finishing** 與 `tryMakeCompleting` 所創建的有些許不同。 `tryMakeCompleting` 的 `rootCause` 會是 `null`，但 `tryMakeCancelling` 中的則需要傳入 `rootCause`：

<br>

```kotlin
// try make new Cancelling state on the condition that we're still in the expected state
private fun tryMakeCancelling(state: Incomplete, rootCause: Throwable): Boolean {
    assert { state !is Finishing } // only for non-finishing states
    assert { state.isActive } // only for active states
    // get state's list or else promote to list to correctly operate on child lists
    val list = getOrPromoteCancellingList(state) ?: return false
    // Create cancelling state (with rootCause!)
    val cancelling = Finishing(list, false, rootCause)
    if (!_state.compareAndSet(state, cancelling)) return false
    // Notify listeners
    notifyCancelling(list, rootCause)
    return true
}
```

### **FINAL_C**

|name|state class |public state|description|
|:--|:--|:--|:--|
| **FINAL_C**  |  **Cancelled** | Cancelling  | Cancelled (final state)  |

此時的 `state` 會是 **CompletedExceptionally** 或它的繼承者 **CancelledContinuation**。

這兩個類別的作用是在將 exception 經由他們傳出去。

<br>

```kotlin
internal open class CompletedExceptionally(
    @JvmField public val cause: Throwable,
    handled: Boolean = false
) {
    private val _handled = atomic(handled)
    val handled: Boolean get() = _handled.value
    fun makeHandled(): Boolean = _handled.compareAndSet(false, true)
    override fun toString(): String = "$classSimpleName[$cause]"
}

internal class CancelledContinuation(
    continuation: Continuation<*>,
    cause: Throwable?,
    handled: Boolean
) : CompletedExceptionally(cause ?: CancellationException("Continuation $continuation was cancelled normally"), handled) {
    private val _resumed = atomic(false)
    fun makeResumed(): Boolean = _resumed.compareAndSet(false, true)
}
```


### **FINAL_R**

|name|state class |public state|description|
|:--|:--|:--|:--|
| **FINAL_C**  |  **\<any\>** | Completed  | produced some result |

這個時候其實整個流程就已經結束，它會將資料或是 `cause` 回傳到需要的地方。



## JobImpl

**JobImpl** 除了是一個 **JobSupport** 還是一個 **CompletableJob**。

由於 **JobSupport** 的 `active` 預設為 `true` 所以它會通過 `initParentJob(Job?)` 設定好 `parentHandle`。
另外，他會通過 **JobSupport** 的 `makeCompleting(Any?)` 來進行完成：

```kotlin
@PublishedApi // for a custom job in the test module
internal open class JobImpl(parent: Job?) : JobSupport(true), CompletableJob {
    init { initParentJob(parent) }
    override val onCancelComplete get() = true
    /*
     * Check whether parent is able to handle exceptions as well.
     * With this check, an exception in that pattern will be handled once:
     * ```
     * launch {
     *     val child = Job(coroutineContext[Job])
     *     launch(child) { throw ... }
     * }
     * ```
     */
    override val handlesException: Boolean = handlesException()
    override fun complete() = makeCompleting(Unit)
    override fun completeExceptionally(exception: Throwable): Boolean =
        makeCompleting(CompletedExceptionally(exception))

    @JsName("handlesExceptionF")
    private fun handlesException(): Boolean {
        var parentJob = (parentHandle as? ChildHandleNode)?.job ?: return false
        while (true) {
            if (parentJob.handlesException) return true
            parentJob = (parentJob.parentHandle as? ChildHandleNode)?.job ?: return false
        }
    }
}
```

### SupervisorJobImpl

**SupervisorJobImpl** 唯一的功能是 攔截 child 拋出的 `cause`，進而阻止其他 children 的終止。

```kotlin
private class SupervisorJobImpl(parent: Job?) : JobImpl(parent) {
    override fun childCancelled(cause: Throwable): Boolean = false
}
```

而這個方法 **只會** 被 **ChildHandleNode** 調用：

```kotlin
internal class ChildHandleNode(
    @JvmField val childJob: ChildJob
) : JobCancellingNode(), ChildHandle {
    override val parent: Job get() = job
    override fun invoke(cause: Throwable?) = childJob.parentCancelled(job)
    override fun childCancelled(cause: Throwable): Boolean = job.childCancelled(cause)
}
```

**ChildHandleNode** 所調用的 `job` 其實就是 **JobNode** 中的 **JobSupport**：

```kotlin
internal abstract class JobNode : CompletionHandlerBase(), DisposableHandle, Incomplete {
    /**
     * Initialized by [JobSupport.makeNode].
     */
    lateinit var job: JobSupport

    // ...
}
```

**JobSupport** 則會由 `makeNode()` 創建時傳如：
```kotlin
private fun makeNode(handler: CompletionHandler, onCancelling: Boolean): JobNode {
    val node = if (onCancelling) {
        (handler as? JobCancellingNode)
            ?: InvokeOnCancelling(handler)
    } else {
        (handler as? JobNode)
            ?.also { assert { it !is JobCancellingNode } }
            ?: InvokeOnCompletion(handler)
    }

    // 這裡設定 JobSupport
    node.job = this
    return node
}
```




# LockFreeLinkedListNode
這便是 **Job** 改變 **state** 時最重要的元素了。

```kotlin
@Suppress("LeakingThis")
@InternalCoroutinesApi
public actual open class LockFreeLinkedListNode {
    private val _next = atomic<Any>(this) // Node | Removed | OpDescriptor
    private val _prev = atomic(this) // Node to the left (cannot be marked as removed)
    private val _removedRef = atomic<Removed?>(null)

    // ... 由於程式太長所以忽略，你可以想像 LockFreeLinkedListNode 就是一個會進行 atomic op 的 LinkedList
}
```

## JobNode / JobCancellingNode
**JobNode** 是個抽象類別。 同時，它也是  **Incomplete** 、 **DisposableHandle** 與 **CompletionHandlerBase**。 它裡還包含著一個 **JobSupport**，並可以通過 **JobSupport** 的 `makeNode` 被創建。

>
>
> **JobNode** 是將 **JobSupport** 的 `state` 從 **EMPTY** 到 **SINGLE** 的關鍵。
> <br>

注意一點， **JobNode** 的預設並不包含 **NodeList**，但它也是一個 **LockFreeLinkedListNode**，而且狀態為 `active`。 另外，它預設會有一個 `listener` 也就是 **JobSupport**。

```kotlin
internal abstract class JobNode : CompletionHandlerBase(), DisposableHandle, Incomplete {
    /**
     * Initialized by [JobSupport.makeNode].
     */
    lateinit var job: JobSupport
    override val isActive: Boolean get() = true   // Incomplete
    override val list: NodeList? get() = null     // Incomplete
    override fun dispose() = job.removeNode(this) // DisposableHandle
    override fun toString() = "$classSimpleName@$hexAddress[job@${job.hexAddress}]"
}
```

**CompletionHandlerBase** 是一個 **LockFreeLinkedListNode** 的抽象類別：

```kotlin
internal actual abstract class CompletionHandlerBase actual constructor() : LockFreeLinkedListNode(), CompletionHandler {
    actual abstract override fun invoke(cause: Throwable?)
}
```

><br>
>
>此時請記住一點：**CompletionHandlerBase** 的 `invoke` 的方法會在 **JobSupport** 中的 `completeStateFinalization(Incomplete, Any?)` 中調用。
><br>

**DisposableHandle** 則是一個介面。 它可以通過調用 `dispose` 的方法來進行 GC：
```kotlin
public fun interface DisposableHandle {
    /**
     * Disposes the corresponding object, making it eligible for garbage collection.
     * Repeated invocation of this function has no effect.
     */
    public fun dispose()
}
```

從 **SINGLE** 狀態講解中，我們得知 **JobNode** 的創建是通過 `makeNode` 達成的。 `makeNode` 會通過 **InvokeOnCancelling** 及 **InvokeOnCompletion** 這兩個類別分別創建出只能被 `invoke` 一次和可被 `invoke` 數次的 **JobNode**。

**JobNode** 的創建會在 **JobSupport** 的 `invokeOnCompletion` 中進行：

```kotlin
public final override fun invokeOnCompletion(
    onCancelling: Boolean,
    invokeImmediately: Boolean,
    handler: CompletionHandler
): DisposableHandle {
    // Create node upfront -- for common cases it just initializes JobNode.job field,
    // for user-defined handlers it allocates a JobNode object that we might not need, but this is Ok.
    val node: JobNode = makeNode(handler, onCancelling)
    // ...
}
```

`invokeOnCompletion` 會使用這個 `node` 達成以下目的：
- 將 `state` 從 **Empty** 更新為 **SINGLE**
- 當 `state` 是 **Incomplete** 和 **Finishing** 而且正在被取消時，它會在確保 `rootCause == null` 或 此 **JobSupport** 是一個尚未結束的 **Coroutine** 後將 `node` 通過 `addLastAtomic` 插入其中
  ```kotlin
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
  ```
- 當 `state` 是 **Incomplete** 但沒有被取消，且 `rootCause` 為 `null` 時，也會將 `node` 通過 `addLastAtomic` 插入其中


所以 **JobSupport** 會在
1. `state` 是 **Empty** 且為 `active` 時，將 `state` 更新成 `node` ( **Empty** => **SINGLE** )
2. 被取消但沒有 `rootCause` 的情況下將 **InvokeOnCancelling** 加到 `list` 的最後
3. 被取消但尚未結束的 **Coroutine** 也會將 **InvokeOnCancelling** 加到 `list` 的最後
4. `state` 是 **Incomplete** 且 `rootCause == null` 時將 `node` 加到 `list` 的最後

`rootCause` 若不是 `null` 表示我們尚未調用 `handler` 也就是尚未被取消。以下便是 **CompletionHandler**：

```kotlin
public typealias CompletionHandler = (cause: Throwable?) -> Unit
```

**JobNode** 的繼承者包括：

|File|Class|Super|
|:--|:--|:--|
|Await.kt   | **AwaitAll.AwaitAllNode**  |**JobNode**|
|Future.kt   | **CancelFutureOnCompletion**   |**JobNode**|
|JobSupport.kt   | **JobSupport.ChildCompletion**\n**ChildHandleNode**\n**ChildContinuation**\n**DisposeOnCompletion**\n****SelectJoinOnCompletion**\n**SelectAwaitOnCompletion**\n**InvokeOnCancelling**\n**InvokeOnCompletion** \n**ResumeOnCompletion**\n**ResumeAwaitOnCompletion**|**JobNode**|

我們從 **JobNode** 類別可以看出它與 **NodeList** 及其他的 **LockFreeLinkedListNode** 關係是：

```
        state              state
     +---------+        +----------+
     |         |        |          |
     | JobNode |  -->   | NodeList | --> state ...
     |         |        |          |
     +---------+        +----------+
                             |
                             v
               +---------------------------+
               |                           |
               | LockFreeLinkedListNode(s) |
               |                           |
               +---------------------------+
```



### JobCancellingNode

```kotlin
internal abstract class JobCancellingNode : JobNode()
```

#### ChildHandleNode

**ChildHandleNode** 是一個 **ChildJob** 的 handler。 它主要負責的是進行 `child` 的取消以及告知 parent \<**Job**\> `child` 取消的原因：

```kotlin
internal class ChildHandleNode(
    @JvmField val childJob: ChildJob
) : JobCancellingNode(), ChildHandle {
    override val parent: Job get() = job
    /* parent 取消 child 時調用的方法 */
    override fun invoke(cause: Throwable?) = childJob.parentCancelled(job)
    /* ChildHandle - 告知 parent child 取消原因，並由 parent 決定是否要一同被取消或是處理此 Throwable */
    override fun childCancelled(cause: Throwable): Boolean = job.childCancelled(cause)
}
```

`childJob.parentCancelled(job)` 的實作可以從 **JobSupport** 看到：

```kotlin
public final override fun parentCancelled(parentJob: ParentJob) {
    cancelImpl(parentJob)
}
```

通過 `cancelImpl` **JobSupport** 會有以下行為：
- 若 `onCancelComplete == true` 便會通過 `tryMakeCompleting` 嘗試更新 `state` 為 **CompletedExceptionally**。 若此時的 `state` 是個 **ChildHandleNode**，那便會導致 `finalState` 得到 **COMPLETING_WAITING_CHILDREN**，最後便直接 return。
- 若 `state` 原來原本就完成，就直接調用 `makeCancelling` 來更新 `rootCause` 或 更新 `exceptionsHolder`。
- 若當下 **JobSupport** 已完成、正在等待 Children 完成 或 `!== TOO_LATE_TO_CANCEL`，那就會回傳 `true`。
  但第三種情況會先調用 `afterCompletion(Any?)` 再回傳 `true`。 而 `afterCompletion` 預設沒有行為。 只有 **BlockingCoroutine**、 **DispatchedCoroutine** 與 **ScopeCoroutine** 才會進行覆寫。
  <br>
  - **BlockingCoroutine** 會在 `Thread.currentThread() != blockedThread` 時進行 `unpark(blockedThread)`
  - **DispatchedCoroutine** 會在 `_decision === SUSPENDED` 時通過 `recoverResult` 從 `uCont` \<**Continuation**\> 與 `state` 取得 **Result**。 最後便會將結果通過 **Continuation**，可能是 **DispatchedContinuation** 或 **ContinuationImpl**，的 `resumeCancellableWith` 將 **Result** 傳回。
  - **ScopeCoroutine** 與 **DispatchedCoroutine** 有同樣行為。

```kotlin
// cause is Throwable or ParentJob when cancelChild was invoked
// returns true is exception was handled, false otherwise
internal fun cancelImpl(cause: Any?): Boolean {
    var finalState: Any? = COMPLETING_ALREADY

    /*
      only JobImpl [SupervisorJobImpl] and CompletableDeferredImpl onCancelComplete == true
    */
    if (onCancelComplete) {
        // make sure it is completing, if cancelMakeCompleting returns state it means it had make it
        // completing and had recorded exception
        finalState = cancelMakeCompleting(cause)
        if (finalState === COMPLETING_WAITING_CHILDREN) return true
    }
    if (finalState === COMPLETING_ALREADY) {
        finalState = makeCancelling(cause)
    }
    return when {
        finalState === COMPLETING_ALREADY -> true
        finalState === COMPLETING_WAITING_CHILDREN -> true
        finalState === TOO_LATE_TO_CANCEL -> false
        else -> {
            afterCompletion(finalState)
            true
        }
    }
}
```

##### ChildHandleNode 的創建

從源碼中我們發現 **ChildHandleNode** 只會在 **JobSupport** 的 `attachChild` 才會被創建：

```kotlin
@Suppress("OverridingDeprecatedMember")
public final override fun attachChild(child: ChildJob): ChildHandle {
    return invokeOnCompletion(onCancelling = true, handler = ChildHandleNode(child).asHandler) as ChildHandle
}
```

而 `attachChild` 也只有在 **JobSupport** 的 `initParentJob(Job?)` 中通過這個 **Job** 調用：

```kotlin
protected fun initParentJob(parent: Job?) {
    assert { parentHandle == null }
    if (parent == null) {
        parentHandle = NonDisposableHandle
        return
    }
    parent.start() // make sure the parent is started
    @Suppress("DEPRECATION")
    val handle = parent.attachChild(this)
    parentHandle = handle
    // now check our state _after_ registering (see tryFinalizeSimpleState order of actions)
    if (isCompleted) {
        handle.dispose()
        parentHandle = NonDisposableHandle // release it just in case, to aid GC
    }
}
```

從此源碼可以看到 `attachChild` 回傳的 **ChildHandleNode** 便是這個 **JobSupport** 的 `parentHandle`。

<u><b>其實這個命名有點混淆，為何 **ChildHandle** 在 **JobSupport** 中會被稱為 `parentHandle` 呢？</b></u>

這是因為 **ChildHandle** 的主要功能就是讓 `child` 試圖對 `parent` 進行取消。當然，最後 `parent` 是否被取消，那就由 `parent` 來決定了。

##### parent 取消於否

我們現在知道當 `child` 被取消時，`child` 可以通過 **Job** 的 `childCancelled` 將 **Throwable** 傳給 `job` 或 `parent` 並讓它決定如何處理此 **Throwable**：

```kotlin
override fun childCancelled(cause: Throwable): Boolean = job.childCancelled(cause)
```

<u><b>那 `parent` 又會如何處理 **Throwable** 呢？</b></u>

其實 `Job.childCancelled` 有三種版本：
- **JobSupport** ( 預設 )
  ```kotlin
  public open fun childCancelled(cause: Throwable): Boolean {
      if (cause is CancellationException) return true
      return cancelImpl(cause) && handlesException
  }
  ```

- **FlowCoroutine** / **FlowProduceCoroutine**
  ```kotlin
  override fun childCancelled(cause: Throwable): Boolean {
      if (cause is ChildCancelledException) return true
      return cancelImpl(cause) // JobSupport
  }
  ```
- **SupervisorCoroutine** / **SupervisorJobImpl**
  ```kotlin
  override fun childCancelled(cause: Throwable): Boolean = false
  ```

以上的 `childCancelled` 中所調用的 `cancelImpl` 行為是由 **JobSupport** 定義。 它會嘗試將其他的 `child`，可能會是 **ChildHandleNode** 或 **NodeList**， 進行取消。

<u><b>那 `childCancelled` 的結果又會有什麼後果呢？</b></u>

**ChildHandle** 的 `childCancelled` 的結果只會在 **JobSupport** 的 `cancelParent` 中調用：

```kotlin
private fun cancelParent(cause: Throwable): Boolean {
    // Is scoped coroutine -- don't propagate, will be rethrown
    if (isScopedCoroutine) return true

    /* CancellationException is considered "normal" and parent usually is not cancelled when child produces it.
     * This allow parent to cancel its children (normally) without being cancelled itself, unless
     * child crashes and produce some other exception during its completion.
     */
    val isCancellation = cause is CancellationException
    val parent = parentHandle
    // No parent -- ignore CE, report other exceptions.
    if (parent === null || parent === NonDisposableHandle) {
        return isCancellation
    }

    // Notify parent but don't forget to check cancellation
    return parent.childCancelled(cause) || isCancellation
}
```

而此結果便會在 `finalizeFinishingState(Finishing, Any?)` 中使用：

```kotlin
// Now handle the final exception
if (finalException != null) {
    val handled = cancelParent(finalException) || handleJobException(finalException)
    if (handled) (finalState as CompletedExceptionally).makeHandled()
}
```

當 `handled == true`， **JobSupport** 便會調用 `finalState` 的 `makeHandled()` 來更新 **CompleteExceptionally** 的 `_handled` 成 `true`。

最後 `finalizeFinishingState` 還會調用 `onCompletionInternal(Any?)`。 雖然這方法預設是沒有行為，但卻會在 **AbstractCoroutine** 中實作：

```kotlin
@Suppress("UNCHECKED_CAST")
protected final override fun onCompletionInternal(state: Any?) {
    if (state is CompletedExceptionally)
        onCancelled(state.cause, state.handled)
    else
        onCompleted(state as T)
}
```

此時 **AbstractCoroutine** 會調用 `onCancelled(Throwable, Boolean)`。 這方法在 **AbstractCoroutine** 雖然也是沒有行為的，但我們可以從 **BroadcastCoroutine** 與 **ProduceCoroutine** 中看到實作：

```kotlin
override fun onCancelled(cause: Throwable, handled: Boolean) {
    val processed = _channel.close(cause)
    if (!processed && !handled) handleCoroutineException(context, cause)
}
```

><br>
>
>我們可以從以上分析出， 如果 **ChildHandle** 的 `childCancelled` 回傳為 `false` 時 **AbstractCoroutine** 便可能會調用 `handleCoroutineException(CoroutineContext, Throwable)` 來讓 context 中的 **CoroutineExceptionHandler** 處理此事。 而於此同時 **BroadcastCoroutine** 與 **ProduceCoroutine** 也會調用 `_channel.close(cause)`。
><br>

```kotlin
// AndroidExceptionPreHandler
override fun handleException(context: CoroutineContext, exception: Throwable) {
    /*
     * Android Oreo introduced private API for a global pre-handler for uncaught exceptions, to ensure that the
     * exceptions are logged even if the default uncaught exception handler is replaced by the app. The pre-handler
     * is invoked from the Thread's private dispatchUncaughtException() method, so our manual invocation of the
     * Thread's uncaught exception handler bypasses the pre-handler in Android Oreo, and uncaught coroutine
     * exceptions are not logged. This issue was addressed in Android Pie, which added a check in the default
     * uncaught exception handler to invoke the pre-handler if it was not invoked already (see
     * https://android-review.googlesource.com/c/platform/frameworks/base/+/654578/). So the issue is present only
     * in Android Oreo.
     *
     * We're fixing this by manually invoking the pre-handler using reflection, if running on an Android Oreo SDK
     * version (26 and 27).
     */
    if (Build.VERSION.SDK_INT in 26..27) {
        (preHandler()?.invoke(null) as? Thread.UncaughtExceptionHandler)
            ?.uncaughtException(Thread.currentThread(), exception)
    }
}
```

所以我們現在知道， `parent.childCancelled` 除了 **SupervisorCoroutine** / **SupervisorJobImpl** 外，都會試圖通過 `cancelImpl` 來取消其他的 `child`。 如果 `parent.childCancelled` 回傳為 `false`，這就表示 **Job** 已經無法被取消。 這時就可能調用 **CoroutineExceptionHandler** 的 `handleCoroutineException` 來讓當下的 **Thread** 知道 **UncaughtException**。

#### ChildContinuation

**ChildContinuation** 與 **ChildHandleNode** 很像， 但缺少了 **ChildHandle** 的行為：

```kotlin
// Same as ChildHandleNode, but for cancellable continuation
internal class ChildContinuation(
    @JvmField val child: CancellableContinuationImpl<*>
) : JobCancellingNode() {
    override fun invoke(cause: Throwable?) {
        child.parentCancelled(child.getContinuationCancellationCause(job))
    }
}
```

所以 **ChildContinuation** 無法通過 `childCancelled(Throwable)` 將 **Throwable** 傳給 `parent: Job`，也就無法讓 **Job** 進行 `child` 的取消。

而 **ChildContinuation** 的 `getContinuationCancellationCause(Job)` 會通過 `parent: Job` 的 `getCancellationException` 取得 **Throwable**。 此 **Throwable** 便會傳入 `child` 的 `parentCancelled(Throwable)`：

```kotlin
@PublishedApi
internal open class CancellableContinuationImpl<in T>(
    final override val delegate: Continuation<T>,
    resumeMode: Int
) : DispatchedTask<T>(resumeMode), CancellableContinuation<T>, CoroutineStackFrame {

  private fun installParentHandle(): DisposableHandle? {
      val parent = context[Job] ?: return null // don't do anything without a parent
      // Install the handle
      val handle = parent.invokeOnCompletion(
          onCancelling = true,
          handler = ChildContinuation(this).asHandler
      )
      parentHandle = handle
      return handle
  }

  internal fun parentCancelled(cause: Throwable) {
      if (cancelLater(cause)) return
      cancel(cause)
      detachChildIfNonResuable()
  }
}
```

**ChildContinuation** 的 `invoke` 會有以下行為：
1. 調用 `child.getContinuationCancellationCause(Job)` 並通過 `parent.getCancellationException()` 跟 **Job** 取得取消原由 ( **Throwable** )
2. 調用 **CancelledContinuationImpl** 的 `parentCancelled` 將 **Throwable** 傳入
3. `parentContext` 會試圖通過 **DispatchedContinuation** 或 直接將 `_state` 更新為 **CancelledContinuation** 來取消自己。
4. 最後還會通過 `detachChild()` 調用 `handle.dispose()`。 此時的 `handle` 應該是 **ChildContinuation**。

`invoke` 的方法會在 **JobSupport** 的 `completeStateFinalization` 中調用：

```kotlin
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


#### InvokeOnCancelling

**InvokeOnCancelling** 與 **InvokeOnCompletion** 差不多一模一樣，但差在 `invoke` 的行為：

```kotlin
private class InvokeOnCancelling(
    private val handler: CompletionHandler
) : JobCancellingNode()  {
    // delegate handler shall be invoked at most once, so here is an additional flag
    private val _invoked = atomic(0) // todo: replace with atomic boolean after migration to recent atomicFu
    override fun invoke(cause: Throwable?) {
        if (_invoked.compareAndSet(0, 1)) handler.invoke(cause)
    }
}
```

**InvokeOnCompletion** 的行為：

```kotlin
override fun invoke(cause: Throwable?) = handler.invoke(cause)
```

從此可以知道 **InvokeOnCompletion** 可以被多次調用，但 **InvokeOnCancelling** 只能被調用一次。

這兩者皆在 `makeNode` 中創建的：

```kotlin
private fun makeNode(handler: CompletionHandler, onCancelling: Boolean): JobNode {
    val node = if (onCancelling) {
        (handler as? JobCancellingNode)
            ?: InvokeOnCancelling(handler)
    } else {
        (handler as? JobNode)
            ?.also { assert { it !is JobCancellingNode } }
            ?: InvokeOnCompletion(handler)
    }
    node.job = this
    return node
}
```

#### SelectBuilderImpl.SelectOnCancelling
```kotlin
private inner class SelectOnCancelling : JobCancellingNode() {
    // Note: may be invoked multiple times, but only the first trySelect succeeds anyway
    override fun invoke(cause: Throwable?) {
        if (trySelect())
            resumeSelectWithException(job.getCancellationException())
    }
}
```

**SelectOnCancelling** 的 `invoke` 中會通過 `trySelect()` 來試圖更新 `_state` 並選擇當下 **SelectOnCancelling** ：

```kotlin
override fun trySelect(): Boolean {
    val result = trySelectOther(null)
    return when {
        result === RESUME_TOKEN -> true // was selected with this marker
        result == null -> false // already selected or selected with different marker
        else -> error("Unexpected trySelectIdempotent result $result") // abort
    }
}
```





`resumeSelectWithException` 會試圖取得 **ContinuationInterceptor** 並調用 `resumeWith` 將 **Result.failure** 傳出：

```kotlin
// Resumes in dispatched way so that it can be called from an arbitrary context
override fun resumeSelectWithException(exception: Throwable) {
    doResume({ CompletedExceptionally(recoverStackTrace(exception, uCont)) }) {
        uCont.intercepted().resumeWith(Result.failure(exception))
    }
}
```

#### CompletionHandlerBase

#### ChildCompletion

#### CancelFutureOnCompletion





#### ResumeAwaitOnCompletion




#### AwaitAll.AwaitAllNode

```kotlin
private inner class AwaitAllNode(private val continuation: CancellableContinuation<List<T>>) : JobNode()
```

{::comment}
===================== JobSupport.## state =====================
{:/comment}

## CompletableDeferredImpl
```kotlin
/**
 * Concrete implementation of [CompletableDeferred].
 */
@Suppress("UNCHECKED_CAST")
private class CompletableDeferredImpl<T>(
    parent: Job?
) : JobSupport(true), CompletableDeferred<T>, SelectClause1<T> {
    init { initParentJob(parent) }
    override val onCancelComplete get() = true
    override fun getCompleted(): T = getCompletedInternal() as T
    override suspend fun await(): T = awaitInternal() as T
    override val onAwait: SelectClause1<T> get() = this
    override fun <R> registerSelectClause1(select: SelectInstance<R>, block: suspend (T) -> R) =
        registerSelectClause1Internal(select, block)

    override fun complete(value: T): Boolean =
        makeCompleting(value)
    override fun completeExceptionally(exception: Throwable): Boolean =
        makeCompleting(CompletedExceptionally(exception))
}
```




## AbstractCoroutine

**AbstractCoroutine** 其實是一個可被設定 active 於否的 **JobSupport**。 除此之外，擁有 **CoroutineScope** 能力的也就表示他會擁有一個 **CoroutineContext** ：

```kotlin
@InternalCoroutinesApi
public abstract class AbstractCoroutine<in T>(
    parentContext: CoroutineContext,
    initParentJob: Boolean,
    active: Boolean
) : JobSupport(active), Job, Continuation<T>, CoroutineScope
```

**AbstractCoroutine** 會在 `init` 時調用 **JobSupport** 的 `initParentJob`：
```kotlin
init {
    if (initParentJob) initParentJob(parentContext[Job])
}
```

而他的 **CoroutineContext** 除了 `parentContext` 外，它自己也被算入：

```kotlin
@Suppress("LeakingThis")
public final override val context: CoroutineContext = parentContext + this
```

這是因為它自己是個 **Job: CoroutineContext.Element**。

在 **AbstractCoroutine** 中，有以下的方法可以被繼承者覆寫：
```kotlin
protected open fun onCompleted(value: T) {}
protected open fun onCancelled(cause: Throwable, handled: Boolean) {}
protected open fun afterResume(state: Any?): Unit = afterCompletion(state)

// JobSupport
protected open fun afterCompletion(state: Any?) {}
```

那有誰繼承呢？

|Overried Method|Class|
|:--|:--|
| `onCompleted`   | **BroadcastCoroutine**\n**ProduceCoroutine**  |
| `onCancelled`  | 同上  |
| `afterResume`   |  **DispatchedCoroutine**\n**ScopeCoroutine**\n**UndispatchedCoroutine** |


### BroadcastCoroutine

**BroadcastCoroutine** 是一個不會啟動 `parentJob` 的 **AbstractCoroutine**：

```kotlin
private open class BroadcastCoroutine<E>(
    parentContext: CoroutineContext,
    protected val _channel: BroadcastChannel<E>,
    active: Boolean
) : AbstractCoroutine<Unit>(parentContext, initParentJob = false, active = active),
    ProducerScope<E>, BroadcastChannel<E> by _channel
```

由於 `initParentJob == false`，所以不會調用 `initParentJob(Job)`。 也就表示 `parentHandle: ChildHandle? == null`，並且無法調用 **DisposableHandle** 的 `dispose()`：
```kotlin
if (initParentJob) initParentJob(parentContext[Job])
```

但同時因為這個原因，當 `cancelParent(Throwable)` 被調用時，它會直接回傳 `isCancelling`：

```kotlin
val isCancellation = cause is CancellationException
if (parent === null || parent === NonDisposableHandle) {
    return isCancellation
}
```

再來我們看看 **BroadcastCoroutine** 所覆寫的方法。

```kotlin
override fun onCompleted(value: Unit) {
    _channel.close()
}

override fun onCancelled(cause: Throwable, handled: Boolean) {
    val processed = _channel.close(cause)
    if (!processed && !handled) handleCoroutineException(context, cause)
}
```

我們可以看到 `onCompleted(Unit)` 的主要功能是調用 **BroadcastChannel\<E\>** 的 `close(Throwable)`。 這個方法其實是 **SendChannel\<E\>** 的介面方法：

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



#### LazyBroadcastCoroutine
```kotlin
private class LazyBroadcastCoroutine<E>(
    parentContext: CoroutineContext,
    channel: BroadcastChannel<E>,
    block: suspend ProducerScope<E>.() -> Unit
) : BroadcastCoroutine<E>(parentContext, channel, active = false)
```



#### 暸解 BroadcastCoroutine


### ProduceCoroutine


### DispatchedCoroutine


### ScopeCoroutine


### UndispatchedCoroutine


# CompletedExceptionally

**CompleteExceptionally** 是一個擁有 **Throwable** 的類別。 通過 `makeHandled`， 他的是否 handled 的值會從 `false` 改為 `true`：

```kotlin
internal open class CompletedExceptionally(
    @JvmField public val cause: Throwable,
    handled: Boolean = false
) {
    private val _handled = atomic(handled)
    val handled: Boolean get() = _handled.value
    fun makeHandled(): Boolean = _handled.compareAndSet(false, true)
    override fun toString(): String = "$classSimpleName[$cause]"
}
```

**CompleteExceptionally** 會在以下地方建立：

```kotlin
/*
FILE                              | PACKAGE                     | CLASS                     | METHOD
---------------------------------- ----------------------------- --------------------------- -----------------------
CompletionState.kt                  kotlinx.coroutines            Result<T>                   toState
CompletableDeferred.kt              kotlinx.coroutines            CompletableDeferredImpl     completeExceptionally
CancellableContinuationImpl.kt      kotlinx.coroutines            CoroutineDispatcher         resumeUndispatchedWithException
JobSupport.kt                       ...                           JobSupport                  finalizeFinishingState
                                                                                              makeCancelling
                                                                                              cancelMakeCompleting
Undispatched.kt                     kotlinx.coroutines.intrinsics  ScopeCoroutine             undispatchedResult
Select.kt                           kotlinx.coroutines.selects    SelectBuilderImpl           resumeSelectWithException
*/
```

我們來看看這些創建 **CompleteExceptionally** 的方法吧。

## 創建 CompleteExceptionally 的方法

### ScopeCoroutine

**ScopeCoroutine** 是個 **AbstractCoroutine** ：
```kotlin
/**
 * This is a coroutine instance that is created by [coroutineScope] builder.
 */
internal open class ScopeCoroutine<in T>(
    context: CoroutineContext,
    @JvmField val uCont: Continuation<T> // unintercepted continuation
) : AbstractCoroutine<T>(context, true, true), CoroutineStackFrame
```

他會通過 `undispatchedResult` 中調用 `startBlock` 時將 **Exception** 給接住。 最後便會將這個 exception 放在 **CompleteExceptionally** 中：


```kotlin
private inline fun <T> ScopeCoroutine<T>.undispatchedResult(
    shouldThrow: (Throwable) -> Boolean,
    startBlock: () -> Any?
): Any? {
    val result = try {
        startBlock()
    } catch (e: Throwable) {
        CompletedExceptionally(e)
    }
    /*
     * We're trying to complete our undispatched block here and have three code-paths:
     * (1) Coroutine is suspended.
     * Otherwise, coroutine had returned result, so we are completing our block (and its job).
     * (2) If we can't complete it or started waiting for children, we suspend.
     * (3) If we have successfully completed the coroutine state machine here,
     *     then we take the actual final state of the coroutine from makeCompletingOnce and return it.
     *
     * shouldThrow parameter is a special code path for timeout coroutine:
     * If timeout is exceeded, but withTimeout() block was not suspended, we would like to return block value,
     * not a timeout exception.
     */
    if (result === COROUTINE_SUSPENDED) return COROUTINE_SUSPENDED // (1)
    val state = makeCompletingOnce(result)
    if (state === COMPLETING_WAITING_CHILDREN) return COROUTINE_SUSPENDED // (2)
    return if (state is CompletedExceptionally) { // (3)
        when {
            shouldThrow(state.cause) -> throw recoverStackTrace(state.cause, uCont)
            result is CompletedExceptionally -> throw recoverStackTrace(result.cause, uCont)
            else -> result
        }
    } else {
        state.unboxState()
    }
}
```

### Select

**SelectBuilderImpl** 會在 `resumeSelectWithException` 中創建 **CompletedExceptionally**：

```kotlin
// Resumes in dispatched way so that it can be called from an arbitrary context
override fun resumeSelectWithException(exception: Throwable) {
    doResume({ CompletedExceptionally(recoverStackTrace(exception, uCont)) }) {
        uCont.intercepted().resumeWith(Result.failure(exception))
    }
}
```

而這個方法則會由以下地方調用：
- **SelectBuilderImpl.SelectOnCancelling** 的 `invoke`
- **JobSupport** 的 `registerSelectClause1Internal`、`selectAwaitCompletion` 和 `resumeSendClosed`
- **AbstractSendChannel.SendSelect** 的 `resumeSendClosed`
- **ConflatedBroadcastChannel** 中的 `registerSelectSend`



### Result

**Result** 是一個包括著 `value` 的類別：
```kotlin
/**
 * A discriminated union that encapsulates a successful outcome with a value of type [T]
 * or a failure with an arbitrary [Throwable] exception.
 */
@SinceKotlin("1.3")
@JvmInline
public value class Result<out T> @PublishedApi internal constructor(
    @PublishedApi
    internal val value: Any?
) : Serializable
```

我們可以通過以下參數來檢查 **Result** 是成功與否：

```kotlin
public val isSuccess: Boolean get() = value !is Failure
public val isFailure: Boolean get() = value is Failure

public fun exceptionOrNull(): Throwable? =
    when (value) {
        is Failure -> value.exception
        else -> null
    }
```

與 **Result** 相反的 **Result.Failure** 則是包含 **Throwable** 的類別：
```kotlin
internal class Failure(
    @JvmField
    val exception: Throwable
) : Serializable {
    override fun equals(other: Any?): Boolean = other is Failure && exception == other.exception
    override fun hashCode(): Int = exception.hashCode()
    override fun toString(): String = "Failure($exception)"
}
```

**Result** 可以通過 `toState` 創建 **CompleteExceptionally**：

```kotlin
internal fun <T> Result<T>.toState(
    onCancellation: ((cause: Throwable) -> Unit)? = null
): Any? = fold(
    onSuccess = { if (onCancellation != null) CompletedWithCancellation(it, onCancellation) else it },
    onFailure = { CompletedExceptionally(it) }
)

internal fun <T> Result<T>.toState(caller: CancellableContinuation<*>): Any? = fold(
    onSuccess = { it },
    onFailure = { CompletedExceptionally(recoverStackTrace(it, caller)) }
)

internal data class CompletedWithCancellation(
    @JvmField val result: Any?,
    @JvmField val onCancellation: (cause: Throwable) -> Unit
)

```

第一個 `toState` 中的 `onSuccess` 會在 `onCancellation != null` 時使用 **CompletedWithCancellation** 包起來：

```kotlin
internal data class CompletedWithCancellation(
    @JvmField val result: Any?,
    @JvmField val onCancellation: (cause: Throwable) -> Unit
)
```

當中的 `fold` 會通過 `exceptionOrNull` 來檢查 `value` 是否為 **Result.Failure**。若為 `true`，那便回傳 `value.exception`。 這個值便會通過 `fold` 將它傳遞至 `onFailure` 中的 **CompletedExceptionally** 中：

```kotlin
@InlineOnly
@SinceKotlin("1.3")
public inline fun <R, T> Result<T>.fold(
    onSuccess: (value: T) -> R,
    onFailure: (exception: Throwable) -> R
): R {
    contract {
        callsInPlace(onSuccess, InvocationKind.AT_MOST_ONCE)
        callsInPlace(onFailure, InvocationKind.AT_MOST_ONCE)
    }
    return when (val exception = exceptionOrNull()) {
        null -> onSuccess(value as T)
        else -> onFailure(exception)
    }
}
```

所以從源碼可以看出， `toState` 會回傳三種不同的狀態：
- `value: R`
- **CompletedWithCancellation**
- **CompleteExceptionally**


當中的 `resumeWithStackTrace` 主要是用來進行 debug 的，所以可以忽略它。

`toState` 會在 `resume` 相關的方法中調用來取得 `result` 的值：

```kotlin
// AbstractCoroutine
/**
 * Completes execution of this with coroutine with the specified result.
 */
public final override fun resumeWith(result: Result<T>) {
    val state = makeCompletingOnce(result.toState())
    if (state === COMPLETING_WAITING_CHILDREN) return
    afterResume(state)
}

// DispatchedContinuation
override fun resumeWith(result: Result<T>) {
    val context = continuation.context
    val state = result.toState()
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

// We inline it to save an entry on the stack in cases where it shows (unconfined dispatcher)
// It is used only in Continuation<T>.resumeCancellableWith
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

// SelectBuilderImpl
// Resumes in direct mode, without going through dispatcher. Should be called in the same context.
override fun resumeWith(result: Result<R>) {
    doResume({ result.toState() }) {
        if (result.isFailure) {
            uCont.resumeWithStackTrace(result.exceptionOrNull()!!)
        } else {
            uCont.resumeWith(result)
        }
    }
}

// CancelledContinuationImpl
override fun resumeWith(result: Result<T>) =
    resumeImpl(result.toState(this), resumeMode)

```

### JobSupport
**JobSupport** 會在 `cancelMakeCompleting` 和 `makeCancelling` 建立 **CompleteExceptionally**。而創建出來的 **CompleteExceptionally** 都會傳入 `tryMakeCompleting`：

```kotlin
// cause is Throwable or ParentJob when cancelChild was invoked
// It contains a loop and never returns COMPLETING_RETRY, can return
// COMPLETING_ALREADY -- if already completed/completing
// COMPLETING_WAITING_CHILDREN -- if started waiting for children
// final state -- when completed, for call to afterCompletion
private fun cancelMakeCompleting(cause: Any?): Any? {
    loopOnState { state ->
        if (state !is Incomplete || state is Finishing && state.isCompleting) {
            // already completed/completing, do not even create exception to propose update
            return COMPLETING_ALREADY
        }
        val proposedUpdate = CompletedExceptionally(createCauseException(cause))
        val finalState = tryMakeCompleting(state, proposedUpdate)
        if (finalState !== COMPLETING_RETRY) return finalState
    }
}


// transitions to Cancelling state
// cause is Throwable or ParentJob when cancelChild was invoked
// It contains a loop and never returns COMPLETING_RETRY, can return
// COMPLETING_ALREADY -- if already completing or successfully made cancelling, added exception
// COMPLETING_WAITING_CHILDREN -- if started waiting for children, added exception
// TOO_LATE_TO_CANCEL -- too late to cancel, did not add exception
// final state -- when completed, for call to afterCompletion
private fun makeCancelling(cause: Any?): Any? {
    var causeExceptionCache: Throwable? = null // lazily init result of createCauseException(cause)
    loopOnState { state ->
        when (state) {
            is Finishing -> { // already finishing -- collect exceptions
                val notifyRootCause = synchronized(state) {
                    if (state.isSealed) return TOO_LATE_TO_CANCEL // already sealed -- cannot add exception nor mark cancelled
                    // add exception, do nothing is parent is cancelling child that is already being cancelled
                    val wasCancelling = state.isCancelling // will notify if was not cancelling
                    // Materialize missing exception if it is the first exception (otherwise -- don't)
                    if (cause != null || !wasCancelling) {
                        val causeException = causeExceptionCache ?: createCauseException(cause).also { causeExceptionCache = it }
                        state.addExceptionLocked(causeException)
                    }
                    // take cause for notification if was not in cancelling state before
                    state.rootCause.takeIf { !wasCancelling }
                }
                notifyRootCause?.let { notifyCancelling(state.list, it) }
                return COMPLETING_ALREADY
            }
            is Incomplete -> {
                // Not yet finishing -- try to make it cancelling
                val causeException = causeExceptionCache ?: createCauseException(cause).also { causeExceptionCache = it }
                if (state.isActive) {
                    // active state becomes cancelling
                    if (tryMakeCancelling(state, causeException)) return COMPLETING_ALREADY
                } else {
                    // non active state starts completing
                    val finalState = tryMakeCompleting(state, CompletedExceptionally(causeException))
                    when {
                        finalState === COMPLETING_ALREADY -> error("Cannot happen in $state")
                        finalState === COMPLETING_RETRY -> return@loopOnState
                        else -> return finalState
                    }
                }
            }
            else -> return TOO_LATE_TO_CANCEL // already complete
        }
    }
}
```

另外，它還會在 `finalizeFinishingState` 會在 `job` 被取消時創建 **CompleteExceptionally** 並指向 `finalState`：

```kotlin
// Finalizes Finishing -> Completed (terminal state) transition.
// ## IMPORTANT INVARIANT: Only one thread can be concurrently invoking this method.
// Returns final state that was created and updated to
private fun finalizeFinishingState(state: Finishing, proposedUpdate: Any?): Any? {
    /*
     * Note: proposed state can be Incomplete, e.g.
     * async {
     *     something.invokeOnCompletion {} // <- returns handle which implements Incomplete under the hood
     * }
     */
    assert { this.state === state } // consistency check -- it cannot change
    assert { !state.isSealed } // consistency check -- cannot be sealed yet
    assert { state.isCompleting } // consistency check -- must be marked as completing
    val proposedException = (proposedUpdate as? CompletedExceptionally)?.cause
    // Create the final exception and seal the state so that no more exceptions can be added
    var wasCancelling = false // KLUDGE: we cannot have contract for our own expect fun synchronized
    val finalException = synchronized(state) {
        wasCancelling = state.isCancelling
        val exceptions = state.sealLocked(proposedException)
        val finalCause = getFinalRootCause(state, exceptions)
        if (finalCause != null) addSuppressedExceptions(finalCause, exceptions)
        finalCause
    }
    // Create the final state object
    val finalState = when {
        // was not cancelled (no exception) -> use proposed update value
        finalException == null -> proposedUpdate
        // small optimization when we can used proposeUpdate object as is on cancellation
        finalException === proposedException -> proposedUpdate
        // cancelled job final state
        else -> CompletedExceptionally(finalException)
    }
    // Now handle the final exception
    if (finalException != null) {
        val handled = cancelParent(finalException) || handleJobException(finalException)
        if (handled) (finalState as CompletedExceptionally).makeHandled()
    }
    // Process state updates for the final state before the state of the Job is actually set to the final state
    // to avoid races where outside observer may see the job in the final state, yet exception is not handled yet.
    if (!wasCancelling) onCancelling(finalException)
    onCompletionInternal(finalState)
    // Then CAS to completed state -> it must succeed
    val casSuccess = _state.compareAndSet(state, finalState.boxIncomplete())
    assert { casSuccess }
    // And process all post-completion actions
    completeStateFinalization(state, finalState)
    return finalState
}
```


#### CompletableDeferredImpl

**CompletableDeferredImpl** 只會在 `completeExceptionally` 中創建 **CompleteExceptionally** 並傳入 `makeCompleting` 進行之後的等待：

```kotlin
/**
 * Concrete implementation of [CompletableDeferred].
 */
@Suppress("UNCHECKED_CAST")
private class CompletableDeferredImpl<T>(
    parent: Job?
) : JobSupport(true), CompletableDeferred<T>, SelectClause1<T> {
    init { initParentJob(parent) }
    override val onCancelComplete get() = true
    override fun getCompleted(): T = getCompletedInternal() as T
    override suspend fun await(): T = awaitInternal() as T
    override val onAwait: SelectClause1<T> get() = this
    override fun <R> registerSelectClause1(select: SelectInstance<R>, block: suspend (T) -> R) =
        registerSelectClause1Internal(select, block)

    override fun complete(value: T): Boolean =
        makeCompleting(value)
    override fun completeExceptionally(exception: Throwable): Boolean =
        makeCompleting(CompletedExceptionally(exception))
}
```


## CancelledContinuation

**CancelledContinuation** 是一個 **CompleteExceptionally**。 它的 `cause` 預設為 `CancellationException("Continuation $continuation was cancelled normally")`。

除此之外， **CancelledContinuation** 除了可以調用 `makeHandled` 還多了一個方法 `makeResumed`。 `makeResumed` 的行為就跟 `makeHandled` 一樣，都是針對 `atomic` 物件進行更新：

```kotlin
/**
 * A specific subclass of [CompletedExceptionally] for cancelled [AbstractContinuation].
 *
 * @param continuation the continuation that was cancelled.
 * @param cause the exceptional completion cause. If `cause` is null, then a [CancellationException]
 *        if created on first access to [exception] property.
 */
internal class CancelledContinuation(
    continuation: Continuation<*>,
    cause: Throwable?,
    handled: Boolean
) : CompletedExceptionally(cause ?: CancellationException("Continuation $continuation was cancelled normally"), handled) {
    private val _resumed = atomic(false)
    fun makeResumed(): Boolean = _resumed.compareAndSet(false, true)
}
```

# DisposableHandle
**DisposableHandle** 只是一個簡單的 SAM，它的主要行為就是通過 `dispose` 進行 GC：

```kotlin
public fun interface DisposableHandle {
    /**
     * Disposes the corresponding object, making it eligible for garbage collection.
     * Repeated invocation of this function has no effect.
     */
    public fun dispose()
}
```

實作或繼承 **DisposableHandle** 的型別包括：
- **JobNode**
  ```kotlin
  internal abstract class JobNode : CompletionHandlerBase(), DisposableHandle, Incomplete

  /*
      JobNode 的繼承者包括：
      CancelFutureOnCompletion, ChildCompletion, DisposeOnCompletion, SelectJoinOnCompletion,
      SelectAwaitOnCompletion, InvokeOnCompletion, ResumeAwaitOnCompletion, ResumeOnCompletion
      JobCancellingNode, InvokeOnCancelling, ChildContinuation, ChildHandleNode,
  */
  ```
- **ChildHandle**
- **DisposableFutureHandle**
- **AbstractSendChannel.SendSelect**
- **Mutex.LockWaiter**
- **EventLoop.DelayedTask**

## ChildHandle
```kotlin
/**
 * A handle that child keep onto its parent so that it is able to report its cancellation.
 *
 * @suppress **This is unstable API and it is subject to change.**
 */
@InternalCoroutinesApi
@Deprecated(level = DeprecationLevel.ERROR, message = "This is internal API and may be removed in the future releases")
public interface ChildHandle : DisposableHandle {
    @InternalCoroutinesApi
    public val parent: Job?
    @InternalCoroutinesApi
    public fun childCancelled(cause: Throwable): Boolean
}
```

**ChildHandle** 主要的功能是讓 child 通過 `childCancelled` 對 `parent` 進行取消。

繼承 **ChildHandle** 的類別包括：
- **ChildHandleNode**
- **NonDisposableHandle**

### ChildHandleNode
```kotlin
internal class ChildHandleNode(
    @JvmField val childJob: ChildJob
) : JobCancellingNode(), ChildHandle {
    override val parent: Job get() = job
    override fun invoke(cause: Throwable?) = childJob.parentCancelled(job)
    override fun childCancelled(cause: Throwable): Boolean = job.childCancelled(cause)
}
```

**ChildHandleNode** 中可以看到：
- 通過 `invoke`， `parent` 可以對 child 進行取消
- 通過 `childCancelled`， child 可以對 `parent` 進行取消


### NonDisposableHandle
```kotlin
@InternalCoroutinesApi
public object NonDisposableHandle : DisposableHandle, ChildHandle {

    override val parent: Job? get() = null
    override fun dispose() {}
    override fun childCancelled(cause: Throwable): Boolean = false
    override fun toString(): String = "NonDisposableHandle"
}
```


## DisposableFutureHandle

**DisposableFutureHandle** 主要是針對 **Future** 的 **DisposableHandle**：

```kotlin
private class DisposableFutureHandle(private val future: Future<*>) : DisposableHandle {
    override fun dispose() {
        future.cancel(false)
    }
    override fun toString(): String = "DisposableFutureHandle[$future]"
}
```

當中的 **Future** 則是一個負責非同步處理的介面：
```kotlin
// A Future represents the result of an asynchronous computation.

public interface Future<V> {
    boolean cancel(boolean mayInterruptIfRunning);
    boolean isCancelled();
    boolean isDone();
    V get() throws InterruptedException, ExecutionException;
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

**Future** 的行為包括：
- 取消 task
- 檢查是否被取消 \ 是否完成
- 等待 task 的完成

最主要實作 **Future** 的是 **FutureTask**， 一個既是 **Future** 同時也是 **Runnable** 的類別。


## AbstractSendChannel.SendSelect


## EventLoop.DelayedTask

```kotlin
internal abstract class DelayedTask(
    /**
     * This field can be only modified in [scheduleTask] before putting this DelayedTask
     * into heap to avoid overflow and corruption of heap data structure.
     */
    @JvmField var nanoTime: Long
) : Runnable, Comparable<DelayedTask>, DisposableHandle, ThreadSafeHeapNode
```

## SharedFlow.Emitter

```kotlin
private class Emitter(
    @JvmField val flow: SharedFlowImpl<*>,
    @JvmField var index: Long,
    @JvmField val value: Any?,
    @JvmField val cont: Continuation<Unit>
) : DisposableHandle {
    override fun dispose() = flow.cancelEmitter(this)
}
```

## JobNode
[]()


## NonDisposableHandle

## Mutex.LockWaiter
```kotlin
private abstract inner class LockWaiter(
    @JvmField val owner: Any?
) : LockFreeLinkedListNode(), DisposableHandle {
    private val isTaken = atomic(false)
    fun take(): Boolean = isTaken.compareAndSet(false, true)
    final override fun dispose() { remove() }
    abstract fun tryResumeLockWaiter(): Boolean
    abstract fun completeResumeLockWaiter()
}
```

## AbstractChannel.ReceiveSelect

```kotlin
private class ReceiveSelect<R, E>(
    @JvmField val channel: AbstractChannel<E>,
    @JvmField val select: SelectInstance<R>,
    @JvmField val block: suspend (Any?) -> R,
    @JvmField val receiveMode: Int
) : Receive<E>(), DisposableHandle
```

## AbstractChannel.SendSelect

```kotlin
private class SendSelect<E, R>(
    override val pollResult: E, // E | Closed - the result pollInternal returns when it rendezvous with this node
    @JvmField val channel: AbstractSendChannel<E>,
    @JvmField val select: SelectInstance<R>,
    @JvmField val block: suspend (SendChannel<E>) -> R
) : Send(), DisposableHandle
```

**Send** Represents sending waiter in the queue.

```kotlin
internal abstract class Send : LockFreeLinkedListNode() {
    abstract val pollResult: Any? // E | Closed - the result pollInternal returns when it rendezvous with this node
    // Returns: null - failure,
    //          RETRY_ATOMIC for retry (only when otherOp != null),
    //          RESUME_TOKEN on success (call completeResumeSend)
    // Must call otherOp?.finishPrepare() after deciding on result other than RETRY_ATOMIC
    abstract fun tryResumeSend(otherOp: PrepareOp?): Symbol?
    abstract fun completeResumeSend()
    abstract fun resumeSendClosed(closed: Closed<*>)
    open fun undeliveredElement() {}
}
```












# SelectInstance

```kotlin
/**
 * Internal representation of select instance. This instance is called _selected_ when
 * the clause to execute is already picked.
 *
 * @suppress **This is unstable API and it is subject to change.**
 */
@InternalCoroutinesApi // todo: sealed interface https://youtrack.jetbrains.com/issue/KT-22286
public interface SelectInstance<in R> {

    public val isSelected: Boolean
    public fun trySelect(): Boolean

    /**
     * Tries to select this instance. Returns:
     * * [RESUME_TOKEN] on success,
     * * [RETRY_ATOMIC] on deadlock (needs retry, it is only possible when [otherOp] is not `null`)
     * * `null` on failure to select (already selected).
     * [otherOp] is not null when trying to rendezvous with this select from inside of another select.
     * In this case, [PrepareOp.finishPrepare] must be called before deciding on any value other than [RETRY_ATOMIC].
     *
     * Note, that this method's actual return type is `Symbol?` but we cannot declare it as such, because this
     * member is public, but [Symbol] is internal. When [SelectInstance] becomes a `sealed interface`
     * (see KT-222860) we can declare this method as internal.
     */
    public fun trySelectOther(otherOp: PrepareOp?): Any?
    public fun performAtomicTrySelect(desc: AtomicDesc): Any?
    public val completion: Continuation<R>
    public fun resumeSelectWithException(exception: Throwable)

    public fun disposeOnSelect(handle: DisposableHandle)
}
```

**SelectInstance** 的唯一實作者是 **SelectBuilderImpl**，所以我們直接看這個類別吧。


## SelectBuilderImpl

**SelectBuilderImpl** 除了是一個 **SelectInstance** 還有其他角色：

```kotlin
@PublishedApi
internal class SelectBuilderImpl<in R>(
    private val uCont: Continuation<R> // unintercepted delegate continuation
) : LockFreeLinkedListHead(), SelectBuilder<R>,
    SelectInstance<R>, Continuation<R>, CoroutineStackFrame
```

- **LockFreeLinkedListHead**
  這是一個 **LockFreeLinkedListNode** 也是所謂的 **Node**。 它是通過 **Fetch-and-Add** 與 **Compare-and-Swap** 來讓 **LinkedList** 成為 **LockFree** 的。
  <br>
  ```kotlin
  private typealias Node = LockFreeLinkedListNode
  ```

  <br>有興趣的可以看看 [Lock-Free and Practical Doubly Linked List-Based Deques Using Single-Word Compare-and-Swap](https://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.140.4693&rep=rep1&type=pdf)。

  <br>而 **LockFreeLinkedListHead** 表示它永遠都會是 **LinkedList** 中的 **Head**。
- **SelectBuilder**
  這是一個負責 **SelectClause** 行為的介面
  <br>
  ```kotlin
  public interface SelectBuilder<in R> {
      public operator fun SelectClause0.invoke(block: suspend () -> R)
      public operator fun <Q> SelectClause1<Q>.invoke(block: suspend (Q) -> R)
      public operator fun <P, Q> SelectClause2<P, Q>.invoke(param: P, block: suspend (Q) -> R)
      public operator fun <P, Q> SelectClause2<P?, Q>.invoke(block: suspend (Q) -> R): Unit = invoke(null, block)
      @ExperimentalCoroutinesApi
      public fun onTimeout(timeMillis: Long, block: suspend () -> R)
  }
  ```

- **Continuation**
  這介面是 `suspend` 方法的
  <br>
  ```kotlin
  /**
   * Interface representing a continuation after a suspension point that returns a value of type `T`.
   */
  @SinceKotlin("1.3")
  public interface Continuation<in T> {
      /**
       * The context of the coroutine that corresponds to this continuation.
       */
      public val context: CoroutineContext

      /**
       * Resumes the execution of the corresponding coroutine passing a successful or failed [result] as the
       * return value of the last suspension point.
       */
      public fun resumeWith(result: Result<T>)
  }
  ```

- **CoroutineStackFrame**
  這介面只會在 debug 時使用，所以不需要太在意。
  <br>
  ```kotlin
  /**
   * Represents one frame in the coroutine call stack for debugger.
   * This interface is implemented by compiler-generated implementations of
   * [Continuation] interface.
   */
  @SinceKotlin("1.3")
  public interface CoroutineStackFrame {
      /**
       * Returns a reference to the stack frame of the caller of this frame,
       * that is a frame before this frame in coroutine call stack.
       * The result is `null` for the first frame of coroutine.
       */
      public val callerFrame: CoroutineStackFrame?

      /**
       * Returns stack trace element that correspond to this stack frame.
       * The result is `null` if the stack trace element is not available for this frame.
       * In this case, the debugger represents this stack frame using the
       * result of [toString] function.
       */
      public fun getStackTraceElement(): StackTraceElement?
  }
  ```
  <br>

  它是由 `uCont` 所實作：
  <br>
  ```kotlin
  override val callerFrame: CoroutineStackFrame?
          get() = uCont as? CoroutineStackFrame
  ```

這些介面的實作我們之後會說明。

### 狀態
我們在 **SelectBuilderImpl** 中可以看到有一個 `_state` 參數。 而他的預設值為 **NOT_SELECTED**：

```kotlin
// selection state is NOT_SELECTED initially and is replaced by idempotent marker (or null) when selected
private val _state = atomic<Any?>(NOT_SELECTED)
```

另外，它裡面還有一個 `_result`：
```kotlin
// this is basically our own SafeContinuation
private val _result = atomic<Any?>(UNDECIDED)
```


這裡的 **NOT_SELECTED** 與 **UNDECIDED** 其實都是一個 **Symbol**：

```kotlin
@SharedImmutable
internal val NOT_SELECTED: Any = Symbol("NOT_SELECTED")
@SharedImmutable
private val UNDECIDED: Any = Symbol("UNDECIDED")
```

除此之外，我們還有以下 **Symbol**：

```kotlin
@SharedImmutable
internal val ALREADY_SELECTED: Any = Symbol("ALREADY_SELECTED")
@SharedImmutable
private val RESUMED: Any = Symbol("RESUMED")
```

但這並非全部的 `_state` 或 `_result`。 我們還有：

```kotlin
// It is implemented as property with getter to avoid ProGuard <clinit> problem with multifile IntrinsicsKt class
@SinceKotlin("1.3")
public val COROUTINE_SUSPENDED: Any get() = CoroutineSingletons.COROUTINE_SUSPENDED
```

**COROUTINE_SUSPENDED** 其實是 **CoroutineSingletons** 中的其中一個：

```kotlin
// Using enum here ensures two important properties:
//  1. It makes SafeContinuation serializable with all kinds of serialization frameworks (since all of them natively support enums)
//  2. It improves debugging experience, since you clearly see toString() value of those objects and what package they come from
@SinceKotlin("1.3")
@PublishedApi // This class is Published API via serialized representation of SafeContinuation, don't rename/move
internal enum class CoroutineSingletons { COROUTINE_SUSPENDED, UNDECIDED, RESUMED }
```

`_state` 與 `_result` 的不同狀態可以從以下圖表看出來：

```
/* Result state machine

  +-----------+   getResult   +---------------------+   resume   +---------+
  | UNDECIDED | ------------> | COROUTINE_SUSPENDED | ---------> | RESUMED |
  +-----------+               +---------------------+            +---------+
        |
        | resume
        V
  +------------+  getResult
  | value/Fail | -----------+
  +------------+            |
        ^                   |
        |                   |
        +-------------------+
*/
```

上面圖表中所使用的方法其實並非都是修改 `_state` 的，而是包括 `_result`。

所以我們來看看這些方法如何影響 `_state` 與 `_result` 吧。

#### 圖表中對應的方法

##### _result

`_result` 只會在 **SelectBuilderImpl** 中被更改：

```kotlin
/*
  getResult 中會更新 _result
  - 從 UNDECIDED 到 COROUTINE_SUSPENDED

  並且會確保 result 並非 RESUMED 或 CompleteExceptionally

  - 最後回傳的可能是 COROUTINE_SUSPENDED 或 data

*/
@PublishedApi
internal fun getResult(): Any? {
    if (!isSelected) initCancellability()
    var result = _result.value // atomic read
    if (result === UNDECIDED) {
        if (_result.compareAndSet(UNDECIDED, COROUTINE_SUSPENDED)) return COROUTINE_SUSPENDED
        result = _result.value // reread volatile var
    }
    when {
        result === RESUMED -> throw IllegalStateException("Already resumed")
        result is CompletedExceptionally -> throw result.cause
        else -> return result // either COROUTINE_SUSPENDED or data
    }
}

/*
  doResume 會將 _result
  - 從 UNDECIDED 更新成 update
  - 從 COROUTINE_SUSPENDED 更新成 RESUMED
*/
private inline fun doResume(value: () -> Any?, block: () -> Unit) {
    assert { isSelected } // "Must be selected first"
    _result.loop { result ->
        when {
            result === UNDECIDED -> {
                val update = value()
                if (_result.compareAndSet(UNDECIDED, update)) return
            }
            result === COROUTINE_SUSPENDED -> if (_result.compareAndSet(COROUTINE_SUSPENDED, RESUMED)) {
                block()
                return
            }
            else -> throw IllegalStateException("Already resumed")
        }
    }
}
```

另外，我們可以看到 `doResume` 中 `_result` 所取得的 `update` 會以 `value` 帶入。 如果我們進行追蹤，我們可以得知 `value` 可以從以下傳入：
- **SelectBuilderImpl** 的 `resumeWith`
  這會傳入 **Result**
- **SelectBuilderImpl** 的 `resumeSelectWithException`
  這會傳入 **Throwable**

```kotlin
// SelectBuilderImpl
// Resumes in direct mode, without going through dispatcher. Should be called in the same context.
override fun resumeWith(result: Result<R>) {
    doResume({ result.toState() }) {
        if (result.isFailure) {
            uCont.resumeWithStackTrace(result.exceptionOrNull()!!)
        } else {
            uCont.resumeWith(result)
        }
    }
}

// Resumes in dispatched way so that it can be called from an arbitrary context
override fun resumeSelectWithException(exception: Throwable) {
    doResume({ CompletedExceptionally(recoverStackTrace(exception, uCont)) }) {
        uCont.intercepted().resumeWith(Result.failure(exception))
    }
}
```




**_state**
```kotlin
/*
  trySelectOther
*/
// it is just like plain trySelect, but support idempotent start
// Returns RESUME_TOKEN | RETRY_ATOMIC | null (when already selected)
override fun trySelectOther(otherOp: PrepareOp?): Any? {
    _state.loop { state -> // lock-free loop on state
        when {
            // Found initial state (not selected yet) -- try to make it selected
            state === NOT_SELECTED -> {
                if (otherOp == null) {
                    // regular trySelect -- just mark as select
                    if (!_state.compareAndSet(NOT_SELECTED, null)) return@loop
                } else {
                    // Rendezvous with another select instance -- install PairSelectOp
                    val pairSelectOp = PairSelectOp(otherOp)
                    if (!_state.compareAndSet(NOT_SELECTED, pairSelectOp)) return@loop
                    val decision = pairSelectOp.perform(this)
                    if (decision !== null) return decision
                }
                doAfterSelect()
                return RESUME_TOKEN
            }
            state is OpDescriptor -> { // state is either AtomicSelectOp or PairSelectOp
                // Found descriptor of ongoing operation while working in the context of other select operation
                if (otherOp != null) {
                    val otherAtomicOp = otherOp.atomicOp
                    when {
                        // It is the same select instance
                        otherAtomicOp is AtomicSelectOp && otherAtomicOp.impl === this -> {
                            /*
                             * We cannot do state.perform(this) here and "help" it since it is the same
                             * select and we'll get StackOverflowError.
                             * See https://github.com/Kotlin/kotlinx.coroutines/issues/1411
                             * We cannot support this because select { ... } is an expression and its clauses
                             * have a result that shall be returned from the select.
                             */
                            error("Cannot use matching select clauses on the same object")
                        }
                        // The other select (that is trying to proceed) had started earlier
                        otherAtomicOp.isEarlierThan(state) -> {
                            /**
                             * Abort to prevent deadlock by returning a failure to it.
                             * See https://github.com/Kotlin/kotlinx.coroutines/issues/504
                             * The other select operation will receive a failure and will restart itself with a
                             * larger sequence number. This guarantees obstruction-freedom of this algorithm.
                             */
                            return RETRY_ATOMIC
                        }
                    }
                }
                // Otherwise (not a special descriptor)
                state.perform(this) // help it
            }
            // otherwise -- already selected
            otherOp == null -> return null // already selected
            state === otherOp.desc -> return RESUME_TOKEN // was selected with this marker
            else -> return null // selected with different marker
        }
    }
}
```






# OpDescriptor
## PairSelectOp
```kotlin
// The very last step of rendezvous between two select operations
private class PairSelectOp(
    @JvmField val otherOp: PrepareOp
) : OpDescriptor() {
    override fun perform(affected: Any?): Any? {
        val impl = affected as SelectBuilderImpl<*>
        // here we are definitely not going to RETRY_ATOMIC, so
        // we must finish preparation of another operation before attempting to reach decision to select
        otherOp.finishPrepare()
        val decision = otherOp.atomicOp.decide(null) // try decide for success of operation
        val update: Any = if (decision == null) otherOp.desc else NOT_SELECTED
        impl._state.compareAndSet(this, update)
        return decision
    }

    override val atomicOp: AtomicOp<*>
        get() = otherOp.atomicOp
}
```

## AtomicOp
### AtomicSelectOp
```kotlin
private fun prepareSelectOp(): Any? {
    impl._state.loop { state ->
        when {
            state === this -> return null // already in progress
            state is OpDescriptor -> state.perform(impl) // help
            state === NOT_SELECTED -> {
                if (impl._state.compareAndSet(NOT_SELECTED, this))
                    return null // success
            }
            else -> return ALREADY_SELECTED
        }
    }
}

// reverts the change done by prepareSelectedOp
private fun undoPrepare() {
    impl._state.compareAndSet(this, NOT_SELECTED)
}

private fun completeSelect(failure: Any?) {
    val selectSuccess = failure == null
    val update = if (selectSuccess) null else NOT_SELECTED
    if (impl._state.compareAndSet(this, update)) {
        if (selectSuccess)
            impl.doAfterSelect()
    }
}
```

## Deferred

**Deferred** 是另一個 **Job**。 它新增了 **等待** 以及 **要求完成** 的方法：

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

繼承 **Deferred** 的目前只有 2 種類別：
- **CompletableDeferred**
- **DeferredCoroutine**


### CompletableDeferred
#### CompletableDeferredImpl

### DeferredCoroutine

```kotlin
@Suppress("UNCHECKED_CAST")
private open class DeferredCoroutine<T>(
    parentContext: CoroutineContext,
    active: Boolean
) : AbstractCoroutine<T>(parentContext, true, active = active), Deferred<T>, SelectClause1<T> {
    override fun getCompleted(): T = getCompletedInternal() as T
    override suspend fun await(): T = awaitInternal() as T
    override val onAwait: SelectClause1<T> get() = this
    override fun <R> registerSelectClause1(select: SelectInstance<R>, block: suspend (T) -> R) =
        registerSelectClause1Internal(select, block)
}
```

#### LazyDeferredCoroutine

```kotlin
private class LazyDeferredCoroutine<T>(
    parentContext: CoroutineContext,
    block: suspend CoroutineScope.() -> T
) : DeferredCoroutine<T>(parentContext, active = false) {
    private val continuation = block.createCoroutineUnintercepted(this, this)

    override fun onStart() {
        continuation.startCoroutineCancellable(this)
    }
}
```





# OpDescriptor

><br>
>
>The most abstract operation that can be in process. Other threads observing an instance of this class in the fields of their object shall invoke perform to help.
><br>

```kotlin
public abstract class OpDescriptor {
    abstract fun perform(affected: Any?): Any?
    abstract val atomicOp: AtomicOp<*>?
    override fun toString(): String = "$classSimpleName@$hexAddress" // debug
    fun isEarlierThan(that: OpDescriptor): Boolean {
        val thisOp = atomicOp ?: return false
        val thatOp = that.atomicOp ?: return false
        return thisOp.opSequence < thatOp.opSequence
    }
}
```


## AtomicOp



```kotlin
@InternalCoroutinesApi
public abstract class AtomicOp<in T> : OpDescriptor() {
    private val _consensus = atomic<Any?>(NO_DECISION)

    val consensus: Any? get() = _consensus.value
    val isDecided: Boolean get() = _consensus.value !== NO_DECISION
    open val opSequence: Long get() = 0L

    abstract fun prepare(affected: T): Any? // `null` if Ok, or failure reason
    abstract fun complete(affected: T, failure: Any?) // failure != null if failed to prepare op

    fun decide(decision: Any?): Any? {
        assert { decision !== NO_DECISION }
        val current = _consensus.value
        if (current !== NO_DECISION) return current
        if (_consensus.compareAndSet(NO_DECISION, decision)) return decision
        return _consensus.value
    }

    override val atomicOp: AtomicOp<*> get() = this

    // returns `null` on success
    @Suppress("UNCHECKED_CAST")
    final override fun perform(affected: Any?): Any? {
        // make decision on status
        var decision = this._consensus.value
        if (decision === NO_DECISION) {
            decision = decide(prepare(affected as T))
        }
        // complete operation
        complete(affected as T, decision)
        return decision
    }
}
```

### AtomicSelectOp
```kotlin
private class AtomicSelectOp(
        @JvmField val impl: SelectBuilderImpl<*>,
        @JvmField val desc: AtomicDesc
    ) : AtomicOp<Any?>()
```


## LockFreeLinkedListNode.PairSelectOp
```kotlin
// The very last step of rendezvous between two select operations
private class PairSelectOp(
    @JvmField val otherOp: PrepareOp
) : OpDescriptor() {
    override fun perform(affected: Any?): Any? {
        val impl = affected as SelectBuilderImpl<*>
        // here we are definitely not going to RETRY_ATOMIC, so
        // we must finish preparation of another operation before attempting to reach decision to select
        otherOp.finishPrepare()
        val decision = otherOp.atomicOp.decide(null) // try decide for success of operation
        val update: Any = if (decision == null) otherOp.desc else NOT_SELECTED
        impl._state.compareAndSet(this, update)
        return decision
    }

    override val atomicOp: AtomicOp<*>
        get() = otherOp.atomicOp
}
```

## LockFreeLinkedListNode.PrepareOp
```kotlin
// This is Harris's RDCSS (Restricted Double-Compare Single Swap) operation
// It inserts "op" descriptor of when "op" status is still undecided (rolls back otherwise)
public class PrepareOp(
    @JvmField val affected: Node,
    @JvmField val next: Node,
    @JvmField val desc: AbstractAtomicDesc
) : OpDescriptor() {
    override val atomicOp: AtomicOp<*> get() = desc.atomicOp
    public fun finishPrepare(): Unit = desc.finishPrepare(this)
    override fun toString(): String = "PrepareOp(op=$atomicOp)"
}
```

其中的 `perform` 實作為：

```kotlin
// Returns REMOVE_PREPARED or null (it makes decision on any failure)
override fun perform(affected: Any?): Any? {
    assert { affected === this.affected }
    affected as Node // type assertion
    val decision = desc.onPrepare(this)
    if (decision === REMOVE_PREPARED) {
        // remove element on failure -- do not mark as decided, will try another one
        val next = this.next
        val removed = next.removed()
        if (affected._next.compareAndSet(this, removed)) {
            // The element was actually removed
            desc.onRemoved(affected)
            // Complete removal operation here. It bails out if next node is also removed and it becomes
            // responsibility of the next's removes to call correctPrev which would help fix all the links.
            next.correctPrev(null)
        }
        return REMOVE_PREPARED
    }
    // We need to ensure progress even if it operation result consensus was already decided
    val consensus = if (decision != null) {
        // some other logic failure, including RETRY_ATOMIC -- reach consensus on decision fail reason ASAP
        atomicOp.decide(decision)
    } else {
        atomicOp.consensus // consult with current decision status like in Harris DCSS
    }
    val update: Any = when {
        consensus === NO_DECISION -> atomicOp // desc.onPrepare returned null -> start doing atomic op
        consensus == null -> desc.updatedNext(affected, next) // move forward if consensus on success
        else -> next // roll back if consensus if failure
    }
    affected._next.compareAndSet(this, update)
    return null
}
```


## MutexImpl.TryLockDesc.PrepareOp
```kotlin
// This is Harris's RDCSS (Restricted Double-Compare Single Swap) operation
private inner class PrepareOp(override val atomicOp: AtomicOp<*>) : OpDescriptor() {
    override fun perform(affected: Any?): Any? {
        val update: Any = if (atomicOp.isDecided) EMPTY_UNLOCKED else atomicOp // restore if was already decided
        (affected as MutexImpl)._state.compareAndSet(this, update)
        return null // ok
    }
}
```


# AtomicDesc
>A part of multi-step atomic operation AtomicOp.

```kotlin
public abstract class AtomicDesc {
    lateinit var atomicOp: AtomicOp<*> // the reference to parent atomicOp, init when AtomicOp is created
    abstract fun prepare(op: AtomicOp<*>): Any? // returns `null` if prepared successfully
    abstract fun complete(op: AtomicOp<*>, failure: Any?) // decision == null if success
}
```





## AbstractAtomicDesc

```kotlin
public abstract class AbstractAtomicDesc : AtomicDesc() {
    protected abstract val affectedNode: Node?
    protected abstract val originalNext: Node?

    protected abstract fun finishOnSuccess(affected: Node, next: Node)
    public abstract fun updatedNext(affected: Node, next: Node): Any
    public abstract fun finishPrepare(prepareOp: PrepareOp)
}
```










<br><br><br><br><br><br><br>
