---
layout: post
title:  Kotlinx.Coroutine - 從 suspend 來談談 Continuation
date:   2022-08-08 16:15:07 +0800
categories: [coroutine, android, intermediate]
---


# 何謂 suspend function ?
我們在寫 Kotlin 時，當我們需要執行一些可能會需要較長執行時間的都會將方法設為 `suspend`，像是： Network API、 Database API 和 複雜計算 等等。

<u><b>但 `suspend` 這個 keyword 有什麼用呢？</b></u>

我們來看個範例吧：
```kotlin
suspend fun getTasks(): Result<List<Task>>
```

當我們將它變成 byteCode，我們會看到以下：

```java
@Nullable
Object getTasks(@NotNull Continuation var1);
```

原來 `suspend` 會將 **Continuation** 當作是參數放入方法中。

當我們定義這方法的行為後：
```kotlin
override suspend fun getTasks(): Result<List<Task>> = withContext(ioDispatcher) {
    return@withContext try {
        Success(tasksDao.getTasks())
    } catch (e: Exception) {
        Error(e)
    }
}
```

我們再看一次 byteCode。 以下是 `getTasks` 方法的 byteCode ：

```java
// access flags 0x1
  // signature (Lkotlin/coroutines/Continuation<-Lcom/example/android/architecture/blueprints/todoapp/data/Result<+Ljava/util/List<Lcom/example/android/architecture/blueprints/todoapp/data/Task;>;>;>;)Ljava/lang/Object;
  // declaration:  getTasks(kotlin.coroutines.Continuation<? super com.example.android.architecture.blueprints.todoapp.data.Result<? extends java.util.List<com.example.android.architecture.blueprints.todoapp.data.Task>>>)
  public getTasks(Lkotlin/coroutines/Continuation;)Ljava/lang/Object;
  @Lorg/jetbrains/annotations/Nullable;() // invisible
    // annotable parameter count: 1 (visible)
    // annotable parameter count: 1 (invisible)
    @Lorg/jetbrains/annotations/NotNull;() // invisible, parameter 0
   L0
    LINENUMBER 58 L0
    /*
        1. 將 this <TasksDataSource> 放入 stack
           ..., this
    */
    ALOAD 0

    /*
        2. 從 stack pop 出 this <TasksDataSource> 並取得 ioDispatcher
           再將它放入 stack 中
           ..., ioDispatcher
    */
    GETFIELD com/example/android/architecture/blueprints/todoapp/data/source/local/TasksLocalDataSource.ioDispatcher : Lkotlinx/coroutines/CoroutineDispatcher;

    /*
        3. 將 stack 中的 ioDispatcher 定義其型別為 CoroutineContext
           ..., CoroutineContext
    */
    CHECKCAST kotlin/coroutines/CoroutineContext

    /*
        4. 將尚未 init 的 getTasks$2 放入 stack
           ..., CoroutineContext, getTasks$2 (uninit)
    */
    NEW com/example/android/architecture/blueprints/todoapp/data/source/local/TasksLocalDataSource$getTasks$2
    /*
        5. 複製 getTasks$2 (uninitialized)
           ..., CoroutineContext, getTasks$2 (uninit), getTasks$2 (uninit)
    */
    DUP

    /*
        6. 取得 aload_0 this 並放入 stack 中
           ..., CoroutineContext, getTasks$2 (uninit), getTasks$2 (uninit), this
    */
    ALOAD 0

    /*
        7. 放 null 入 stack
           ..., CoroutineContext, getTasks$2 (uninit), getTasks$2 (uninit), this, null
    */
    ACONST_NULL

    /*
        8. 將 stack 中的值按順序放入 <init> 中。
           所以我們會得到：
           getTasks$2.<init>(this, null)

           要注意，這些參數其實會用在 SuspendLambda 的建構子，之後會看到。

           而 stack 則會變成
           ..., CoroutineContext, getTasks$2 (initialized)
    */
    INVOKESPECIAL com/example/android/architecture/blueprints/todoapp/data/source/local/TasksLocalDataSource$getTasks$2.<init> (Lcom/example/android/architecture/blueprints/todoapp/data/source/local/TasksLocalDataSource;Lkotlin/coroutines/Continuation;)V

    /*
        9. getTasks$2 (initialized) 設為 Function2
           ..., CoroutineContext, Function2
    */
    CHECKCAST kotlin/jvm/functions/Function2

    /*
        10. 將 getTasks(Continuation) 中的參數 Continuation 載入 stack
        ..., CoroutineContext, Function2, Continuation
    */
    ALOAD 1

    /*
        11. 調用靜態方法 Builders.withContext

        Builders.common.kt
        public suspend fun <T> withContext(
          context: CoroutineContext,
          block: suspend CoroutineScope.() -> T
        ): T

        並將 CoroutineContext, Function2, Continuation pop 出來並使用在方法中。
        ...
    */
    INVOKESTATIC kotlinx/coroutines/BuildersKt.withContext (Lkotlin/coroutines/CoroutineContext;Lkotlin/jvm/functions/Function2;Lkotlin/coroutines/Continuation;)Ljava/lang/Object;
   L1
    LINENUMBER 64 L1
    ARETURN
   L2
    LOCALVARIABLE this Lcom/example/android/architecture/blueprints/todoapp/data/source/local/TasksLocalDataSource; L0 L2 0
    LOCALVARIABLE $completion Lkotlin/coroutines/Continuation; L0 L2 1
    MAXSTACK = 5
    MAXLOCALS = 2
```

