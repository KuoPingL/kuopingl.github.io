---
layout: post
title: "Design Patterns - General Walkthrough"
date: 2023-11-07 21:22:20  +0800
categories: [Software Design]
keywords: beginner, design pattern
---

# 序言

當我們在寫程式的時候，為了優化整個程式的可擴張性、可讀性、降低耦合性和降低重構的成本，我們會將類別或型別之間的關係進行特別的設計。 而這種設計就是所謂的「設計模式」。

但為了讓程式員能更容易暸解及解釋目前所使用的設計模式， **Gang of Four** 將他們遇到過的設計模式歸類成 三大類別，共 23 種設計模式。 這篇我會搭配 GOF 的 [Design Patterns](https://www.javier8a.com/itc/bd1/articulo.pdf) 與大話設計模式一同說明。希望這篇對讀者們有所幫助。

> 「大話程式模式」中也將 SOLID 包括其中。 雖然很合理，但我還是將 SOLID 放在[另一篇]()獨立說明。

另外，我們還可以將他們通過他們用在的地方進行分類 (GOF, table1.1)：

| Category            | Design Patterns    |                                                                                                                                                         Scope |
| :------------------ | :----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------: |
| Creational Patterns | [Factory Method]() |                                                                                                                                                         Class |
|                     | Object             |                                                                                      [Abstract Factory]() <br>[Builder]() <br>[Prototype]() <br>[Singleton]() |
| Structural Patterns | Class              |                                                                                                                                                   [Adapter]() |
|                     | Object             |                                        [Adapter]() <br>[Bridge]() <br>[Composite]() <br>[Decorator]() <br>[Facade 外觀模式]() <br>[Flyweight]() <br>[Proxy]() |
| Behavioral Patterns | Class              |                                                                                                                       [Interpreter]() <br>[Template Method]() |
|                     | Object             | [Chain of Responsibility]() <br>[Command]() <br>[Iterator]() <br>[Mediator]() <br>[Memento]() <br>[Observer]() <br>[State]() <br>[Strategy]() <br>[Visitor]() |

以下是 GOF 提供的設計模式關係圖：

<center>
    <img src = "/images/posts/jekyll/design_patterns/gof_fig_1.1_dp_relationship.png" style="width:70%"/>
</center>

我將從對底部的 **Facade** 說起，並跟隨路線導向其他的設計模式。 希望這種說明方式可以讓大家更容易暸解。

# Facde (外觀模式)

|觀點|說明|
|類別|Behavioral|
|Scope|Object|
|技術|Facade 只是一個 OOP **封裝** 的運用。|
|優點|增加可塑性<br>減少耦合性<br>增加隱蔽性|

假設我們有一台機器，我們可以通過 `start()` 與 `stop()` 來進行操控 :

<center>
    <img src = "/images/posts/jekyll/design_patterns/dp_facade_before_0.png" style="width:70%"/>
</center>

這樣的設計似乎沒有問題。 但每當我們有新的機器需求時，我們都需要重新創建一個新的機器。而且機器的調用方法還可能會有所不同 :

<center>
    <img src = "/images/posts/jekyll/design_patterns/dp_facade_before_1.png" style="width:70%"/>
</center>

但我們知道的是，無論怎麼改變，我們都可以將他們的方法視作 `start()` 與 `stop()` 的功能。

因此，我們可以創建一個介面，並通過封裝的方式將這些方法統一化：

<center>
    <img src = "/images/posts/jekyll/design_patterns/dp_facade_after_1.png" style="width:70%"/>
</center>

如此一來，使用者只需要知道 `start()` 與 `stop()` 方法，並不需要在意新機器的操控方式。而這個新的介面就是所謂的 **Facade**。

由於每一台機器都會有不同的運作方式，所以我們還可以把機器看作是獨立的 subs-system。 這樣你應該就可以看出 Facade 的價值了。

最常使用的地方就是當我們創建函式庫時，我們只會提供 API 讓使用者調用，並盡量將細節隱藏起來，不讓他人隨意更改。

另一個範例就是演算法的運用。 演算法也是只需要 input 再通過我們不知道的方式得到 output，就如同 Facade 一樣：

<center>
    <img src = "/images/posts/jekyll/design_patterns/dp_facade_sample_algo.png" style="width:70%"/>
</center>

若談到 Android，那我們就可以用 **Context** 當作範例了。 Context 本身是一個抽象類別。 但通過 **ContextImpl** 的實作，我們可以在不知情的情況下與不同的 Manager 互動。

<center>
    <img src = "/images/posts/jekyll/design_patterns/dp_facade_android_context.png" style="width:70%"/>
</center>

# Abstract Factory

# Singleton

|觀點|說明|
|技術|Singleton 與 Facade 差不多，但不同的地方在於|
|優點||
|缺點||
|注意事項||

# OTHERS

As a result with a lot of discussion with various IoC advocates we settled on the name Dependency Injection. [src](https://martinfowler.com/articles/injection.html)

If you read about this stuff in the current discussions about Inversion of Control you'll hear these referred to as type 1 IoC (interface injection), type 2 IoC (setter injection) and type 3 IoC (constructor injection).

<br><br><br><br><br><br><br><br><br><br><br><br><br>
