---
layout: post
title:  "Android Compose - An Introduction"
date:   2022-08-08 16:15:07 +0800
categories: [中文, jetpack compose, android, beginner]
---

## 關於 Jetpack Compose ?

**Jetpack Compose** 是 Google 在 2019年5月 推出的現代化 **UI 工具包**。(至今，Google 已推出 Jetpack Compose **1.3.2** 版本)。

它的誕生是啟發於 React, Litho, Vue, Flutter, 等等 Framework。

與 Android 原本的 UI 定義主要的差異在於 Jetpack Compose 使用的是 **宣告式 (Declarative)** 程式設計而非 **指令式 或 命令式 (Imperative)** 程式設計。

當然，還有另外一個很重要的差異，那就是 Jetpack Compose 並不需要使用到 View 更不需要用到 XML 來定義 Layout 與 Attributes。 但這篇文章主要只是讓大家暸解 Jetpack Compose 的基礎運用。

**參考資料**：
1. [Google I/O 公布了 Compose 1.0，你准备好了吗？](https://juejin.cn/post/6965671818598285325)
2. Richardson L., [Compose From First Principles](http://intelligiblebabble.com/compose-from-first-principles/)

### 指令式 vs 宣告式

指令式與宣告式的差異在於：

  - 指令式注重在「如何 ( **How** ) 」達到某種結果
  - 宣告式則注重在想要達到「**怎樣的** ( **What** ) 」結果

我們來參考這 [影片](https://www.youtube.com/watch?v=E7Fbf7R3x6I) 中提到的一個生活化的例子：

><br>
> 假設我們要在一間餐廳用餐，指令式與宣告式會如何表達呢？
>
> - **指令式** 會定出需要的步驟 :
>   1. 要求兩人桌
>   2. 在入口前進 10 步
>   3. 右轉
>   4. 向前走 20 步
>   5. 找到桌子並坐下
><br>
> - **宣告式** 則只在乎結果：
>   1. 要求兩人桌
><br><br/>

從以上範例可以看出，指令式注重在 **細節**，但也讓他較難閱讀。

而宣告式雖然簡短，但卻更 **易於閱讀與理解**。

><br>
>
> **切記**：
>
>宣告式**並非**沒有步驟地完成目標，而是將指令式的步驟 **封裝** 起來，讓使用者更易閱讀而已。
><br><br/>

那這有什麼好處呢？

#### 使用指令式 UI 的缺點

><br>1. 容易出錯
><br><br/>

由於指令式的設計需要親力親為地寫出每個步驟，所以出現錯誤。
當專案越來越大時，需要花費在維護上的時間也會越來越大。

><br>2. 使用指令來定義 UI 通常需要多個語言的配合。
><br><br/>

在 Android 中，我們就需要動用 Java / Kotlin 配合 XML 來建造 UI。 而且把用來定義相同元件的資訊分開來放，只會增加管理、維護與閱讀的困難，沒有過多好處。

><br>3. 指令式可以讓我們通過多種路徑來對 View 進行修改與更新
><br><br/>

當我們要更新畫面時，Android 可以通過以下的方法來對畫面進行 View 的 內部狀態 ( **internal state** ) 的更新
1. 由於 Android 會將畫面看成 小工具 (widget) 樹狀結構 ， 因此他可以通過 `findViewById` 來尋找對應的 View 並進行更新。
2. 我們也可以使用 **DataBinding** 與 XML 連接並動態更新畫面。

如此一來，也會導致除錯與維護上的困難。


### 使用宣告式 UI 的優點

><br>
>
>1. 只需要**單種語言**進行設計。
><br><br/>

無論是 Compose, SwiftUI 或 Flutter 都只需要單一個語言, Kotlin, Swift 或 Dart 來定義 UI。

也因為如此，所需要的維護與開發時間也會減少。

另外， Compose 有不少 API 是 Kotlin 的 DSL。

><br>
>
> 2. 提高**可讀性**
><br><br/>

由於宣告式會將指令式的步驟封裝起來，我們也就不用花時間在思考要如何才能達到結果。而只需要專注在設定最後的呈現畫面也減少在畫面上花費的時間。

另外，將 View-based 元件的建立變成宣告式後，不僅僅將與相同元件相關的程式碼都集合在一起，還增加的可讀性與可維護性。

說完宣告式 (Declarative) 與指令式 (Imperative) 的優缺點後，接下來我們就要談談 Jetpack Compose 的基礎了。


**參考資料**：

1. [工具包、架構 與 函式庫的差異](https://stackoverflow.com/questions/3057526/framework-vs-toolkit-vs-library/33857320#19790148)
2. [Understanding Imperative and Declarative UI Designs](https://www.rootstrap.com/blog/imperative-v-declarative-ui-design-is-declarative-programming-the-future/)
3. YT : [Imperative vs Declarative Programming](https://www.youtube.com/watch?v=E7Fbf7R3x6I)
4. [Understanding Jetpack Compose — part 1 of 2](https://medium.com/androiddevelopers/understanding-jetpack-compose-part-1-of-2-ca316fe39050)
5. YT : [Declarative UI patterns (Google I/O'19)](https://www.youtube.com/watch?time_continue=965&v=VsStyq4Lzxo&feature=emb_title)

# Jetpack Compose 基礎篇

## Dependency 與 API


## UI Building Block

Compose 提供了三種基礎的結構單元，包括：
- Box
- Column
- Row

### Box


### Column

### Row

## Modifier
從上面的範例中可以看到一個共用的參數，那便是 **Modifier** 了。

Modifier 的定義如下：

><br>
>
>An ordered, immutable collection of modifier elements that **decorate or add behavior** to Compose UI elements.
>
><br>For example, backgrounds, padding and click event listeners decorate or add behavior to rows, text or buttons.
><br><br/>

所以 Modifier 是一個 **有序**、**不可變** 的 **修飾集合體**。

以下便是 Modifier 的源碼：

```Kotlin
@Suppress("ModifierFactoryExtensionFunction")
@Stable
interface Modifier {
    fun <R> foldIn(initial: R, operation: (R, Element) -> R): R
    fun <R> foldOut(initial: R, operation: (Element, R) -> R): R
    fun any(predicate: (Element) -> Boolean): Boolean
    fun all(predicate: (Element) -> Boolean): Boolean

    infix fun then(other: Modifier): Modifier =
        if (other === Modifier) this else CombinedModifier(this, other)
}
```

從源碼中可以看出，Modifier 其實就是 Linked List 中的 Node。

他可以利用 `foldIn` 與 `foldOut` 進行 node 的新增。
`foldIn` 會將 R 放在 Element 前面 ; 或 `foldOut` 則會將 R 放在 Element 後方。

><br>
>由於 Modifier 的設定是有順序性的，因此，相同的方法在不同的順序之下也會有不同的效果。
><br><br/>

至於 `then` ，這方法會將新的與舊的 Modifier 放入 **CombinedModifier** 中，進而達到鏈結的效果：

```Kotlin
// 假設我們希望先執行完 A_Modifier 再執行 B_Modifier
// 我們就可以寫成 ：

val C_Modifier = A_Modifier.then(B_Modifier)
// => C_Modifier = CombinedModifier(A_Modifier, B_Modifier)

val E_Modifier = C_Modifier.then(D_Modifer)
// => E_Modifier = CombinedModifier(C_Modifier, D_Modifer)
// => E_Modifier = CombinedModifier(CombinedModifier(A_Modifier, B_Modifier), D_Modifer)
```

如此一來，當我們要進行搜尋的時候， **CombinedModifier** 可以進行遞迴式的搜尋：

```Kotlin
override fun all(predicate: (Modifier.Element) -> Boolean): Boolean =
    outer.all(predicate) && inner.all(predicate)
```

補充一下：

<details>
<summary>方法中所用的 <b>Element</b> 是繼承 Modifier 的介面：</summary>

```Kotlin
interface Element : Modifier {
    override fun <R> foldIn(initial: R, operation: (R, Element) -> R): R =
        operation(initial, this)

    override fun <R> foldOut(initial: R, operation: (Element, R) -> R): R =
        operation(this, initial)

    override fun any(predicate: (Element) -> Boolean): Boolean = predicate(this)

    override fun all(predicate: (Element) -> Boolean): Boolean = predicate(this)
}
```

</details>

<br>

<details>
<summary>而 <b>CombinedModifier</b> 則是實作 Modifier 的 Class ：</summary>

```Kotlin
class CombinedModifier(
    private val outer: Modifier,
    private val inner: Modifier
) : Modifier {
    override fun <R> foldIn(initial: R, operation: (R, Modifier.Element) -> R): R =
        inner.foldIn(outer.foldIn(initial, operation), operation)

    override fun <R> foldOut(initial: R, operation: (Modifier.Element, R) -> R): R =
        outer.foldOut(inner.foldOut(initial, operation), operation)

    override fun any(predicate: (Modifier.Element) -> Boolean): Boolean =
        outer.any(predicate) || inner.any(predicate)

    override fun all(predicate: (Modifier.Element) -> Boolean): Boolean =
        outer.all(predicate) && inner.all(predicate)

    override fun equals(other: Any?): Boolean =
        other is CombinedModifier && outer == other.outer && inner == other.inner

    override fun hashCode(): Int = outer.hashCode() + 31 * inner.hashCode()

    override fun toString() = "[" + foldIn("") { acc, element ->
        if (acc.isEmpty()) element.toString() else "$acc, $element"
    } + "]"
}
```

</details>

<br>

Modifier 的作用除了之前的範例外，我們還可以從 [官方網站](https://developer.android.com/jetpack/compose/modifiers-list) 看到 Modifier 能做出何種設定。


另外，除了基本的設定外，其實還有不少實作了 Modifier.Element 的 Modifiers，包括：

  - 用來定義內容如何呈現的 **LayoutModifier**
  - 具有將圖畫在 layout 的 **DrawModifier**
  - 能夠監控 Focusable Events 的 **FocusEventModifier**

  等等多種 Modifiers。

接下來，我們來看看如何製作基本的 UI Layout 吧。

## UI Layout

### Scaffold

### RecyclerView

### ViewPager

## State 的作用

### remember


><br>
>
>Remember the value produced by **calculation**.
>**calculation** will only be evaluated during the composition.
>Recomposition will always return the value produced by composition.
><br><br/>

```Kotlin

/**
 Remember the value produced by calculation.

 calculation will only be evaluated during the composition.

 Recomposition will always return the value produced by composition.
*/

@Composable
inline fun <T> remember(
    key1: Any?,
    calculation: @DisallowComposableCalls () -> T
): T {
    return currentComposer.cache(currentComposer.changed(key1), calculation)
}
```

`@DisallowComposableCalls` will prevent composable calls from happening inside of the function that it applies to.

## Interaction

## Navigation


### Testing

## ViewModel


### Testing

## Repository


### Testing




<br><br><br><br><br><br><br><br><br><br>