請注意：上方的 byteCode 中在第一次調用 **INVOKESPECIAL** 時，他是調用 `com/example/android/architecture/blueprints/todoapp/data/source/local/TasksLocalDataSource$getTasks$2`。

而這個類別可以在後面的 byteCode 看到，原來它被設為一個 **INNERCLASS** ：

```java
final static INNERCLASS com/example/android/architecture/blueprints/todoapp/data/source/local/TasksLocalDataSource$getTasks$2 null null
```

以下就是被設定的 **getTasks$2** 類別：

```java
// ================com/example/android/architecture/blueprints/todoapp/data/source/local/TasksLocalDataSource$getTasks$2.class =================
// class version 52.0 (52)
// access flags 0x30
// signature Lkotlin/coroutines/jvm/internal/SuspendLambda;Lkotlin/jvm/functions/Function2<Lkotlinx/coroutines/CoroutineScope;Lkotlin/coroutines/Continuation<-Lcom/example/android/architecture/blueprints/todoapp/data/Result<+Ljava/util/List<+Lcom/example/android/architecture/blueprints/todoapp/data/Task;>;>;>;Ljava/lang/Object;>;
// declaration: com/example/android/architecture/blueprints/todoapp/data/source/local/TasksLocalDataSource$getTasks$2 extends kotlin.coroutines.jvm.internal.SuspendLambda implements kotlin.jvm.functions.Function2<kotlinx.coroutines.CoroutineScope, kotlin.coroutines.Continuation<? super com.example.android.architecture.blueprints.todoapp.data.Result<? extends java.util.List<? extends com.example.android.architecture.blueprints.todoapp.data.Task>>>, java.lang.Object>

/*
  1. 方法被定義為一個 SuspendLambda 的類別，並實作 Function2
*/
final class com/example/android/architecture/blueprints/todoapp/data/source/local/TasksLocalDataSource$getTasks$2 extends kotlin/coroutines/jvm/internal/SuspendLambda implements kotlin/jvm/functions/Function2 {


  // access flags 0x11
  public final invokeSuspend(Ljava/lang/Object;)Ljava/lang/Object;
  @Lorg/jetbrains/annotations/Nullable;() // invisible
    // annotable parameter count: 1 (visible)
    // annotable parameter count: 1 (invisible)
    @Lorg/jetbrains/annotations/NotNull;() // invisible, parameter 0
    TRYCATCHBLOCK L0 L1 L2 java/lang/Exception
    TRYCATCHBLOCK L3 L4 L2 java/lang/Exception

    /*
        2. 通過 static 方法取得 COROUTINE_SUSPENDED
           這是位於 Intrinsics.kt 中的變數：
           public val COROUTINE_SUSPENDED: Any get() = CoroutineSingletons.COROUTINE_SUSPENDED
           由於這方法是用 java 寫的，所以算是 non-native 因此 JVM 會再創建一個 Frame 並將結果放置此 Frame 的 stack 中。
    */
    INVOKESTATIC kotlin/coroutines/intrinsics/IntrinsicsKt.getCOROUTINE_SUSPENDED ()Ljava/lang/Object;
   L5
    LINENUMBER 58 L5
    /*
        3. 將剛剛的 COROUTINE_SUSPENDED pop 出來並存放至 astore_6 之中
    */
    ASTORE 6
    /*
        4. 取得 aload_0
    */
    ALOAD 0
    GETFIELD com/example/android/architecture/blueprints/todoapp/data/source/local/TasksLocalDataSource$getTasks$2.label : I
    TABLESWITCH
      0: L6
      1: L3
      default: L7
   L6
    ALOAD 1
    INVOKESTATIC kotlin/ResultKt.throwOnFailure (Ljava/lang/Object;)V
   L8
    LINENUMBER 59 L8
   L0
    NOP
   L9
    LINENUMBER 60 L9
    ALOAD 0
    GETFIELD com/example/android/architecture/blueprints/todoapp/data/source/local/TasksLocalDataSource$getTasks$2.this$0 : Lcom/example/android/architecture/blueprints/todoapp/data/source/local/TasksLocalDataSource;
    INVOKESTATIC com/example/android/architecture/blueprints/todoapp/data/source/local/TasksLocalDataSource.access$getTasksDao$p (Lcom/example/android/architecture/blueprints/todoapp/data/source/local/TasksLocalDataSource;)Lcom/example/android/architecture/blueprints/todoapp/data/source/local/TasksDao;
    ALOAD 0
    ALOAD 0
    ICONST_1
    PUTFIELD com/example/android/architecture/blueprints/todoapp/data/source/local/TasksLocalDataSource$getTasks$2.label : I
    INVOKEINTERFACE com/example/android/architecture/blueprints/todoapp/data/source/local/TasksDao.getTasks (Lkotlin/coroutines/Continuation;)Ljava/lang/Object; (itf)
   L1
    DUP
    ALOAD 6
    IF_ACMPNE L10
   L11
    LINENUMBER 58 L11
    ALOAD 6
    ARETURN
   L3
    NOP
    ALOAD 1
    INVOKESTATIC kotlin/ResultKt.throwOnFailure (Ljava/lang/Object;)V
    ALOAD 1
   L10
    LINENUMBER 60 L10
    ASTORE 4
    ALOAD 4
    ASTORE 5
    NEW com/example/android/architecture/blueprints/todoapp/data/Result$Success
    DUP
    ALOAD 5
    INVOKESPECIAL com/example/android/architecture/blueprints/todoapp/data/Result$Success.<init> (Ljava/lang/Object;)V
    CHECKCAST com/example/android/architecture/blueprints/todoapp/data/Result
   L12
    ASTORE 2
   L4
    GOTO L13
   L2
    LINENUMBER 61 L2
    ASTORE 3
   L14
    LINENUMBER 62 L14
    NEW com/example/android/architecture/blueprints/todoapp/data/Result$Error
    DUP
    ALOAD 3
    INVOKESPECIAL com/example/android/architecture/blueprints/todoapp/data/Result$Error.<init> (Ljava/lang/Exception;)V
    CHECKCAST com/example/android/architecture/blueprints/todoapp/data/Result
   L15
    ASTORE 2
   L16
    LINENUMBER 59 L16
   L13
    ALOAD 2
    ARETURN
   L7
    LINENUMBER 58 L7
    NEW java/lang/IllegalStateException
    DUP
    LDC "call to 'resume' before 'invoke' with coroutine"
    INVOKESPECIAL java/lang/IllegalStateException.<init> (Ljava/lang/String;)V
    ATHROW
    LOCALVARIABLE e Ljava/lang/Exception; L14 L16 3
    LOCALVARIABLE this Lcom/example/android/architecture/blueprints/todoapp/data/source/local/TasksLocalDataSource$getTasks$2; L8 L7 0
    LOCALVARIABLE $result Ljava/lang/Object; L8 L7 1
    MAXSTACK = 4
    MAXLOCALS = 9

  @Lkotlin/coroutines/jvm/internal/DebugMetadata;(f="TasksLocalDataSource.kt", l={60}, i={}, s={}, n={}, m="invokeSuspend", c="com.example.android.architecture.blueprints.todoapp.data.source.local.TasksLocalDataSource$getTasks$2")

  // access flags 0x0
  /* ============ getTasks$2.<init> ================ */
  <init>(Lcom/example/android/architecture/blueprints/todoapp/data/source/local/TasksLocalDataSource;Lkotlin/coroutines/Continuation;)V

    /*
        1. 將 this <getTasks$2> 放入 stack
           ..., this
    */
    ALOAD 0

    /*
        2. 將第一個參數 <TasksLocalDataSource> 放入 stack
           ..., this, TasksLocalDataSource
    */
    ALOAD 1

    /*
        3. 將 this 與 TasksLocalDataSource pop 出來，並將後者設為 this 的參數。
           ...
    */
    PUTFIELD com/example/android/architecture/blueprints/todoapp/data/source/local/TasksLocalDataSource$getTasks$2.this$0 : Lcom/example/android/architecture/blueprints/todoapp/data/source/local/TasksLocalDataSource;

    /*
        4. 再次將載入 this
           ..., this
    */
    ALOAD 0

    /*
        5. 將 iconst_2 === 2 放入 stack 中
           ..., this, 2

        iconst： Push the int constant onto the operand stack.
    */
    ICONST_2

    /*
        6. 將第二個參數 Continuation...null 載入 stack
           ..., this, 2, null
    */
    ALOAD 2

    /*
        7. 這會調用 getTasks$2 的父類別的 <init>

        ContinuationImpl.kt

        @SinceKotlin("1.3")
        // Suspension lambdas inherit from this class
        internal abstract class SuspendLambda(
            public override val arity: Int,
            completion: Continuation<Any?>?
        ) : ContinuationImpl(completion), FunctionBase<Any?>, SuspendFunction

        所以這裡就會將 this pop 出來 super.init(2, null)
    */
    INVOKESPECIAL kotlin/coroutines/jvm/internal/SuspendLambda.<init> (ILkotlin/coroutines/Continuation;)V
    /*
        8. return void
    */
    RETURN
    MAXSTACK = 3
    MAXLOCALS = 3

  // access flags 0x0
  I label

  // access flags 0x1010
  final synthetic Lcom/example/android/architecture/blueprints/todoapp/data/source/local/TasksLocalDataSource; this$0

  // access flags 0x11
  // signature (Ljava/lang/Object;Lkotlin/coroutines/Continuation<*>;)Lkotlin/coroutines/Continuation<Lkotlin/Unit;>;
  // declaration: kotlin.coroutines.Continuation<kotlin.Unit> create(java.lang.Object, kotlin.coroutines.Continuation<?>)
  public final create(Ljava/lang/Object;Lkotlin/coroutines/Continuation;)Lkotlin/coroutines/Continuation;
  @Lorg/jetbrains/annotations/NotNull;() // invisible
    // annotable parameter count: 2 (visible)
    // annotable parameter count: 2 (invisible)
    @Lorg/jetbrains/annotations/Nullable;() // invisible, parameter 0
    @Lorg/jetbrains/annotations/NotNull;() // invisible, parameter 1
   L0
    ALOAD 2
    LDC "completion"
    INVOKESTATIC kotlin/jvm/internal/Intrinsics.checkNotNullParameter (Ljava/lang/Object;Ljava/lang/String;)V
    NEW com/example/android/architecture/blueprints/todoapp/data/source/local/TasksLocalDataSource$getTasks$2
    DUP
    ALOAD 0
    GETFIELD com/example/android/architecture/blueprints/todoapp/data/source/local/TasksLocalDataSource$getTasks$2.this$0 : Lcom/example/android/architecture/blueprints/todoapp/data/source/local/TasksLocalDataSource;
    ALOAD 2
    INVOKESPECIAL com/example/android/architecture/blueprints/todoapp/data/source/local/TasksLocalDataSource$getTasks$2.<init> (Lcom/example/android/architecture/blueprints/todoapp/data/source/local/TasksLocalDataSource;Lkotlin/coroutines/Continuation;)V
    ASTORE 3
    ALOAD 3
    ARETURN
   L1
    LOCALVARIABLE this Lkotlin/coroutines/jvm/internal/BaseContinuationImpl; L0 L1 0
    LOCALVARIABLE value Ljava/lang/Object; L0 L1 1
    LOCALVARIABLE completion Lkotlin/coroutines/Continuation; L0 L1 2
    MAXSTACK = 4
    MAXLOCALS = 4

  // access flags 0x11
  public final invoke(Ljava/lang/Object;Ljava/lang/Object;)Ljava/lang/Object;
    ALOAD 0
    ALOAD 1
    ALOAD 2
    CHECKCAST kotlin/coroutines/Continuation
    INVOKEVIRTUAL com/example/android/architecture/blueprints/todoapp/data/source/local/TasksLocalDataSource$getTasks$2.create (Ljava/lang/Object;Lkotlin/coroutines/Continuation;)Lkotlin/coroutines/Continuation;
    CHECKCAST com/example/android/architecture/blueprints/todoapp/data/source/local/TasksLocalDataSource$getTasks$2
    GETSTATIC kotlin/Unit.INSTANCE : Lkotlin/Unit;
    INVOKEVIRTUAL com/example/android/architecture/blueprints/todoapp/data/source/local/TasksLocalDataSource$getTasks$2.invokeSuspend (Ljava/lang/Object;)Ljava/lang/Object;
    ARETURN
    MAXSTACK = 3
    MAXLOCALS = 3

  @Lkotlin/Metadata;(mv={1, 9, 0}, k=3, d1={"\u0000\u0016\n\u0000\n\u0002\u0018\u0002\n\u0002\u0010 \n\u0002\u0018\u0002\n\u0002\u0018\u0002\n\u0002\u0008\u0002\u0010\u0000\u001a\u000e\u0012\n\u0012\u0008\u0012\u0004\u0012\u00020\u00030\u00020\u0001*\u00020\u0004H\u008a@\u00a2\u0006\u0004\u0008\u0005\u0010\u0006"}, d2={"<anonymous>", "Lcom/example/android/architecture/blueprints/todoapp/data/Result;", "", "Lcom/example/android/architecture/blueprints/todoapp/data/Task;", "Lkotlinx/coroutines/CoroutineScope;", "invoke", "(Ljava/lang/Object;Ljava/lang/Object;)Ljava/lang/Object;"})
  OUTERCLASS com/example/android/architecture/blueprints/todoapp/data/source/local/TasksLocalDataSource getTasks (Lkotlin/coroutines/Continuation;)Ljava/lang/Object;
  // access flags 0x18
  final static INNERCLASS com/example/android/architecture/blueprints/todoapp/data/source/local/TasksLocalDataSource$getTasks$2 null null
  // compiled from: TasksLocalDataSource.kt
}
```

