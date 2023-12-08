---
layout: post
title: "Algorithm : Search"
date: 11/07/23 21:22:20  +0800
use_math: true
categories: [Algorithm Topics]
keywords: algorithm
---

# 序言

[搜尋演算法](https://en.wikipedia.org/wiki/Search_algorithm) 的運用其實無所不在的。從資料的搜尋、最短路線的尋找、 破解數獨 到 AI 領域的運用都離不開搜尋演算法。這篇文章中我們將會在探討不同的 [搜尋演算法](https://en.wikipedia.org/wiki/Search_algorithm) 以及他們的運用。

在寫這篇文章前，我一直認為搜尋演算法除了線性搜尋演算法外都需要使排列好的資料，但我這是錯得離譜。搜尋演算法其實有很多種，幸好我們可以通過演算法的機制將他們大致分成三種類 :

| 機制        | 搜尋演算法                                                   | Space Complexity                                          |                         Data Sorted                         | Data Size |
| :---------- | :----------------------------------------------------------- | :-------------------------------------------------------- | :---------------------------------------------------------: | :-------: |
| **Linear**  | Linear Search<br>BFS<br>DFS                                  | Best : $O(n)$<br>Avg : $O(n)$<br>Worst : $O(n)$           |                            false                            |    小     |
| **Binary**  | Binary Search<br>Exponential Search<br>Quantum Binary Search | Best : $O(1)$<br>Avg : $O(log(n))$<br>Worst : $O(log(n))$ | 當我們知道資料 **已排序好**，我們就可以使用 Binary 演算法。 |   true    | 皆可 |
| **Hashing** | Hashing Search                                               | Best : $O(1)$<br>Avg : $O(1)$<br>Worst : $O(n)$           |                            false                            |    大     |

接下來我們會針對這幾種機制一一說明 。

# Linear

# Binary

# Hashing

# 補充資料

1. [Wiki: 搜尋演算法](https://en.wikipedia.org/wiki/Search_algorithm)
2. [Wiki: 排列演算法](https://en.wikipedia.org/wiki/Sorting_algorithm)
3. [Searching and Hashing](https://learn.saylor.org/mod/book/tool/print/index.php?id=32990)
