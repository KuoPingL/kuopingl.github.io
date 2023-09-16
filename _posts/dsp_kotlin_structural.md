---
layout: post
title:  "Design Patterns : Structural Design Patterns"
use_math: true
date:   2022-08-08 16:15:07 +0800
categories: [design patterns, Structural]
---

### 閱讀須知

><br>
>
>想要理解 Structural Design Patterns (結構式設計模式) 只需要知道以下兩點的實作即可：
>
> 1. **封裝** (Encapsulate)
> 2. **注入** (Injection)
><br><br/>

因為結構式設計模式其實就是這兩者的運用。

而結構式設計模式共有七種：

||English|中文|
|:--:|:--|:--|
|1. |  [**Bridge**](#bridge_top)  |  橋樑模式 |
|2.   | **Adapter**  |   轉接器模式 |
|3.  |**Composite**  | 合成模式  |
|4.   | **Proxy**  |  代理模式  |
|5.   | **Decorator**   | 裝飾者模式  |
|6.   | **Flyweight**   | 蠅量級 / 享元 模式  |
|7.   |   **Facade** |  門面 / 外觀 / 表象設計模式 |

<br>

接下來，我們會運用以上的模式來編寫一個「帝國的創建」。


## 背景

想要創建一個帝國，先要條件就是有足夠的兵力。

目前我們所需要的有以下的兵種：

|角色|武器|交通工具|
|:--|:--|:--|
|步兵 (infantry)  | 手槍  | 腳  |
|槍兵 (gunner)   | 機槍  | 強力的腳  |
|騎兵 (rider)   | 雷射槍  | 機車  |
|偵察兵 (scout)   | 靜音槍 |  沈穩的腳 |

但是很顯然我們所需要的兵種並不止止這些。 而且就算有足夠的軍人了，我們還有很多的事務需要處理才能開始建立一個帝國。

所以接下來我們就看看要如何使用 **結構設計模式** 來建立一個帝國吧。 Enjoy。

**參考**
1. Alexey S., [Kotlin Design Patterns and Best Practices](https://www.amazon.com/Kotlin-Design-Patterns-Best-Practices/dp/1801815720), 2<sup>nd</sup> ed.,
<br>


### 增加兵種

想要新增兵種，我們需要先建立一個基礎類別：

```Kotlin
interface Solider {
    fun attack()
    fun walk()
}
```

<br>

而原本已有的兵種便會成為 **Solider** 的實作者：

```Kotlin
class Infantry: Solider {
    override fun attack() {
      // 進行攻擊
    }
    override fun walk() {
      // 進行行走
    }
}

```

<details>
<summary><b>其他的角色也是依樣畫葫蘆</b></summary>

```Kotlin
class Gunner: Solider {
    override fun attack() {
      // 進行計算
      return attackPoint;
    }
    override fun walk() {
      // 進行計算
      return speed;
    }
}


class Rider: Solider {
    override fun attack():Int {
      // 進行計算
      return attackPoint;
    }
    override fun walk(): Int {
      // 進行計算
      return speed;
    }
}


class Scout: Solider {
    override fun attack():Int {
      // 進行計算
      return attackPoint;
    }
    override fun walk(): Int {
      // 進行計算
      return speed;
    }
}

```

</details>

<br>

此時的 UML 為：

<center>
<img src="/images/posts/jekyll/design_patterns/structural/solider_before_bridge.png" style="width:80%"/>
</center>


#### 問題

><br>
>
> **Q : 如果今天需要新增不同的兵種時，你會如何進行？**
>
>一般來說，如果只是一兩個兵種，我們可能還是會像之前一樣。建立新的 **Solider** 類別並設定**特定**的實作。
>
>**Q : 但如果今天希望能自由創建兵種 ( 不同的武器、不同的交通工具 ) 呢？**
>
>這時我相信大多數人都會想到將 `attack()` 與 `walk()` 的實作注入進 **Solider** 中吧？
><br><br/>

這時我們就可以使用 **Bridge Design Pattern** 來進行設計了。

#### Bridge Design Pattern
<a id="bridge_top"></a>

><br>
>
>**Bridge** decouples an abstraction from its implementation so that the two can vary independently.
>
>**橋樑模式** 是將 **抽象** 與 **實作** 進行解藕，如此一來任ㄧ方的更改皆不會影響另一方。
><br><br/>

這裡我們要拆分的對象是 **Solider** 中的 `attack()` 與 `walk()` 的行為。


為了要實作`attack()` 與 `walk()` 的方法， 我們需要設計兩個介面或類別 **Weapon** 與 **Transportation**。


```Kotlin
// 設為 open 而非 sealed
// 是為了能隨時創建新的 Weapon 和 Transportation

open class Weapon(val action: ()->Unit) {
    // 實作 attack 的計算，目前只是簡單的回傳 Int
    class HandGun: Weapon({
      // handgun action
      })
    class MachineGun: Weapon({
      // machine gun action
      })
    class LaserGun: Weapon({
      // laser gun action
      })
    class Silencer: Weapon({
      // silencer action
      })
}

open class Transportation(val action: ()->Unit) {
    class Leg: Transportation({
        // travel with leg
      })
    class StrongLeg: Transportation({
        // travel with strong leg
      })
    class MotorCycle: Transportation({
        // travel with motor-cycle
      })
    class SteadyLeg: Transportation({
        // travel with steady leg
      })
}
```

再來就是重新設計 **Solider** ：

```Kotlin
data class CustomizableSolider(
  val weapon: Weapon,
  val transportation: Transportation): Solider {
    override fun attack() = weapon.action.invoke()
    override fun walk() = transportation.action.invoke()
}
```

如此一來， **Solider** 與 `attack()` 和 `walk()` 的行為也就被分開了。

而我們也可以隨時創造新的 **Weapon** 或 **Transportation** 並通過*注入*來創建新的兵種。

所以更新後，原本的兵種變成以下：

```Kotlin
val infantry = CustomizableSolider(
    Weapon.HandGun(),
    Transportation.Leg(),
  )
```

<details>
<summary><b>如此一來，之前的類別可以寫成以下</b></summary>

```Kotlin

val gunner = CustomizableSolider(
    Weapon.MachineGun(),
    Transportation.StrongLeg(),
)

val rider = CustomizableSolider(
    Weapon.LaserGun(),
    Transportation.MotorCycle(),
)

val scout = CustomizableSolider(
    Weapon.Silencer(),
    Transportation.SteadyLeg(),
)

```
</details>

<br>

我們甚至可以隨時創建客製化的 **Solider** ：

```Kotlin
val superSolider = CustomizableSolider(
    Weapon({
      // action
      }),
    Transportation({
      // walking
      }),
)
```


此時的 UML：

<center>
<img src="/images/posts/jekyll/design_patterns/structural/solider_after_bridge.png" style="width:80%"/>
</center>

<br>

雖說士兵們都繼承了 **CustomizableSolider**，但他們內部的實作都是由 **Weapon** 與 **Transportation** 定義。

所以我們也可以忽略士兵們，而專注於實作的部分：

<center>
<img src="/images/posts/jekyll/design_patterns/structural/solider_after_bridge_2.png" style="width:100%"/>
</center>

<br>

#### 結論

與一開始的 UML 相較之下，我們可以發現， `attack()` 與 `walk()` 的抽象與實作被分開了 ( **decoupled** )。 進而讓我們更有可塑性 ( **flexibility** )。

><br>
>
> **Bridge**
> **優點**： 增加可塑性、降低耦合性、符合開放閉關原則
> **缺點**： 增加抽象 / 介面，導致複雜性增加。
>
> 所以在實作 **Bridge** 時，我們需要衡量可塑性與複雜度之間的取捨。
><br><br/>


### 建立軍隊

擁有了士兵後，下一步我們需要建立一個軍隊。

一個軍隊的編制由小至大為： 伙、伍、班、排、連、營、團、旅 及 師。

而我們剛剛建立的 **Solider** 則是最小的單位，士兵。

當我們建立一個伙時，2 人，我們可以很容易地對他們進行調用。

```Kotlin
val soliderA = CustomizableSolider(Weapon.HandGun(), Transportation.MotorCycle())
val soliderB = CustomizableSolider(Weapon.HandGun(), Transportation.MotorCycle())

soliderA.attack()
soliderB.attack()
```

而當人數已經到一個班的時候，25-60 人，我們會使用 **for-loop** 的方法來調用他們。

```Kotlin
// 群體攻擊
soliders.forEach {
  it.attack()
}
```

#### 問題

><br>
>
>雖然我們可以用 for-loop 來對士兵們進行調用，但難道每次要調用時都需要使用 for-loop ?
>
>如果士兵需要進行分工呢？ 我們要如何知道哪些兵是一組的？ 而哪些是不需要被調動的？
>
>有沒有比較簡單的做法呢？
><br><br/>

當然有！ 我們可以將士兵編進不同的排。 而每次對士兵的調動我們只需要跟排長說一聲，他也就可以替我們轉達給士兵們。

<center>
<img src="/images/posts/jekyll/design_patterns/structural/troop_before_after_composite.png" style="width:80%"/>
</center>

所以我們大概就會作出以下設計：

```Kotlin
class Troop {
  val soliders = mutableListOf<CustomizableSolider>()

  fun attack() {
    soliders.forEach {
      it.attack()
    }
  }

  fun walk(): Int {
    soliders.forEach {
      it.walk()
    }
  }
}
```

如此一來，我們只需要如同調用 **Solider** 一樣地調用 **Troop** 了：

```Kotlin
solider.attack() // 一個士兵攻擊
troop.attack() // 一群士兵攻擊
```

**那你有發現了什麼嗎？**
... 沒錯？！ **Troop** 與 **Solider** 的行為其實都會是一樣的。 所以，我們可以讓 **Troop** 實作 **Solider** ：

```Kotlin
class Troop: Solider {
  val soliders = mutableListOf<CustomizableSolider>()

  override fun attack(): Int {
    // 同上
  }

  override fun walk(): Int {
    // 同上
  }
}
```
而這個設計就是所謂的 **Composite Design Pattern**。

是不是很神奇？

#### Composite Design Pattern

><br>
>
>**Composite** composes objects into tree structures to represent **part-whole** hierarchies.
It lets clients treat individual objects and compositions of objects **uniformly**.
>
>**組合模式** 會將物件以樹狀的方式來表示**部分與整體**結構的層次。
> 並讓客戶以**相同的方式**來使用物件單元與組合單元。
><br><br/>

根據剛剛所得到的結果，我們可以將 **Troop** 、 **CustomizableSolider** 與 **Solider** 的關係用以下 UML 表示：


<center>
<img src = "/images/posts/jekyll/design_patterns/structural/troop_composite_uml.png" style = "width:80%"/>
</center>

<br>

而更 **廣義** 的表達方式為：

<center>
<img src = "/images/posts/jekyll/design_patterns/structural/general_composite_uml.png" style = "width:80%"/>
</center>

<br>

經由 **Composite Design Pattern**， 使用者只需要針對 **Component** 的調用，而無需知道物件到底是 **Leaf** 還是 **Composite**。

#### 樹狀結構

**那你知道為什麼我們說 Composite 設計模式可以讓物件們以樹狀方式展現嗎？**

主要原因是因為 **Composite** 中的 `List<Leaf>` 未必只是一般的 **Leaf** 串列， 也有可能是一個 **Composite** 串列，甚至更複雜的結構。

><br>
>
>**範例：**
>假設我們不再是 **隊** ( 25-60 人 ) 而是一個 **營** ( 400-600 人 或 2-6 隊 ) 呢？
>
>那你會怎麼設計呢？
><br><br/>

最直覺的設計會跟之前的 **Troop** 一樣，用 Composite 包裹著 Leaf。

只是可能會將 **List\<Solider\>** 改成 **List\<Troop\>** ， 如同以下：

```Kotlin
class Battalion: Solider {
  val troops = mutableListOf<Troop>()

  override fun attack(): Int {
    // ...
  }

  override fun walk(): Int {
    // ...
  }
}
```

但是這樣的設計方法除了沒有好好地發揮 **Composite Design Pattern** 的優點，還會限制 **Battalion** 的功能。

假設每個連都會有額外兩名後勤士兵，那你是否會直接新增兩名士兵呢？

```Kotlin
class Battalion: Solider {
  val troops = mutableListOf<Troop>()

  // 新增士兵 A
  val soliderA = CustomizableSolider(
      Weapon({
        // action
      })
      Transportation({
        // travel
      })
    )

    // 新增士兵 B
    val soliderB = CustomizableSolider(
        Weapon({
          // action
        })
        Transportation({
          // travel
        })
      )

    override fun attack(): Int {
      // ...
    }

    override fun walk(): Int {
      // ...
    }

}
```

很顯然這個做法會導致難以維護或更新，也違背的 Open/Close Principle。 因為每次要新增士兵時，我們都需要進來修改。

**那要如何改善呢？**

其實竟然都是實作 **Solider**，那為何要限制 **Battalion** 中只能放 **Troop** 呢？

我們可以直接把 **Troop** 也變成 **Solider** 即可：

```Kotlin
class Battalion: Solider {
  val soliders = mutableListOf<CustomizableSolider>()

  override fun attack(): Int {
    // ...
  }

  override fun walk(): Int {
    // ...
  }
}
```

如此一來，無論我們需要新增多少的士兵 或 隊，我們都可以將他們全部加入 `soliders` 中。

而 **Battalion** 的架構可以用以下圖表表示：


<center>
<img src = "/images/posts/jekyll/design_patterns/structural/battalion_compose_uml.png" style = "width:80%"/>
</center>

<br>

現在看出為什麼説 **Composite Design Pattern** 可以讓物件以樹狀架構顯示出來了吧？


### 創建帝國

恭喜各位！！我們終於可以開始 **建立帝國** 了。

一個國家的安定並不能只依賴強大的軍事，更需要團結的民心。而民心需要安穩的生活環境來培養。為了要建立一個安穩的環境，君主需要解決大量的問題，從 **農務**、**法律**、**商貿** 到 **國防**，真的是業務龐大的工作啊。

當然，為了減輕君主的業務，他必須要建立多個獨立的部門或單位，包括：

- 國防
- 法部
- 農部
- 商部
- 外交

等等。

理想狀態下，這些部門與君主會有獨自的溝通管道，讓他們可以各自執行君主的要求：

<center>
<img src = "/images/posts/jekyll/design_patterns/structural/facade_before_country_structure.png" style = "width:80%"/>
</center>

<br>

但是這只是奢望，畢竟這些部門有太多錯綜復雜的關係了。單舉一個法律的建立就是一個跨部門的議題了。

所以一般來說我們會看到以下的情況：

<center>
<img src = "/images/posts/jekyll/design_patterns/structural/facade_before_country_function.png" style = "width:80%"/>
</center>

<br>

看到這你應該很難想像只使用一個君主 **Ruler** 類別來實作吧。


#### 問題

><br>
>
> 如果是你，你會如何 **簡化** 這個架構 呢？
><br><br/>

我想大家的第一個反應就是，如果我們可以定義出特定的行為，那不就可以定義特定的 API 了？

這樣就可以將這些複雜的行為都 **遮掩起來** 了!!!
( 這不就是 **封裝** ？)

**沒錯喔，而這便是 Facade Design Pattern 的作用了**

經過 **Facade** 模式，我們可以將上面的架構變為：

<center>
<img src = "/images/posts/jekyll/design_patterns/structural/facade_after_country_function.png" style = "width:80%"/>
</center>

<br>

而 **Facade** 則需要實作以下介面：

```Kotlin
interface RulerBehavior {
  fun establishLaw()
  fun establishRelationshipWithCountry()
  fun performAgriculturalTrading()
}
```

如此一來， **Ruler** 從原本需要通過不同部門的 API 來進行不同的功能，到現在只需要通過簡潔易懂的 API 達到相同的效果。

但是，我們依舊需要實作 **RulerBehavior** ：

```Kotlin
data class Ruler(
  val judicial: Judicial,
  val foreign:ForeignMinistry,
  val trading: TradingDepartment,
  val agriculture: AgriculturalTrading,
  val military: Military ): RulerBehavior {

    override
    fun establishLaw() {
      // do something
      println("EstablishLaw Done")
    }

    override
    fun establishRelationshipWithCountry() {
      // do something
      println("EstablishRelationshipWithCountry Done")
    }

    override
    fun performAgriculturalTrading() {
      // do something
      println("AgriculturalTrading Done")
    }

}

```

#### Facade Design Pattern

><br>
>
>**Facade** provides a **unified interface** to a set of interfaces in a **subsystem**.
>
>It defines a **higher-level interface** that **makes the subsystem easier to use**.
>
>**表面設計模式** 會將子系統的介面們整合成一個 **整合介面** 。
>
>通過 **整合介面** 使用者便可更簡單地調用子系統。
>
><br><br/>

以下便是 **Gang of Four** 的 **Design Patterns** 中用來表示 **Facade** 作用的圖示：

<center>
<img src = "/images/posts/jekyll/design_patterns/structural/facade_gof.png" style = "width:100%"/>

Facade: Design Patterns by Gang of Four
</center>

<br>

#### 題外話

我在寫這個設計模式時，我一直將 **Facade** 與 **Proxy** 搞混。 不過這些就等全部結構模式談完再談吧。

這裡先給個提示：
**Facade**、 **Proxy** 與 **Adapter** 的差異主要來自他們的 **介面**、**實作** 與 **作用**。

#### 參考

1. "**Facade**", Design Patterns: Elements of Reusable Object-Oriented Software, GOF.

### 招募人才

經過部門及溝通橋樑的建立後，帝國的運作也漸入佳境。

但就算有想法，還是需要有人去執行才能實現。 所以廢話不多說，我們需要招募人才了。

想要招募人才，第一件事當然就是設計宣傳單啦 ~~~

```Kotlin
data class Leaflet (
    val title: String,
    val description: String,
    val catchPhrase: String,
    val eyeCatchingArt: Bitmap,
    val extra: Map <String: Any?>
  )
```

有了傳單，我們就可以 **大肆發放** 。

<center>
<img src = "/images/posts/jekyll/design_patterns/structural/flyweight_before_leaflet.png" style = "width:80%"/>

</center>

<br>

假設我們需要一千張傳單，我們可以寫成：

```Kotlin
val people = mutableListOf<People>(
    // list of people data
  )

people.forEach {
  it.get(
    Leaflet(
      "Title", "Description", "Catch Phrase",
      BitmapFactory.decodeStream(url.openConnection().getInputStream()),
      extra = mapOf()
    ))
}

```
恩，看起來有點怪怪的，你們有發現什麼嗎？

#### 問題

><br>
>
>假設一張傳單的手工製作需要 1 小時，那如果我希望在一天內製作上百張甚至上千張的傳單時，我需要多少 **人力** 呢？
>
> 如果人手不足，那我們要 **如何用最少的人力得到最大的效果** 呢？
><br><br/>

經過這兩個問題，我相信大家都想到這裡的 「**人力**」  指的就是電腦中的 「**記憶體**」。

而在創建 **Leaflet** 時，由於不斷地建立相同的 **Bitmap**，**記憶體** 也會逐步地增加。

**為了要解決這個問題，我們該怎麼做呢？**

對於傳單而言，如果我們希望用最少的人力並得到最高的效率，我們其實可以使用 **公告板**。 讓全部的人都可以**到同一個地方來讀取相同的資訊**。

假設我們所需的 **Bitmap** 是固定的，我們就可以用固定數量的 **Bitmap** 物件來存放指定的資料。而且我們也只需要建立以下：

```Kotlin
val people = mutableListOf<People>(
    // list of people data
  )

val bitmap = BitmapFactory.decodeStream(url.openConnection().getInputStream())

val leaflet = Leaflet(
    "Title", "Description", "Catch Phrase",
    bitmap,extra = mapOf())

people.forEach {
  it.read(leaflet)
}

```

這樣，全部的人都可以閱讀相同的 **Leaflet** 但 **所需要的記憶體卻是大大下降**。

當然，更好的作法便是使用 cache 來存放製作出來的 Bitmap 並在需要的時候才會產生 / 回傳所需的 Bitmap。

而此優化方法便是 **Flyweight Design Pattern** 的主要作用了。

#### Flyweight Design Pattern

><br>
>
> **Flyweight** uses **sharing** to support large numbers of fine-grained objects efficiently.
>
> **Flyweight** 顧名思義，就是讓重量飛走，也就是減輕重量的設計模式。 而他會通過物件 **共享** 來減輕記憶體的負擔，因此中文稱之為「享元模式」。
><br><br/>

通過 **Flyweight** 的運用，我們順利地將以下的架構

<center>
<img src = "/images/posts/jekyll/design_patterns/structural/flyweight_before_leaflet.png" style = "width:80%"/>

</center>

<br>

轉成以下架構：

<center>
<img src = "/images/posts/jekyll/design_patterns/structural/flyweight_after_leaflet.png" style = "width:80%"/>

</center>

<br>

暸解了嗎？

#### 參考

1. "**Flyweight**", Design Patterns: Elements of Reusable Object-Oriented Software, GOF.


### 人才的養成

雖然現在我們已經擁有人力了，但是他們依舊缺乏相關的知識。 因此，我們需要開始針對性的培養。

首先，我們先定義這些人才的行為吧!

```Kotlin
interface StaffBehavior {
  fun doSomething()
}
```

接下來，我們需要將他們分配至不同部門來學習。
所以，最直接的方法便是使用繼承：

```Kotlin
// 舉幾個例子
class JudicialStaff: StaffBehavior {
  override
  fun doSomething() {
    println("我懂法部")
  }
}

class MilitaryStaff: StaffBehavior {
  override
  fun doSomething() {
    println("我懂國防")
  }
}

class AgriculturalStaff: StaffBehavior {
  override
  fun doSomething() {
    println("我懂農務")
  }
}

// 以此類推

```

這個設計會形成以下 UML ：

<center>
<img src = "/images/posts/jekyll/design_patterns/structural/decoration_before_structure.png" style = "width:80%"/>

</center>

<br>

但還沒完，因為部門之間的作業多少互有關聯。 所以一般的時候，員工會發放到不同的部門來觀摩與實習。

如果是如此，想必人員的 `doSomething` 也需要不斷地擴張才行。

><br>
>
> 如果是你，你會如何設計呢？
>
>**切記**：我們需要做的不僅僅增加員工的 `doSomething` 功能，還需要配合隨時 **新增的部門**。
> <br><br/>

我們有以下的選擇：

繼承
--
雖然說繼承可以解決我們的問題，但我們卻需要將所有的可能性都實作出來。

若每位員工都有兩項技能，那我們就需要新增 10 個類別：

||農部|法部|商部|外交|國防|
|:--:|:--:|:--:|:--:|:--:|:--:|
| 農部  | x  | 農法  | 農商  | 農外  | 農防  |
|法部   |  -- | x  |  法商 | 法外  | 法防  |
|商部   |  -- | --  | x  | 商外  | 商防  |
|外交   | --  |--   |--   | x  | 外防  |
|國防   | --  | --  | --  |  -- | x  |

如果全部都要懂的話，就會有：

$$(5 + \frac{5 \times 4} {2} + \frac{5 \times 4 \times 3} {3} + \frac{5 \times 4 \times 3 \times 2}{4} + 1) = 66 種類別$$

可想而知，這是不可行的。

橋樑模式
--

那如果用 **Bridge** 呢？

我們也可以對 **StaffBehavior** 進行不同的實作並注入一個 **Staff** 類別中。

但我們會面對與繼承相同的問題，因為也是需要實作 **66** 種。

看來以上兩種方法都不符合我們的需求。

#### 問題

><br>
>
> 有沒有辦法在在不大量實作的前提下，讓員工都擁有想要的行為呢？
> <br><br/>

我們回想一下，為什麼這問題可以用 **繼承** 來解決呢？
不就是因為了可以在新增行為的同時，保留原本的行為嗎？

```Kotlin
open class Staff: StaffBehavior {
  override
  fun doSomething() {
    // do nothing
  }
}

open class JudicialStaff: Staff {
  override
  fun doSomething() {
    println("我會法部")
  }
}

class JudicialAgriculturalStaff: JudicialStaff() {
  override
  fun doSomething() {
    super.doSomething()
    println("我會農部")
  }
}

// 當我們調用
val jaStaff = JudicialAgriculturalStaff()
jaStaff.doSomething()

// 結果：
// 我會法部
// 我會農部
```

**那竟然目的是保留原本的行為，有沒有可能是用物件的方式保留呢？**

這個需求可以用 **注入** 來解決：

```Kotlin
class Staff: StaffBehavior {
  override
  fun doSomething() {
    // do nothing
  }
}

interface StaffDecorator: StaffBehavior {
  val staff: StaffBehavior
}

// 注入 staff
class JudicialStaff(
  override val staff: StaffBehavior
  ): StaffDecorator {

  override
  fun doSomething() {
    staff.doSomething()
    println("我會法部")
  }
}

// 注入 staff
class AgriculturalStaff(
  override val staff: StaffBehavior
  ): StaffDecorator {

  override
  fun doSomething() {
    staff.doSomething()
    println("我會農部")
  }
}

val jaStaff = JudicialStaff(AgriculturalStaff(Staff()))

jaStaff.doSomething() // 得到相同結果
```

這個設計便稱為 **Decorator Design Pattern**。

#### Decorator Design Pattern

><br>
>
>**Decorator** attaches additional responsibilities to an object **dynamically** . It provides a flexible alternative to subclassing for **extending functionality** .
>
>**裝飾模式** 可以 **動態地** 新增物件的行為或職責。 他也是使用繼承來為 **延伸或擴張方法** 的第二方案。
><br><br/>

根據上面提出的程式碼，我們可以用以下 UML 來表示：

<center>
<img src = "/images/posts/jekyll/design_patterns/structural/decorator_after_structure.png" style = "width:90%"/>

</center>

<br>

雖然說這個模式可以大幅下降複雜度，但卻有個致命的問題。

回想一下我們這個類別主要需要做的是什麼？

我們希望能培育人才來做某些部門的事情。

**那問題來了：**

```Kotlin
val someStaff: StaffBehavior = ....
```

你能告訴我這個 `someStaff` 會些什麼嗎？

#### 表象的缺點

很顯然，你是無法從表象看出 `someStaff` 會些什麼。
這是因為 **Decorator** 就像一個 Wrapper 一樣，將 **Staff** 一層一層地包裹起來。

而 `jaStaff` 就可以用以下圖示來表達這種層疊關係：

<center>
<img src = "/images/posts/jekyll/design_patterns/structural/decorator_layer.png" style = "width:90%"/>

</center>

<br>

所以如果我們查看 `jaStaff` 的類別就會有以下結果：

```Kotlin
jaStaff is StaffBehavior // true
jaStaff is JudicialStaff // true
jaStaff is StaffDecorator // true

jaStaff is AgriculturalStaff // false
jaStaff is Staff // false
```

這是因為對外界而言，他們只會看到 **JudicialStaff** 和 他所繼承的類別。

**那我們有沒有辦法知道 Staff 到底能做什麼呢？**

答案是可以的，我們可以修改一下 **StaffDecorator** ：

```Kotlin
// 改成 abstract class
abstract class StaffDecorator: StaffBehavior {
    abstract val staff: StaffBehavior

    // 新增 isType 來做類別的檢查
    fun isType(clazz: Class<*>): Boolean {
        if (clazz.isInstance(this)) {
            return true
        } else {
            if (staff is StaffDecorator) {
                return (staff as StaffDecorator).isType(clazz)
            } else {
                return clazz.isInstance(staff)
            }
        }
    }
}

```

經過修改後，我們重新再測試一次就可得到不同結果了：

```Kotlin
jaStaff is StaffBehavior // true
jaStaff is JudicialStaff // true
jaStaff is StaffDecorator // true

jaStaff is AgriculturalStaff // true
jaStaff is Staff // true
```

所以在使用 **Decorator** 時要留意這點。

### 代理人的課題

擁有人力與人才後，為了以防萬一，我們需要開始提拔一位 **君主的代理人**。

為了培養一位合格的代理人，我們必須要教導這位未來代理人 ， **ProxyRuler**， 如何執行不同的事項。

#### 問題

><br>
>
> 那你覺得一位君主的代理人 **需要會做什麼** 呢？
><br><br/>

很顯然，君主的代理人必須會做所有君主所會做的事。
所以我們會重複使用 **RulerBehavior** ：

```Kotlin
// 這與君主所用的一樣
interface RulerBehavior {
  fun establishLaw()
  fun establishRelationshipWithCountry()
  fun performAgriculturalTrading()
}
```

下一個問題：**ProxyRuler 要如何實作** ？

難道跟 **Ruler** 一樣？

自然不是，因為代理人必須要知道如何跟 **被代理者**，也就是君主，回報。

所以 **ProxyRuler** 的設計會如下：

```Kotlin
data class ProxyRuler(val ruler: Ruler): RulerBehavior {
  override
  fun establishLaw() {
    // do something
    println("Message Sent")Â
    ruler.establishLaw()
  }

  override
  fun establishRelationshipWithCountry() {
    // do something
    println("Message Sent")
    ruler.establishRelationshipWithCountry()
  }

  override
  fun performAgriculturalTrading() {
    // do something
    println("Message Sent")
    ruler.performAgriculturalTrading()
  }
}
```

如此一來，當君主不在時，我們就可以跟 **ProxyRuler** 説，並由他來告知 **Ruler** ：

```Kotlin
val proxy = ProxyRuler(ruler)

proxy.establishLaw()
```

這個設計便是 **Proxy Design Pattern**。

#### Proxy Design Pattern

><br>
>
>**Proxy** provides a surrogate or placeholder for another object to control access to it.
>
>**代理模式** 提供了 **代理人** 或 **保留區** 來讓其他物件來調用。
><br><br/>


我們可以用 UML 來展現 **Ruler**、 **RulerBehavior** 與 **ProxyRuler** 間的關係：

<center>
<img src = "/images/posts/jekyll/design_patterns/structural/proxy_result_uml.png" style = "width:100%"/>

</center>

<br>

#### Proxy 的使用方法

到目前為止，也許你還在想 **Proxy** 難道只是用來調用 `println` 嗎？

這當然不是， 不過 **Proxy** 的主要功能就是在調用對象 (real object) 前後 **新增行為** 。 而這些行為包括：

|Type|Description|
|:--|:--|
| **Smart Reference**  | 記錄使用對象的數量 (reference counter)  |
| **Remote** Proxy  |  **隱藏** 對象的所在位置。<br>譬如資料的所在位置是 **遠端** 還是 **本地端**。 |
|  **Virtual** proxy |提供一個即刻的資料，儘管真實資料尚未取得。 <br>舉例： placeholder   |
| **Protection** proxy  |用來 **限制** 可調用對象的目標。 <br>舉例：在取得資料前，先檢查使用者是否登入。   |

#### 參考

1. "**Proxy**", Design Patterns: Elements of Reusable Object-Oriented Software, GOF.
2. iThome: [Design Pattern: Proxy 代理模式](https://ithelp.ithome.com.tw/articles/10222056)


### 建交之旅

目前帝國裡面的運作似乎也上了軌道，此時我們就需要開始與他國進行建交了。

建交最大的問題就是 **語言** 與 **文化** 。

文化可以花時間去研究與暸解，但如果無法使用當地語言那就什麼都做不了了。 因此建交時，官員都會帶上一位 **翻譯人員** 或 **外交官** ( **Translator** ) 來幫忙翻譯。

而我們所說的語言是中文：

```Kotlin
interface Chinese {
  fun hallo()
  fun woHenHaoXieXie()
  fun niHaoMa()
}

```

當官員要進行對話時，他便可以跟 **ChineseTranslator** 説一聲：

```Kotlin

class OfficialPersonel {
  var translator: ChineseTranslator = ChineseTranslator()

  fun makeConversation() {
    translator.hallo()
    translator.niHaoMa()
    translator.woHenHaoXieXie()
  }
}

```

而這則是我們所不懂的語言：

```Kotlin
interface French() {
  fun bonjour()
  fun jeVaisBienMerci()
  fun commentFaitesVous()
}
```

#### 問題

><br>
>
> 由於我們只會講中文，那我們又該如何用 **French** 溝通呢？
><br><br/>

你有可能會想説，啊直接實作一個 **French** 不就好了。
但是別忘了，我們不會講 **French**，所以無法調用 **French** 的行為。

竟然我們只會講 **Chinese**，那有沒有可能由 **ChineseTranslator** 將 **French** 包裹起來，並在實作時調用 **French** 呢？ 就像這樣：

```Kotlin
class ChineseTranslator(val french: French): Chinese {

  override
  fun hallo() {
    french.bonjour()
  }

  override
  fun woHenHaoXieXie() {
    french.jeVaisBienMerci()
  }

  override
  fun niHaoMa() {
    french.commentFaitesVous()
  }
}
```

恩，沒毛病。
我們成功地經由 **ChineseTranslator** 調用 **French** 的行為了。
所以我們也可以說法語了。

如果我們用圖來表示 **ChineseTranslator**、 **French** 與 **Client** 的關係，那就大概長這樣：

<center>
<img src = "/images/posts/jekyll/design_patterns/structural/adapter_flow.png" style = "width:100%"/>

</center>

<br>

在這， **ChineseTranslator** 可被稱作為 **Adapter**。

因為他的行為是將 **French** 介面 **轉換成** Client 所熟悉的 **Chinese** 介面。

而這也就是所謂的 **Adapter Design Pattern** 。


#### Adapter Design Pattern

><br>
>
>**Adapter** **converts** the interface of a class into another interface clients expect.
It **lets classes work together** that couldn't otherwise because of **incompatible** interfaces.
>
> **轉接器模式** 會將 目標介面 **轉換成** 客戶端所使用的介面。
  他會讓兩個 **不同行為** 的類別一同合作。
><br><br/>

如果我們將上面的結果畫成 UML，那就會是如此：

<center>
<img src = "/images/posts/jekyll/design_patterns/structural/adapter_after_translator.png" style = "width:100%"/>

</center>

<br>

我們也可以將他畫成更廣義的樣子：

<center>
<img src = "/images/posts/jekyll/design_patterns/structural/adapter_general_translator.png" style = "width:100%"/>

</center>

<br>

使用 **Adapter** 的好處就是可以在 **遵守開閉原則** 的前提進行介面的轉換。

不過，如果 **進行內部修改會** 比 **建立新的介面並重新實作** 更有效率時，還是考慮內部修改即可。

#### 參考

1. "**Adapter**", Design Patterns: Elements of Reusable Object-Oriented Software, GOF.
2. Refactoring Guru: [Adapter](https://refactoring.guru/design-patterns/adapter)

### 下一篇章

以上就是結構模式的範例，接下來我們來在 Android 源碼中有哪些地方使用了哪種模式吧。

### Android 中的 Structural Design Patterns

#### Bridge


#### Facade

#### Flyweight

#### Decorator

#### Adapter

#### Proxy


### 補充

#### Proxy vs Adapter vs Facade

在寫這篇的時候，我一直覺得這三者感覺是一樣的東西。






<br><br><br><br><br><br><br>