也就是說， `suspend function` 在 byteCode 中其實會被設為 **SuspendLambda** 的繼承者。

而以下便是 **getTasks$2** 對應的 Java code：

```java
@Nullable
public Object getTasks(@NotNull Continuation $completion) {
  return BuildersKt.withContext((CoroutineContext)this.ioDispatcher, (Function2)(new Function2((Continuation)null) {
     int label;

     @Nullable
     public final Object invokeSuspend(@NotNull Object $result) {
        Result var2;
        Exception var10000;
        label34: {
           Object var6 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
           Object var10;
           boolean var10001;
           switch (this.label) {
              case 0:
                 ResultKt.throwOnFailure($result);

                 try {
                    TasksDao var11 = TasksLocalDataSource.this.tasksDao;
                    this.label = 1;
                    var10 = var11.getTasks(this);
                 } catch (Exception var8) {
                    var10000 = var8;
                    var10001 = false;
                    break label34;
                 }

                 if (var10 == var6) {
                    return var6;
                 }
                 break;
              case 1:
                 try {
                    ResultKt.throwOnFailure($result);
                    var10 = $result;
                    break;
                 } catch (Exception var9) {
                    var10000 = var9;
                    var10001 = false;
                    break label34;
                 }
              default:
                 throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
           }

           try {
              Object var4 = var10;
              var2 = (Result)(new Result.Success(var4));
              return var2;
           } catch (Exception var7) {
              var10000 = var7;
              var10001 = false;
           }
        }

        Exception e = var10000;
        var2 = (Result)(new Result.Error(e));
        return var2;
     }

     @NotNull
     public final Continuation create(@Nullable Object value, @NotNull Continuation completion) {
        Intrinsics.checkNotNullParameter(completion, "completion");
        Function2 var3 = new <anonymous constructor>(completion);
        return var3;
     }

     public final Object invoke(Object var1, Object var2) {
        return ((<undefinedtype>)this.create(var1, (Continuation)var2)).invokeSuspend(Unit.INSTANCE);
     }
  }), $completion);
}
```

