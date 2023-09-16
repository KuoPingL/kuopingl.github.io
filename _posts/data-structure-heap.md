---
layout: post
title:  "Data Structure - Heap"
date:   2022-08-08 16:15:07 +0800
categories: [data structure, tree, beginner]
---

### What is a Heap ?
><br>A **Heap** is a special **Tree-based** data structure in which the tree is a **complete binary tree**. -- [geeksforgeeks](https://www.geeksforgeeks.org/heap-data-structure/)
>
> A **complete binary tree** is a special type of binary tree where **all the levels** of the tree are **filled completely** *except* **the lowest level nodes** which are *filled from as left as possible*. -- [geeksforgeeks](https://www.geeksforgeeks.org/complete-binary-tree/)
><br><br/>

<figure>
<center>
<img src = "/images/posts/jekyll/algorithms/data_structure_heap.jpg" style="width:80%"/>
<br>Complete Binary Tree
</center>
</figure>

### Properties
In order for a **complete binary tree** to become a **Heap data structure**, it needs to fulfill one of the two **heap properties** :

- **Max Heap Property**
- **Min Heap Property**


#### Max Heap Property
A **Max Heap Property** follows the following rules :

1. Any given node is **always greater** than its child node/s.
2. The **key of the root node** is the **largest among all other nodes**.

  <figure>
  <center>
  <img src = "/images/posts/jekyll/data_structure/heap_maxheap.png" style="width:80%"/>
  <br>max heap [ <a href="https://www.programiz.com/dsa/heap-data-structure">ref</a> ]
  </center>
  </figure>

#### Min Heap Property
A **Min Heap Property** follows the following rules :

1. Any given node is **always smaller** than the child node/s
2. The **key of the root node** is the **smallest among all other nodes**.

  <figure>
  <center>
  <img src = "/images/posts/jekyll/data_structure/heap_minheap.png" style="width:80%"/>
  <br>min heap [ <a href="https://www.programiz.com/dsa/heap-data-structure">ref</a> ]
  </center>
  </figure>

These structures are also called a **Binary Heap**.

To create a **Heap**, we need to go through a process called **Heapify**.

### Heapify
To **Heapify** an array or other data structures, even from other non-heap binary tree, we need to make sure that :

1. All Nodes are always inserted from **Top to Bottom** and from **Left to Right**.
2. If a binary tree is created, all nodes need to follow the selected heap properties.


><br>To illustrate how **heapify** works, let's see how we can create a heap from this array **[ 3, 9, 2, 1, 4, 5 ]**
><br><br/>


1. Create a **Complete Binary Tree**
   This is simply done by following the order of the array :

   ```
               3
          ┌----┴----┐
          9         2
        ┌-┴-┐     ┌-┴-┐
        2   1     4   5
   ```
   However, this does not fulfill the **min** or **max** heap properties.

2. Fulfill the **max heap property**
   Just to make it easier for us to implement, we will use the array directly to show what is happening.

   - we will start by sorting the bottom left subtree
     [ 3, **9**, 2, **2**, **1**, 4, 5 ]
     This is correct, since the parent ( **9** ) is greater than children 





<br><br><br><br><br>
