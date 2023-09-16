---
layout: post
title:  "Algorithm - Search"
date:   2022-11-14 15:27:40 +08:00
use_math: true
categories: [algorithm, beginner]
---

### Background

><br>**Searching Algorithms** are designed to check for or retrieve an element from any data structure where it is stored, **sorted or not**.
><br/>

In general, searching algorithms can be categorized into **2 categories** :
- **Sequential** Search (**continuous**) - ie **Linear Search**
- **Interval** Search (**discrete**) - ie **Binary Search**


Table 1 : Sequential Search
|algorithm|Best|Average|Worst|Space|
|--:|:--:|:--:|:--:|:--:|
|[Linear Search](#linear-search)   | $$O(1)$$  |  $$O(n/2)$$ | $$O(n)$$  | $$O(1)$$  |


#### Reference
1. [geeksforgeeks: Searching Algorithms](https://www.geeksforgeeks.org/searching-algorithms/)


### Linear Search
||Best|Average|Worst|Space|Sorted
|--:|:--:|:--:|:--:|:--:|:--:|
|Complexity   | $$O(1)$$  |  $$O(\frac{n}{2})$$ | $$O(n)$$  | $$O(1)$$  |$$\times$$|

><br>**Linear Search** is the easiest search algorithm. It is defined as a *sequential* search that goes from 1 end of a **list** and search through each element until the desired element is found.
><br/>

<figure>
<center>
<img src = "/images/posts/jekyll/algorithms/search/linear-search-geeks.png" style="width:90%"/>
<a href="https://www.geeksforgeeks.org/linear-search/">Linear Search </a>
</center>
</figure>

#### Time Complexity
The time complexity is quite straight forward.

In the **best scenario**, the desired value is located at the first index, making the time complexity to $O(1)$.

However, most of the time this is not the case.

In the **average scenario**, the input array is unsorted, meaning target value is located somewhere between **1 : n**, which makes the average time complexity to be $O(n/2)$.

As for the **worst scenario**, the target value is located at the **end** of the array, thus $O(n)$.

#### Space Complexity
Since there is no extra space needed to perform **Linear Search**, the space complexity is $O(1)$.

#### Source Code
Since **Linear Search** is quite simple, the psudocode and demo are not discussed.

<details>
<summary><b>C++ Iteration Approach</b></summary>

```cpp
using namespace std;
#include <vector>

class LinearSearch {
    public :
    int search(vector<int>& input, int target) {
        for (int i = 0; i < input.size(); i++) {
            if (input[i] == target) return i;
        }
        return -1;
    }
};
```

</details>

<details>
<summary><b>C++ Recursive Approach</b></summary>

```cpp
using namespace std;
#include <vector>

class LinearSearch {
    public :
    int recursiveSearch(vector<int>& input, int target, int index) {
        if (index > input.size() - 1) return -1;
        if (input[index] == target) return i;

        return recursiveSearch(input, target, index++);
    }
};
```

</details>

#### Reference
1. [wiki](https://en.wikipedia.org/wiki/Linear_search)
2. [geeksforgeeks: Linear Search](https://www.geeksforgeeks.org/linear-search/)


### Binary Search








<br><br><br><br><br><br><br><br><br><br><br><br>