接下來，我們看看 **SuspendLambda** 是什麼吧。

## SuspendLambda

```kotlin
@SinceKotlin("1.3")
// Suspension lambdas inherit from this class
internal abstract class SuspendLambda(
    public override val arity: Int,
    completion: Continuation<Any?>?
) : ContinuationImpl(completion), FunctionBase<Any?>, SuspendFunction {
    constructor(arity: Int) : this(arity, null)

    public override fun toString(): String =
        if (completion == null)
            Reflection.renderLambdaToString(this) // this is lambda
        else
            super.toString() // this is continuation
}
```

原來 **SuspendLambda** 是一個 **SuspendFunction**，但其實這只是一個用來分類的介面：

```kotlin
@SinceKotlin("1.3")
// To distinguish suspend function types from ordinary function types all suspend function types shall implement this interface
internal interface SuspendFunction
```

而 **FunctionBase** 則是用來帶入 `arity` 參數 ：

```kotlin
interface FunctionBase<out R> : Function<R> {
    val arity: Int
}
```

最後 **ContinuationImpl** 便是 **Continuation** 的實作：

```kotlin
@SinceKotlin("1.3")
// State machines for named suspend functions extend from this class
internal abstract class ContinuationImpl(
    completion: Continuation<Any?>?,
    private val _context: CoroutineContext?
) : BaseContinuationImpl(completion)

@SinceKotlin("1.3")
internal abstract class BaseContinuationImpl(
    // This is `public val` so that it is private on JVM and cannot be modified by untrusted code, yet
    // it has a public getter (since even untrusted code is allowed to inspect its call stack).
    public val completion: Continuation<Any?>?
) : Continuation<Any?>, CoroutineStackFrame, Serializable
```

