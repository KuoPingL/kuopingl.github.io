---
layout: post
title: "OOP Design - SOLID Principle"
date: 2023-11-07 21:22:20  +0800
categories: [Software Design]
keywords: beginner, solid
---

# 序言

物件導向程式設計 (OOP) 已經成了大家耳熟能響的用詞了。 但大家對他該如何設計又懂多少呢？

這篇我們將探討 **Robert C. Martin** (aka Uncle Bob) 所提出的經典之規範， SOLID。

之所以程式員都應該跟從這個規範是為了讓程式更有以下優點：

- 代碼的重複性利用
- 可擴張性的提升
- 代碼可讀性的增加
- 降低重構的成本 等等

我們將會一一說明 SOLID 的優點以及如何實作他們。

# SOLID

SOLID 指的是五個主要規範 :

| Abbreviation | Acronym | Meaning                         | Description                                                                                                                             |
| :----------: | :------ | :------------------------------ | :-------------------------------------------------------------------------------------------------------------------------------------- |
|      S       | SRP     | Single Responsibility Principle | 每個類別只能有單一職責。                                                                                                                |
|      O       | OCP     | Open-Close Principle            | 程式設計中，不管是模組、類別、方法、等等，都應該可被延展，但也應當不被修改。                                                            |
|      L       | LSP     | Liskov Subsitution Principle    | 程式中，父類別可以由子類別替換，並不會造成任何未知的影響。                                                                              |
|      I       | ISP     | Interface Segration Principle   | 在程式設計中，類別不應當存在他所不需要的方法。 因此，我們可以用介面或抽象類別將方法按職責分開來。並只實作所需要的介面即可。             |
|      D       | DIP     | Dependency Inversion Principle  | 1. 高層次的模組不應該依賴於低層次的模組，兩者都應該依賴於抽象介面。<br>2.抽象介面不應該依賴於具體實現。而具體實現則應該依賴於抽象介面。 |

# SRP

# OCP

# LCP

# ISP

# DIP

<br><br><br><br><br><br><br><br><br><br><br><br><br>
