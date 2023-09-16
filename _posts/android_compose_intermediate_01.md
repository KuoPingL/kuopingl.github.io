---
layout: post
title:  "Android Compose - How Compose Works ? (中文)"
date:   2022-08-08 16:15:07 +0800
categories: [中文, jetpack compose, android, intermediate]
---

## 什麼是 Compose ?
我們雖然在 [基礎篇]() 看過一些 Jetpack Compose 的操作， 但應該還不理解 Compose 的核心。

在 [A Jetpack Compose by any other name](https://jakewharton.com/a-jetpack-compose-by-any-other-name/) 的文章中 [Jake Wharton](https://jakewharton.com/) 提到 ：

><br>
>
> 我們現在所看到的 **Jetpack Compose** 其實是由兩個不同的專案結合而成的：
> 1. **Android UI 工具包**
      此專案是將 **React** 中的宣告式元件經由像是 Svelte 的 **AOT 編輯器** 包起來，在使用 **Kotlin** 讓他成為 Android UI 工具包。如此一來， 全部 View-based UI 的程式設計就可以從指令式變成宣告式了。
><br>
>
> 2. **UI 行為的統整**
>    經由減少 OEM (代工商) 與 UI 之間的 touchpoints，應該指的是客製化方面，就能統整 UI 行為。 這也在創建好的 UI 上減少不少的成本。
>
> 最後這兩個專案也變得密不可分了。
> 但有趣的是，這兩個專案都**只需要用到 Compose** ，而*不需要 Compose UI*。
><br><br/>


這是為什麼呢？

><br>
>
>原來 ：
>Compose is, at its core, **a general-purpose tool for managing a tree of nodes of any type**
>
>意思是說， Compose 本身就是用來**管理樹狀結構** 的工具。
>而所謂的「樹狀結構」其實還可以代表**任何一種類型的資料**。 也就是說， Compose 是可以跨平台的。
>
> Touchlab 也有寫了一篇關於[如何將 Compose 運用在 iOS 上](https://touchlab.co/compose-ui-for-ios/)，有興趣的可以看看。
><br><br/>

由於 Compose 所管理的樹狀結構不在乎 **節點 (node)** 的類型，所以他們也不一定是要 UI 相關的。

- 在 Presenter 層，他們可以是 View 的**狀態 (state)**。
- 在 Data 層，他們可以是資料模型 (model)。
- 他們甚至可以是原生類別。

而 Android 則是選擇將能夠**自行在 Canvas 上顯示出來的 Compose UI 工具包內部類型** 設為 Compose 中，樹狀的節點。

也因為如此， Composer 除了是樹狀管理者， 也成了渲染 Android 與 Desktop 軟體的 UI 工具包與 DSL。 這就是我們所知道的 Compose UI 了。

所以說了那麼多， Compose 的核心到底是什麼呢？

其實 Compose 的核心成員只有兩個 :

   1. Compose **Compiler**
   2. Compose **Runtime**

而 UI 也是基於這兩者而產生的。

<center>
<img src = "/images/posts/jekyll/jetpack_compose/compse_source_to_ui.png"/>
Jetpack Compose internals
</center>

<br>



**參考資料** :
1. A Jetpack Compose by any other name, https://jakewharton.com/a-jetpack-compose-by-any-other-name/
2. Compose UI for iOS, https://touchlab.co/compose-ui-for-ios/
3. Castillo J.,  [Jetpack Compose internals](https://jorgecastillo.dev/book/)

## 深入 Jetpack Compose

接下來，我們要看看 Jetpack Compose 的架構與運作原理。

### Level 0 : 註解 (Annotation)

在使用 Jetpack Compose 時，我們會遇到不同的註解，像是：

- **Composable**
- **ComposeCompilerApi**
- **InternalComposeApi**
- **DisallowComposableCalls**

等等

一般來說，註解的使用必需要對應的 Annotation Processor Tool (apt) 或 Kotlin APT (kapt) 才能夠運作。

但 Jetpack Compose 並不需要 apt，因為它使用的是 **Kotlin Compiler Plugin**。

以下是它們兩者的運作上的差異：

1. 運算時間點
   - apt/kapt 需要在編譯前才進行運行。
   - Kotlin Compiler Plugin 讓 Jetpack Compose 可以在**編譯的時候就對程式進行了暸解與修改**。 所以省下不少時間。

2. 程式碼的修改
   - apt/kapt 只能對程式碼進行修改
   - Kotlin Compiler Plugin 卻可以對字結碼 (bytecode) 進行修改

以上的差異也讓 Kotlin Compiler Plugin 多了不少的優勢。

另外，對 Jetpack Compose 而言， Kotlin Compiler Plugin 也算是核心之一 ：

<center>
<img src = "/images/posts/jekyll/jetpack_compose/compose_internal_libraries.png" style = "width:80%"/>

<a href = "https://blog.csdn.net/vitaviva/article/details/118097292">Jetpack Compose Runtime 与 NodeTree 管理</a>
</center>


**參考資料** :
1. Castillo J.,  [Jetpack Compose internals](https://jorgecastillo.dev/book/)
2. fundroid_方卓, <a href = "https://blog.csdn.net/vitaviva/article/details/118097292">Jetpack Compose Runtime 与 NodeTree 管理</a>
3. [谷歌开发者大会扔物线演讲原稿整理：Jetpack Compose](https://rengwuxian.com/jetpack-compose-1/)

### Level 1 : Jetpack Compose 的 View

我們已經知道 Compose 是用來管理 Tree 的，那是不是表示我們可以將 View 當作是 node 來進行管理呢？

是的 ... 我們*是可以這樣做*，但是為了換成宣告式程式碼， Jetpack Compose 並沒有這麼做。 取而代之的是 **LayoutNode** 。

><br> **那是不是表示 View 都不見了呢？**
><br>

當然不是啦。

在 App 運作時，我們依舊需要 **PhoneWindow**, **DecorView**, **ComposeView**, **AndroidComposeView** 等等的 UI 底層架構，之後才會放上對應的 Compose 類別。

<center>
<img src = "/images/posts/jekyll/jetpack_compose/compose_simple_architecture.png" style="width:80%"/>
<br> ComposeUI 的結果
</center>

### Level 1.1 : LayoutNode 簡介

現在最想知道的應該是： **LayoutNode** 是如何取代 **View** 的呢？

但在回答這問題之前，我們需要先知道 **LayoutNode** 是如何產生的。

#### LayoutNode 的生成

在啟動 App 時，第一個生成的 **LayoutNode** 是在 **AndroidComposeView** 的創建時所生成：

```Kotlin
// AndroidComposeView.kt
override val root = LayoutNode().also {
    it.measurePolicy = RootMeasurePolicy
    it.density = density
    // Composed modifiers cannot be added here directly
    it.modifier = Modifier
        .then(semanticsModifier)
        .then(rotaryInputModifier)
        .then(_focusManager.modifier)
        .then(keyInputModifier)
}
```

除此之外，**LayoutNode** 還會在 `ReusableComposeNode` 與 `ComposeNode` 中經由 factory 的調用而生成 ：

```Kotlin
// Composable.kt

if (currentComposer.inserting) {
    currentComposer.createNode(factory)
} else {
    currentComposer.useNode()
}

// 或

if (currentComposer.inserting) {
    currentComposer.createNode { factory() }
} else {
    currentComposer.useNode()
}

```

`factory` 指向的都是 LayoutNode.Constructor

而主要生成 **ReusableComposeNode** 的則是經由調用 `Layout` 方法：

<details>
<summary><b> Layout </b></summary>

```Kotlin
// Layout.kt
@Suppress("ComposableLambdaParameterPosition")
@UiComposable
@Composable inline fun Layout(
    content: @Composable @UiComposable () -> Unit,
    modifier: Modifier = Modifier,
    measurePolicy: MeasurePolicy
) {
    val density = LocalDensity.current
    val layoutDirection = LocalLayoutDirection.current
    val viewConfiguration = LocalViewConfiguration.current

    // 調用 ReusableComposeNode
    ReusableComposeNode<ComposeUiNode, Applier<Any>>(
        factory = ComposeUiNode.Constructor,
        update = {
            set(measurePolicy, ComposeUiNode.SetMeasurePolicy)
            set(density, ComposeUiNode.SetDensity)
            set(layoutDirection, ComposeUiNode.SetLayoutDirection)
            set(viewConfiguration, ComposeUiNode.SetViewConfiguration)
        },
        skippableUpdate = materializerOf(modifier),
        content = content
    )
}
```

</details>

<details>
<summary><b>ReusableComposeNode</b></summary>

```Kotlin
@Composable @ExplicitGroupsComposable
inline fun <T, reified E : Applier<*>> ReusableComposeNode(
    noinline factory: () -> T,
    update: @DisallowComposableCalls Updater<T>.() -> Unit,
    noinline skippableUpdate: @Composable SkippableUpdater<T>.() -> Unit,
    content: @Composable () -> Unit
) {
    if (currentComposer.applier !is E) invalidApplier()
    currentComposer.startReusableNode()

    // 生成 / 取得 LayoutNode
    if (currentComposer.inserting) {
        currentComposer.createNode(factory)
    } else {
        currentComposer.useNode()
    }

    currentComposer.disableReusing()
    Updater<T>(currentComposer).update()
    currentComposer.enableReusing()
    SkippableUpdater<T>(currentComposer).skippableUpdate()
    currentComposer.startReplaceableGroup(0x7ab4aae9)
    content()
    currentComposer.endReplaceableGroup()
    currentComposer.endNode()
}
```

</details>

<br>

所以只要會調用 `Layout` 的元件，像是 `Box`, `Row`, `Column` 等等的所有 UI 元件，都會產生出 **LayoutNode**。

在生成 **LayoutNode** 的同時，也會生成兩個重要物件：
1. **NodeChain** (`nodes`)
2. **LayoutNodeLayoutDelegate** (`layoutDelegate`)
   - 提供當下 layout 狀況 : `layoutDelegate.layoutState`


><br>
>
>**NodeChain** 的主要功能就是以 linkedList 的方式存放 **Modifier**
><br><br/>

**Modifer** 的功能大致上可以分成以下 ：

```Kotlin
// NodeKind.kt
@OptIn(ExperimentalComposeUiApi::class)
internal object Nodes {
    val Any                 = NodeKind<Modifier.Node>(0b1)
    val Layout              = NodeKind<LayoutModifierNode>(0b1 shl 1)
    val Draw                = NodeKind<DrawModifierNode>(0b1 shl 2)
    val Semantics           = NodeKind<SemanticsModifierNode>(0b1 shl 3)
    val PointerInput        = NodeKind<PointerInputModifierNode>(0b1 shl 4)
    val Locals              = NodeKind<ModifierLocalNode>(0b1 shl 5)
    val ParentData          = NodeKind<ParentDataModifierNode>(0b1 shl 6)
    val LayoutAware         = NodeKind<LayoutAwareModifierNode>(0b1 shl 7)
    val GlobalPositionAware = NodeKind<GlobalPositionAwareModifierNode>(0b1 shl 8)
    val IntermediateMeasure = NodeKind<IntermediateLayoutModifierNode>(0b1 shl 9)
    // ...
}
```



#### LayoutNode 的顯示

與 **View** 相似， **LayoutNode** 也有對應的  `measure`, `layout` 與 `draw` 方法。

但 **LayoutNode** 會將這些方法間接地 delegate 給 **MeasurePassDelegate**。

接下來我們就一一說明吧。

#### **Measure** 的實現

由於 **AndroidComposeView** 是第一個與 **LayoutNode** 連接的元件，所以我們需要從 **ComposeView** 調用 `measure` 時開始觀察整個流程。

<center>
<img src = "/images/posts/jekyll/jetpack_compose/flows/composeview_to_androidcomposeview_measure.png" style = "width:80%"/>
</center>


<br>

從 flowchart 可以看出 **AndroidComposeView** 會通過 **MeasureAndLayoutDelegate** 通知 **LayoutNode** (root) 需要進行 measure。





```Kotlin
// LayoutNode.kt
internal fun draw(canvas: Canvas) = outerCoordinator.draw(canvas)
```



#### LayoutNode

<br><br><br>


我們可以經由追蹤 MainActivity 的 `setContent` 做了些什麼來暸解整個過程。

#### MainActivity 的 setContent

當我們開啟新的 Compose 專案時，我們會看到以下：

```Kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContent {
        // 這裡是 ComposeView 的 content
        ComposeDemosTheme {
            // A surface container using the 'background' color from the theme
            Surface(
                modifier = Modifier.fillMaxSize(),
                color = MaterialTheme.colors.background
            ) {
                Greeting("Android")
            }
        }
    }
}
```

其中的 `setContent` 主要的作用就是建立 **ComposeView** 並進行基本設定 ：

```Kotlin
public fun ComponentActivity.setContent(
    parent: CompositionContext? = null,
    content: @Composable () -> Unit
) {
    // 1. 建立 ComposeView， 這是一個 ViewGroup
    val existingComposeView = window.decorView
        .findViewById<ViewGroup>(android.R.id.content)
        .getChildAt(0) as? ComposeView

    if (existingComposeView != null) with(existingComposeView) {

        // 2. 設定 CompositionContext 並調用 ComposeView#setContent
        setParentCompositionContext(parent)
        setContent(content)

    } else ComposeView(this).apply {
        // 第一次會進到這裡

        setParentCompositionContext(parent)
        setContent(content)

        // 3. 設定 ViewTreeLifecycleOwner, ViewTreeViewModelStoreOwner 與 ViewTreeSavedStateRegistryOwner
        setOwners()

        // 4. 調用 ComponentActivity#setContentView
        setContentView(this, DefaultActivityContentLayoutParams)
    }
}
```

**ComposeView** 其實就是 ViewGroup。

之後我們還會繼續針對 ComposeView 進行 `setContent`。 不過他的作用只是設定 ComposeView 的 content 是甚麼，也立好






---

在暸解 Compose 的渲染過程前，我們必須要知道 Compose UI 的結構與 View Hierarchy 是有差異的。

原先 Android 的 View 是由 View Hierarchy 也就是 View Tree 所組成。

<center>
<img src = "/images/posts/jekyll/jetpack_compose/view_tree.png"/>
出處：<a href="https://blog.csdn.net/linghu_java/article/details/23882681">ViewGroup的onMeasure和onLayout分析</a>
</center>

<br>

但是在 Compose 我們會看

## 由 Box 談起
我們先看看最基本的結構單元， **Box** ，的架構吧。

```Kotlin
@Composable
inline fun Box(
    modifier: Modifier = Modifier,
    contentAlignment: Alignment = Alignment.TopStart,
    propagateMinConstraints: Boolean = false,
    content: @Composable BoxScope.() -> Unit
) {
    val measurePolicy = rememberBoxMeasurePolicy(contentAlignment, propagateMinConstraints)
    Layout(
        content = { BoxScopeInstance.content() },
        measurePolicy = measurePolicy,
        modifier = modifier
    )
}
```

### @Composable
`@Composable` 是用來定義 Composable 方法的 Annotation ：

```Kotlin
@MustBeDocumented
@Retention(AnnotationRetention.BINARY)
@Target(
    // function declarations
    // @Composable fun Foo() { ... }
    // lambda expressions
    // val foo = @Composable { ... }
    AnnotationTarget.FUNCTION,

    // type declarations
    // var foo: @Composable () -> Unit = { ... }
    // parameter types
    // foo: @Composable () -> Unit
    AnnotationTarget.TYPE,

    // composable types inside of type signatures
    // foo: (@Composable () -> Unit) -> Unit
    AnnotationTarget.TYPE_PARAMETER,

    // composable property getters and setters
    // val foo: Int @Composable get() { ... }
    // var bar: Int
    //   @Composable get() { ... }
    AnnotationTarget.PROPERTY_GETTER
)
annotation class Composable
```
`@Composable` 確保只有 Composable 方法才能調用 Composable 方法。

而每次調用 Composable 方法時，都會有一個 composable context 用來存取方法的相關資訊。

由於只有 Composable 方法才能調用 Composable 方法， 所以這個 context 也會從調用者傳入被調用者。

如此一來後者就可以知道前者的相關訊息了。

### rememberBoxMeasurePolicy

在調用 **Box** 時，第一件事就是會建立一個 remember 相關的資訊。

```Kotlin
// Box
val measurePolicy = rememberBoxMeasurePolicy(contentAlignment, propagateMinConstraints)
```

由名字可以看出， rememberBoxMeasurePolicy 所儲存的狀態是 Box 的 measure policy。

```Kotlin
// rememberBoxMeasurePolicy
@PublishedApi
@Composable
internal fun rememberBoxMeasurePolicy(
    alignment: Alignment,
    propagateMinConstraints: Boolean
) = remember(alignment) {
    if (alignment == Alignment.TopStart && !propagateMinConstraints) {
        DefaultBoxMeasurePolicy
    } else {
        boxMeasurePolicy(alignment, propagateMinConstraints)
    }
}
```

從這源碼可以知道， rememberBoxMeasurePolicy 的主要作用就是當 `alignment` 狀態改變時， Box 會進行判斷並使用對應的 measure policy 。

當 Box 的 Alignment 值為 `Alignment.TopStart` 且 `propagateMinConstraints == false` 時， Box 會調用 `DefaultBoxMeasurePolicy` ：

```Kotlin
internal val DefaultBoxMeasurePolicy: MeasurePolicy = boxMeasurePolicy(Alignment.TopStart, false)
```

但其實也是調用 `boxMeasurePolicy` 罷了。

而 `boxMeasurePolicy` 其實就是建立 **MeasurePolicy**，並進行 `layout` 的行為。

```Kotlin
internal fun boxMeasurePolicy(alignment: Alignment, propagateMinConstraints: Boolean) =
    MeasurePolicy { measurables, constraints ->
        // ...
    }
```

那什麼是 **measure policy** 呢？

#### Measure Policy

**MeasurePolicy** 是一個 SAM ：

```Kotlin
@Stable
fun interface MeasurePolicy {
  fun MeasureScope.measure(
      measurables: List<Measurable>,
      constraints: Constraints
  ): MeasureResult
}
```

其中 **MeasureScope** 定義了能進行 measurement 的範圍。 而 `measure` 方法中則帶有 **Measurable** 陣列與 **Constraints** 參數。

當我們看向 Box 對 MeasurePolicy 的實作時，我們可以看出他其實與 **ViewGroup** 的 `onMeasure` 極為相似。

而且其中還會調用與 `onLayout` 相似的方法 `layout` ：

```Kotlin
internal fun boxMeasurePolicy(alignment: Alignment, propagateMinConstraints: Boolean) =
    MeasurePolicy { measurables, constraints ->

        // 1. 定義 contentConstraints

        if (measurables.isEmpty()) {
            return@MeasurePolicy layout(
                constraints.minWidth,
                constraints.minHeight
            ) {}
        }

        val contentConstraints = if (propagateMinConstraints) {
            constraints
        } else {
            constraints.copy(minWidth = 0, minHeight = 0)
        }

        // 2. 調用 measurable 的 measure 方法，並將回傳值 Placeable 存起來，
        // 這就如同 ViewGroup 調用 child.measure 一樣。

        if (measurables.size == 1) {
            val measurable = measurables[0]
            val boxWidth: Int
            val boxHeight: Int
            val placeable: Placeable
            if (!measurable.matchesParentSize) {
                placeable = measurable.measure(contentConstraints)
                boxWidth = max(constraints.minWidth, placeable.width)
                boxHeight = max(constraints.minHeight, placeable.height)
            } else {
                boxWidth = constraints.minWidth
                boxHeight = constraints.minHeight
                placeable = measurable.measure(
                    Constraints.fixed(constraints.minWidth, constraints.minHeight)
                )
            }
            return@MeasurePolicy layout(boxWidth, boxHeight) {
                placeInBox(placeable, measurable, layoutDirection, boxWidth, boxHeight, alignment)
            }
        }

        val placeables = arrayOfNulls<Placeable>(measurables.size)
        // First measure non match parent size children to get the size of the Box.
        var hasMatchParentSizeChildren = false
        var boxWidth = constraints.minWidth
        var boxHeight = constraints.minHeight
        measurables.fastForEachIndexed { index, measurable ->
            if (!measurable.matchesParentSize) {
                val placeable = measurable.measure(contentConstraints)
                placeables[index] = placeable
                boxWidth = max(boxWidth, placeable.width)
                boxHeight = max(boxHeight, placeable.height)
            } else {
                hasMatchParentSizeChildren = true
            }
        }

        // Now measure match parent size children, if any.
        if (hasMatchParentSizeChildren) {
            // The infinity check is needed for default intrinsic measurements.
            val matchParentSizeConstraints = Constraints(
                minWidth = if (boxWidth != Constraints.Infinity) boxWidth else 0,
                minHeight = if (boxHeight != Constraints.Infinity) boxHeight else 0,
                maxWidth = boxWidth,
                maxHeight = boxHeight
            )
            measurables.fastForEachIndexed { index, measurable ->
                if (measurable.matchesParentSize) {
                    placeables[index] = measurable.measure(matchParentSizeConstraints)
                }
            }
        }

        // 3. 調用 layout 並間接調用 Placeable.place 方法
        // 這如同 ViewGroup 中調用 child.layout 一樣。

        // Specify the size of the Box and position its children.
        layout(boxWidth, boxHeight) {
            placeables.forEachIndexed { index, placeable ->
                placeable as Placeable
                val measurable = measurables[index]
                placeInBox(placeable, measurable, layoutDirection, boxWidth, boxHeight, alignment)
            }
        }
    }
```

相對於 ViewGroup 的源碼， 宣告式 的確比 指令式 更為易讀。

最後調用的 `layout` 方法其實是 **MeasureScope** 中的方法：

```Kotlin
fun layout(
        width: Int,
        height: Int,
        alignmentLines: Map<AlignmentLine, Int> = emptyMap(),
        placementBlock: Placeable.PlacementScope.() -> Unit
    ) = object : MeasureResult {
        override val width = width
        override val height = height
        override val alignmentLines = alignmentLines
        override fun placeChildren() {
            Placeable.PlacementScope.executeWithRtlMirroringValues(
                width,
                layoutDirection,
                placementBlock
            )
        }
    }
```

這方法的作用便是實作 **MeasureResult**，一個擁有 measured layout 相關資料 (size 與 alignment lines) 與 child **配置邏輯** 的介面。

```Kotlin
interface MeasureResult {
    val width: Int
    val height: Int
    val alignmentLines: Map<AlignmentLine, Int>
    fun placeChildren()
}
```


接下來，我們來看看 MeasurePolicy 又是如何得到 Measurable 與 Constraints 的吧。

### Layout

在我們定義了 MeasurePolicy 後， Box 就會調用 `Layout` 方法：

```Kotlin
@Composable
inline fun Box(
    modifier: Modifier = Modifier,
    contentAlignment: Alignment = Alignment.TopStart,
    propagateMinConstraints: Boolean = false,
    content: @Composable BoxScope.() -> Unit
) {
    val measurePolicy = rememberBoxMeasurePolicy(contentAlignment, propagateMinConstraints)
    Layout(
        content = { BoxScopeInstance.content() },
        measurePolicy = measurePolicy,
        modifier = modifier
    )
}
```

Layout 中， 由於 `content` 是被視為 `BoxScope` 的 **extension function**， 所以才可以直接調用 `BoxScopeInstance.content()` 。

其主要目的就是讓 Layout 知道當下的 Scope 屬於 Box 的。

當 Layout 被調用後，`Layout` 會將裝置的 UI 資訊，包括： Density 、 Layout 的方向 (RTL 或 LTR) 與 互動參數與 content, modifier, measurePolicy 一同帶入 `ReusableComposeNode` 中 ：

```Kotlin
@Suppress("ComposableLambdaParameterPosition")
@Composable inline fun Layout(
    content: @Composable () -> Unit,
    modifier: Modifier = Modifier,
    measurePolicy: MeasurePolicy
) {
    val density = LocalDensity.current
    val layoutDirection = LocalLayoutDirection.current
    val viewConfiguration = LocalViewConfiguration.current
    ReusableComposeNode<ComposeUiNode, Applier<Any>>(
        factory = ComposeUiNode.Constructor,
        update = {
            set(measurePolicy, ComposeUiNode.SetMeasurePolicy)
            set(density, ComposeUiNode.SetDensity)
            set(layoutDirection, ComposeUiNode.SetLayoutDirection)
            set(viewConfiguration, ComposeUiNode.SetViewConfiguration)
        },
        skippableUpdate = materializerOf(modifier),
        content = content
    )
}
```

其中的 **ViewConfiguration** 所擁有的互動參數如下：

```Kotlin
interface ViewConfiguration {
    val longPressTimeoutMillis: Long
    val doubleTapTimeoutMillis: Long
    val doubleTapMinTimeMillis: Long
    val touchSlop: Float
    val minimumTouchTargetSize: DpSize
        get() = DpSize(48.dp, 48.dp)
}
```

接下來，我們先看看帶入 **ReusableComposeNode** 的參數吧。

#### factory

```Kotlin
factory = ComposeUiNode.Constructor
```

factory 定義了創建 **ComposeUiNode** 的方法，而 `ComposeUiNode.Constructor` 其實會建立 **LayoutNode** :

```Kotlin
// ComposeUiNode.kt
val Constructor: () -> ComposeUiNode = LayoutNode.Constructor

// LayoutNode.kt
internal val Constructor: () -> LayoutNode = { LayoutNode() }
```

所謂的 **LayoutNode** 便是 Compose UI 結構的基礎元素。

以下是 **LayoutNode** 的定義：

```Kotlin
internal class LayoutNode(
  private val isVirtual: Boolean = false
) : Measurable, Remeasurement, OwnerScope, LayoutInfo, ComposeUiNode
```

#### update

```Kotlin
update = {
            set(measurePolicy, ComposeUiNode.SetMeasurePolicy)
            set(density, ComposeUiNode.SetDensity)
            set(layoutDirection, ComposeUiNode.SetLayoutDirection)
            set(viewConfiguration, ComposeUiNode.SetViewConfiguration)
        }
```

`update` 主要功能就是經由 Composer 更新 Slot Table 中的參數。

所謂的 **SlotTable** 其實就是一個用來存取 **CompositionGroup** 的 Iterable。

```Kotlin
internal class SlotTable : CompositionData, Iterable<CompositionGroup>

interface CompositionGroup : CompositionData {
    val key: Any
    val sourceInfo: String?
    val node: Any?
    val data: Iterable<Any?>
}

interface CompositionData {
    val compositionGroups: Iterable<CompositionGroup>
    val isEmpty: Boolean
}
```

#### skippableUpdate

在定義 skippableUpdate 參數時， `Layout` 會對 mmodifier 調用 `materializerOf` 方法：

```Kotlin
skippableUpdate = materializerOf(modifier)
```

`materializerOf` 會做兩件事：

1. 用遞迴的方式將 **ComposedModifier** 利用自己的 factory 轉換成其他 **Modifer** 類型 。並將全部 **Modifer** 用 **CombinedModifier** 包起來。


2. 回傳一個 `@Composable SkippableUpdater<ComposeUiNode>.() -> Unit`

```Kotlin
update {
        set(materialized, ComposeUiNode.SetModifier)
    }
```

所以 `skippableUpdate` 最後會得到 `update{...}` lambda 值。

接下來，我們繼續看 **ReusableComposeNode** 是什麼吧。


### ReusableComposeNode

根據文件的說明， `ReusableComposeNode` 主要的功能如下：

><br>Emits a recyclable node into the composition of type *T*. Nodes emitted inside of content will become children of the emitted node.
><br><br/>

我們可以從源碼中看到更多的細節：

```Kotlin
@Composable @ExplicitGroupsComposable
inline fun <T, reified E : Applier<*>> ReusableComposeNode(
    noinline factory: () -> T,
    update: @DisallowComposableCalls Updater<T>.() -> Unit,
    noinline skippableUpdate: @Composable SkippableUpdater<T>.() -> Unit,
    content: @Composable () -> Unit
) {
    /**
      1. 確保 Composer 的 applier 與 E 相同
         否則就會丟出 IllegalException
    */
    if (currentComposer.applier !is E) invalidApplier()

    /**
      2. 建立 或 準備一個 SlotTable
    */
    currentComposer.startReusableNode()

    /**
      3. inserting 只會在首次 composition 或是 當有新的 node 要被加入時才會為 true。
         當 inserting 為 true 時，就會根據 factory 來創建新的 ComposeUiNode
         若為 false，就會將當下的 Node 設為「使用中」
    */
    if (currentComposer.inserting) {
        currentComposer.createNode(factory)
    } else {
        currentComposer.useNode()
    }

    /**
      4. 將 reusing 參數設為 false
    */
    currentComposer.disableReusing()

    /**
      5. 調用 update()
         Box 中的 update() 只是更新對應的 block

         update =
             {
                set(measurePolicy, ComposeUiNode.SetMeasurePolicy)
                set(density, ComposeUiNode.SetDensity)
                set(layoutDirection, ComposeUiNode.SetLayoutDirection)
                set(viewConfiguration, ComposeUiNode.SetViewConfiguration)
             }
    */
    Updater<T>(currentComposer).update()

    /**
      6. 恢復 reusing
    */
    currentComposer.enableReusing()

    /**
      7. 將 ComposeUiNode.SetModifier 更新成 materialized 後的 CombinedModifier
    */
    SkippableUpdater<T>(currentComposer).skippableUpdate()

    /**
      8. 在 SlotTable 位於當下位置插入 "Replaceable Group" starting marker。
         所謂 Replaceable Group 指的是只能被插入與移除的組件。 他們是無法與 sibling 互換的。

         另外，編譯器將他們插入前需要經過 conditional logic，
         包括 if, when, early return 與 null-coalescing operators，
         判斷後才能插入。
    */
    currentComposer.startReplaceableGroup(0x7ab4aae9)

    /**
      9. 調用 content() 將 Composable function
         裡的 components 建立起來。
         建立的過程其實就與建立 Box 時一樣。
    */
    content()

    /**
      10. 進行寫入
    */
    currentComposer.endReplaceableGroup()

    /**
      11. 進行更新
    */
    currentComposer.endNode()
}
```

從源碼可以看出 **ReusableComposeNode** 其實就是不斷地使用遞迴的方式來 emit node ， 也就是將新的資訊更新至 SlotTable 上。

問題來了：
**SlotTable** 上的 nodes 什麼時候才能真正的顯示在畫面上呢？

為了回答這個問題，我們需要研究一下 MainActivity.kt 中 `onCreate` 做了什麼了。

## MainActivity
以下是預設的 MainActivity :

```Kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            ComposeDemosTheme {
                // A surface container using the 'background' color from the theme
                Surface(
                    modifier = Modifier.fillMaxSize(),
                    color = MaterialTheme.colors.background
                ) {
                    Greeting("Android")
                }
            }
        }
    }
}
```

在 `setContent` 之後的方法都是 `@Composable` 方法。
所以重點落在 `setContent` 是怎麼實作的。

### ComponentActivity.setContent

```Kotlin
public fun ComponentActivity.setContent(
    parent: CompositionContext? = null,
    content: @Composable () -> Unit
) {

    /**
      1. 將 Window 的 DecorView 視為 ComposeView
    */
    val existingComposeView = window.decorView
        .findViewById<ViewGroup>(android.R.id.content)
        .getChildAt(0) as? ComposeView

    /**
      2. 若有 ComposeView，就將其當作 parent 並調用 setContent(content)
      3. 否則，就建立 ComposeView，調用 setContent(content) 後，還要調用 setOwners 與 setContent(this, DefaultActivityContentLayoutParams)
    */
    if (existingComposeView != null) with(existingComposeView) {
        setParentCompositionContext(parent)
        setContent(content)
    } else ComposeView(this).apply {
        // Set content and parent **before** setContentView
        // to have ComposeView create the composition on attach
        setParentCompositionContext(parent)
        setContent(content)
        // Set the view tree owners before setting the content view so that the inflation process
        // and attach listeners will see them already present
        setOwners()
        setContentView(this, DefaultActivityContentLayoutParams)
    }
}
```

我們先看看啥是 **ComposeView** 吧。

### 啥是 ComposeView ?

><br>
>
> 注意了 ： **ComposeView** 原來就是 **ViewGroup** 。
>
>更正確的說法是， **ComposeView** 是一個可以主持? (host) Compose UI 內容 的 ViewGroup。
><br><br/>

```Kotlin
class ComposeView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : AbstractComposeView(context, attrs, defStyleAttr) {

    private val content = mutableStateOf<(@Composable () -> Unit)?>(null)

    @Suppress("RedundantVisibilityModifier")
    protected override var shouldCreateCompositionOnAttachedToWindow: Boolean = false
        private set

    @Composable
    override fun Content() {
        content.value?.invoke()
    }

    override fun getAccessibilityClassName(): CharSequence {
        return javaClass.name
    }

    /**
     * Set the Jetpack Compose UI content for this view.
     * Initial composition will occur when the view becomes attached to a window or when
     * [createComposition] is called, whichever comes first.
     */
    fun setContent(content: @Composable () -> Unit) {
        shouldCreateCompositionOnAttachedToWindow = true
        this.content.value = content
        if (isAttachedToWindow) {
            createComposition()
        }
    }
}

abstract class AbstractComposeView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : ViewGroup(context, attrs, defStyleAttr)
```

蠻有趣的是，由於 `content` 是可變的，所以 **ComposeView** 會把 `content` 設為 `mutableStateOf` :

```Kotlin
private val content = mutableStateOf<(@Composable () -> Unit)?>(null)
```

而 `setContent` 也就會更新 `content` 值，並調用 `createComposition`。


```Kotlin
fun createComposition() {
    check(parentContext != null || isAttachedToWindow) {
        "createComposition requires either a parent reference or the View to be attached" +
            "to a window. Attach the View or call setParentCompositionReference."
    }
    ensureCompositionCreated()
}

@Suppress("DEPRECATION") // Still using ViewGroup.setContent for now
private fun ensureCompositionCreated() {
    if (composition == null) {
        try {
            creatingComposition = true
            composition = setContent(resolveParentCompositionContext()) {
                Content()
            }
        } finally {
            creatingComposition = false
        }
    }
}
```

`createComposition` 會先檢查 parent 是否 attach to window。 若有就會調用 `ensureCompositionCreated`。


`ensureCompositionCreated` 做了兩件事：

```Kotlin
composition = setContent(
  // 1. 取得 Recomposer : CompositionContext
  resolveParentCompositionContext()
  ) {
    // 2. 調用 ComposeView 的 `Content()` 方法
    //    也就是 content.value?.invoke()
    Content()
}
```

`resolveParentCompositionContext` 是用來取得 **CompositionContext** ：

```Kotlin
private fun resolveParentCompositionContext() = parentContext
        ?: findViewTreeCompositionContext()?.cacheIfAlive()
        ?: cachedViewTreeCompositionContext?.get()?.takeIf { it.isAlive }
        ?: windowRecomposer.cacheIfAlive()
```

**CompositionContext** 的作用是將兩個 Composition 「連結」 在一起，也因為如此，他也可以被當作  "parent" of a root composition， 也就是 **Recomposer** 。

另外一個實作 **CompositionContext** 的是 **ComposerImpl** ， 也是用來存放 SlotTable 的類別。

而 `ensureCompositionCreated` 的 `setContent` 則是調用 **AbstractComposeView** 的 `setContent` ：

```Kotlin
internal fun AbstractComposeView.setContent(
    parent: CompositionContext,
    content: @Composable () -> Unit
): Composition {
    GlobalSnapshotManager.ensureStarted()
    val composeView =
        if (childCount > 0) {
            getChildAt(0) as? AndroidComposeView
        } else {
            removeAllViews(); null
        } ?: AndroidComposeView(context).also { addView(it.view, DefaultLayoutParams) }
    return doSetContent(composeView, parent, content)
}
```

## Layouts

### RenderNodeLayer

### OwnedLayout


### AndroidComposeView


## 總結


<br><br><br><br><br><br><br>