因此，我們現在知道 **getTasks$2** 也是一種 **Continuation**。


接下來，我們來看看 **Continuation** 相關的型別吧。



# Continuation
```kotlin
@SinceKotlin("1.3")
public interface Continuation<in T> {
    public val context: CoroutineContext
    public fun resumeWith(result: Result<T>)
}
```
**Continuation** 有以下實作者：
- **SequenceBuilderIterator**
- **DeepRecursiveScopeImpl**
- **SafeContinuation**
- **StackFrameContinuation**
- **RunSuspend**
- **DebugProbesImpl.CoroutineOwner**

還有以下抽象類別：
- **BaseContinuationImpl**
- **ContinuationImpl**
- **SuspendLambda**
- **AbstractCoroutine**

我們先從抽象類別來看吧。

## BaseContinuationImpl
```kotlin
@SinceKotlin("1.3")
internal abstract class BaseContinuationImpl(
    // This is `public val` so that it is private on JVM and cannot be modified by untrusted code, yet
    // it has a public getter (since even untrusted code is allowed to inspect its call stack).
    public val completion: Continuation<Any?>?
) : Continuation<Any?>, CoroutineStackFrame, Serializable
```

**BaseContinuationImpl** 讓之後的 **Continuation** 增加了 serialization 與 debug 的作用。

