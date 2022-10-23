---
layout: post
title:  "Data Structure - Binary Tree"
date:   2022-08-08 16:15:07 +0800
categories: [data structure, tree, beginner]
---
### What's a Binary Tree ?
><br>**Binary Tree** is defined as a **Tree data structure** with **at most 2 children** that looks like the figure below. -- [geeksforgeeks](https://www.geeksforgeeks.org/binary-tree-data-structure/)
><br>

<figure>
<center>
<a href = "https://web.ntnu.edu.tw/~algo/BinaryTree.html">
<img src = "/images/posts/jekyll/data_structure/binary_tree_simple.png" style="width:80%"/></a>
</center>
</figure>

#### Anatomy
><br>To help understand algorithms related to binary tree, it is important for us to learn some of the terms related to binary tree.
><br>

A Binary Tree is composed of **nodes** and **branches**.
- the **node on the top** is called **root**, and
- **nodes at the bottom** are called **leaf**.

A **node** can play a role as :
- **parent** : if more nodes are connected to this node at lower level.
- **sibling** : nodes that are connected to the **same parent**, they are sibling of one another.
- **child** : nodes that are connected to a **parent** is called a **child**, and depending on its orientation, it can eitehr be **left** or **right** child.

<figure>
<center>
<a href = "https://web.ntnu.edu.tw/~algo/BinaryTree.html">
<img src = "/images/posts/jekyll/data_structure/binary_tree_anatomy.png"/></a>
</center>
</figure>


#### Types of Binary Tree
><br>Depending on the final structure of the binary tree, it can be categorized into **3 types**
><br>

<figure>
<center>
<a href = "https://web.ntnu.edu.tw/~algo/BinaryTree.html">
<img src = "/images/posts/jekyll/data_structure/binary_tree_all.png"/></a>
</center>
</figure>

Let's take a look at their description -- [programiz](https://www.programiz.com/dsa/full-binary-tree):
- A **Full** binary tree is a special type of binary tree in which every parent node/internal node has **either two or no children**.
  It is also known as **proper  binary tree**
- A **Complete** binary tree is similar to **full** binary tree but with **2 major differences** :
  1. All the leaf elements must **lean towards the left**.
  2. The **last leaf** element **might not** have a **right sibling** <br>i.e. a complete binary tree doesn't have to be a full binary tree.
- A **Perfect** binary tree is a type of binary tree in which every internal node has **exactly two child nodes** and **all the leaf nodes are at the same level**.

#### More on Types of Binary Tree
><br>The previous section only describe the general types of **Binary Tree**, but we can further categorize **Binary Tree** based on the structures of **left/right subtree**.
><br>

A **Balanced binary tree**, aka **height-balanced binary tree**, is defined as a binary tree in which **the height of the left and right subtree** of **any node** differ by **not more than 1**.

  <figure>
  <center>
  <a href= "https://adrianmejia.com/self-balanced-binary-search-trees-with-avl-tree-data-structure-for-beginners/">
  <img src = "/images/posts/jekyll/data_structure/balanced_vs_non_balanced_tree.jpg" style="width:80%"/>
  <center>
  </figure>






<br><br><br><br><br><br><br><br><br><br><br>