除此之外， **BaseContinuationImpl** 還新增了一個抽象方法：
```kotlin
protected abstract fun invokeSuspend(result: Result<Any?>): Any?
```

還有三個 open 方法 `releaseIntercepted` 與 `create` ：
```kotlin
protected open fun releaseIntercepted() {
    // does nothing here, overridden in ContinuationImpl
}

public open fun create(completion: Continuation<*>): Continuation<Unit> {
    throw UnsupportedOperationException("create(Continuation) has not been overridden")
}

public open fun create(value: Any?, completion: Continuation<*>): Continuation<Unit> {
    throw UnsupportedOperationException("create(Any?;Continuation) has not been overridden")
}
```

實作 **BaseContinuationImpl** 的抽象類別包括：
- **RestrictedContinuationImpl**
- **ContinuationImpl**

但無論是誰繼承 **BaseContinuationImpl** ，調用的 `resumeWith` 皆會由 **BaseContinuationImpl** 實作：

```kotlin
// This implementation is final. This fact is used to unroll resumeWith recursion.
public final override fun resumeWith(result: Result<Any?>) {
    // This loop unrolls recursion in current.resumeWith(param) to make saner and shorter stack traces on resume
    var current = this
    var param = result
    while (true) {
        // Invoke "resume" debug probe on every resumed continuation, so that a debugging library infrastructure
        // can precisely track what part of suspended callstack was already resumed
        probeCoroutineResumed(current)
        with(current) {
            val completion = completion!! // fail fast when trying to resume continuation without completion
            val outcome: Result<Any?> =
                try {
                    val outcome = invokeSuspend(param)
                    if (outcome === COROUTINE_SUSPENDED) return
                    Result.success(outcome)
                } catch (exception: Throwable) {
                    Result.failure(exception)
                }
            releaseIntercepted() // this state machine instance is terminating
            if (completion is BaseContinuationImpl) {
                // unrolling recursion via loop
                current = completion
                param = outcome
            } else {
                // top-level completion reached -- invoke and return
                completion.resumeWith(outcome)
                return
            }
        }
    }
}
```

`resumeWith` 的行為有以下的流程：
1. 將 `current` 指向 this \<**BaseContinuationImpl**\> ; `param` 指向 `result` \<**Result**\>
2. 進行無限 loop 並無限地觀察 `current`
3. 通過 `probeCoroutineResumed` 觀察 `current` 的狀態。 但 `probeCoroutineResumed` 的實作已被 debugger 取代
4. 檢查 `completion != null`，但卻會通過強行解開 `completion!!`。 所以調用 `resumeWith` 的方法需要記得用 `try ... catch ...` 來接住 exception。
5. 通過 `invokeSuspend (param)` 取得使用 **Result** 包裹起來的 `outcome`
   如果 `invokeSuspend` 的結果是 **COROUTINE_SUSPENDED** 那就會直接結束 `resumeWith`。
   否則，這便表示 `current` 已經完成。
6. **Continuation** 完成之後就會通過 `releaseIntercepted` 試圖將 `context` 中的 **ContinuationInterceptor** 從 `parentHandle` \<**DisposableHandle**\> 移除。
7. 檢查目前的 `complete` 是否為 **BaseContinuationImpl**
   若 `true`，那就需要讀取下一個 `complete` 並重新回到 (2)
   否則，這就是最後的 **Continuation**。 所以直接調用 `completion.resumeWith(outcome)` 並 return

<u><b>那這裡所說的最後一個 **Continuation** 會是誰呢？</b></u>

其實就是 **Continuation** 的繼承者們。


### RestrictedContinuationImpl
```kotlin
@SinceKotlin("1.3")
// State machines for named restricted suspend functions extend from this class
internal abstract class RestrictedContinuationImpl(
    completion: Continuation<Any?>?
) : BaseContinuationImpl(completion) {
    init {
        completion?.let {
            require(it.context === EmptyCoroutineContext) {
                "Coroutines with restricted suspension must have EmptyCoroutineContext"
            }
        }
    }

    public override val context: CoroutineContext
        get() = EmptyCoroutineContext
}
```

**RestrictedContinuationImpl** 主要限制了 `context` 為 **EmptyCoroutineContext**。 另外，它還在 `init` 時確認傳入的 `completion` 的 `context` 必須是 **EmptyCoroutineContext**。

也就是說 **RestrictedContinuationImpl** 的 `context` 必須是 **EmptyCoroutineContext**。

繼承這個抽象類別的就只有 **RestrictedSuspendLambda**。

#### RestrictedSuspendLambda
```kotlin
@SinceKotlin("1.3")
// Restricted suspension lambdas inherit from this class
internal abstract class RestrictedSuspendLambda(
    public override val arity: Int,
    completion: Continuation<Any?>?
) : RestrictedContinuationImpl(completion), FunctionBase<Any?>, SuspendFunction {
    constructor(arity: Int) : this(arity, null)

    public override fun toString(): String =
        if (completion == null)
            Reflection.renderLambdaToString(this) // this is lambda
        else
            super.toString() // this is continuation
}
```

這個抽象類別多了 `arity` 的傳入。 這是由編輯器轉換成 Java 時會用到。 我們可以從一開始的 byteCode 範例中看到類似的用法。


### ContinuationImpl
```kotlin
@SinceKotlin("1.3")
// State machines for named suspend functions extend from this class
internal abstract class ContinuationImpl(
    completion: Continuation<Any?>?,
    private val _context: CoroutineContext?
) : BaseContinuationImpl(completion)
```

**ContinuationImpl** 除了可以傳入 **Continuation** 外，還可以傳入 **CoroutineContext**。 如果沒有提供 `_context` 時， **ContinuationImpl** 便會使用 `completion` 的 `context`：

```kotlin
constructor(completion: Continuation<Any?>?) : this(completion, completion?.context)
```

另外， **ContinuationImpl** 中還有一個 `intercepted`：

```kotlin
@Transient
private var intercepted: Continuation<Any?>? = null
```

這個 `intercepted` 會通過 `intercepted()` 取得：
```kotlin
public fun intercepted(): Continuation<Any?> =
    intercepted
        ?: (context[ContinuationInterceptor]?.interceptContinuation(this) ?: this)
            .also { intercepted = it }
```

`intercepted()` 的流程會是以下：
1. 檢查 `context` 是否有 **ContinuationInterceptor**
2. 若 `true`，那就調用 `interceptContinuation(this)` 並將 **ContinuationImpl** 包裹在 **ContinuationInterceptor** 中。 最後就會將 `intercepted` 指向結果。
3. 反之，就會直接將 `intercepted` 指向 this。

至於包裹著 **ContinuationImpl** 的 **ContinuationInterceptor** 其實是 **CoroutineDispatcher**。 而 `interceptContinuation` 則會回傳一個 **DispatchedContinuation**：

```kotlin
public final override fun <T> interceptContinuation(continuation: Continuation<T>): Continuation<T> =
    DispatchedContinuation(this, continuation)
```

最後，**ContinuationImpl** 還覆寫了 `releaseIntercepted`：

```kotlin
protected override fun releaseIntercepted() {
    val intercepted = intercepted
    if (intercepted != null && intercepted !== this) {
        context[ContinuationInterceptor]!!.releaseInterceptedContinuation(intercepted)
    }
    this.intercepted = CompletedContinuation // just in case
}
```

這方法會在 **BaseContinuationImpl** 的 `resumeWith` 中在 `current` \<**Continuation**\> 完成後所調用的。

`releaseIntercepted` 會有以下流程：
1. 檢查 `intercepted` 是否為 `null` 且 是否為 `this`
   之所以要檢查 `this` 是因為 `intercepted` 只有在 `context` 沒有 **ContinuationInterceptor** 時被設為 intercepted。
2. 若兩者皆為 `true`，這便表示 `context` 有 **ContinuationInterceptor**。 而這個 **ContinuationInterceptor** 則是 **CoroutineDispatcher**。 所以 `releaseInterceptedContinuation` 的行為便是以下：

   <br>

   ```kotlin
   // CoroutineDispatcher
   public final override fun releaseInterceptedContinuation(continuation: Continuation<*>) {
        /*
         * Unconditional cast is safe here: we only return DispatchedContinuation from `interceptContinuation`,
         * any ClassCastException can only indicate compiler bug
         */
        val dispatched = continuation as DispatchedContinuation<*>
        dispatched.release()
    }

    // DispatchedContinuation
    fun release() {
        /*
         * Called from `releaseInterceptedContinuation`, can be concurrent with
         * the code in `getResult` right after `trySuspend` returned `true`, so we have
         * to wait for a release here.
         */
        awaitReusability()
        reusableCancellableContinuation?.detachChild()
    }

    // CancellableContinuationImpl
    internal fun detachChild() {
        val handle = parentHandle ?: return
        handle.dispose()
        parentHandle = NonDisposableHandle
    }
   ```
   <br>

   通過 `releaseInterceptedContinuation`， **ContinuationImpl** 會將 `intercepted` 從 **DisposableHandle** 移除。

3. 將 `intercepted` 指向 **CompletedContinuation**

   <br>

   ```kotlin
   internal object CompletedContinuation : Continuation<Any?> {
       override val context: CoroutineContext
           get() = error("This continuation is already complete")

       override fun resumeWith(result: Result<Any?>) {
           error("This continuation is already complete")
       }

       override fun toString(): String = "This continuation is already complete"
   }
   ```
   <br>
   可以注意到 **CompletedContinuation** 是無法取得 `context` 也無法調用 `resumeWith`。

實作 **ContinuationImpl** 的是兩個抽象類別：
- **SuspendLambda**
- **SafeCollector**


#### SuspendLambda

**SuspendLambda** 就是在 `suspend` 方法轉換成 Java 時所繼承的抽象類別。

```kotlin
@SinceKotlin("1.3")
// Suspension lambdas inherit from this class
internal abstract class SuspendLambda(
    public override val arity: Int,
    completion: Continuation<Any?>?
) : ContinuationImpl(completion), FunctionBase<Any?>, SuspendFunction {
    constructor(arity: Int) : this(arity, null)

    public override fun toString(): String =
        if (completion == null)
            Reflection.renderLambdaToString(this) // this is lambda
        else
            super.toString() // this is continuation
}
```

#### SafeCollector

```kotlin
@Suppress("CANNOT_OVERRIDE_INVISIBLE_MEMBER", "INVISIBLE_MEMBER", "INVISIBLE_REFERENCE", "UNCHECKED_CAST")
internal actual class SafeCollector<T> actual constructor(
    @JvmField internal actual val collector: FlowCollector<T>,
    @JvmField internal actual val collectContext: CoroutineContext
) : FlowCollector<T>, ContinuationImpl(NoOpContinuation, EmptyCoroutineContext), CoroutineStackFrame
```

**SafeCollector**


```kotlin
private object NoOpContinuation : Continuation<Any?> {
    override val context: CoroutineContext = EmptyCoroutineContext

    override fun resumeWith(result: Result<Any?>) {
        // Nothing
    }
}
```






其中我們最常用到的是 **SafeContinuation**，所以我們會專注在這個類別中。

## SequenceBuilderIterator
```kotlin
private class SequenceBuilderIterator<T> : SequenceScope<T>(), Iterator<T>, Continuation<Unit>
```

**SequenceScope** 是一個會通過 `suspend` 方法來將 `value` 傳出的抽象類別：

```kotlin
@RestrictsSuspension
@SinceKotlin("1.3")
public abstract class SequenceScope<in T> internal constructor() {
    public abstract suspend fun yield(value: T)

    public abstract suspend fun yieldAll(iterator: Iterator<T>)

    public suspend fun yieldAll(elements: Iterable<T>) {
        if (elements is Collection && elements.isEmpty()) return
        return yieldAll(elements.iterator())
    }

    public suspend fun yieldAll(sequence: Sequence<T>) = yieldAll(sequence.iterator())
}
```


## DeepRecursiveScopeImpl
```kotlin
@Suppress("UNCHECKED_CAST")
private class DeepRecursiveScopeImpl<T, R>(
    block: suspend DeepRecursiveScope<T, R>.(T) -> R,
    value: T
) : DeepRecursiveScope<T, R>(), Continuation<R>
```

## SafeContinuation




# Builders.withContext

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













<br><br><br><br><br><br><br><br><br>
