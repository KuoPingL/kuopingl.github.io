---
layout: post
title:  "Algorithm - Sorting"
date:   2022-08-08 16:15:07 +0800
categories: [algorithm, beginner]
---

### Background
This article will focus on sorting algorithms including :
|Sorting Algorithm|Time Complexity|
|--|--|
|**Bubble** Sort|  **O(n<sup>2</sup>)**|
|**Selection** Sort| **O(n<sup>2</sup>)**|
|**Insertion** Sort |**O(n<sup>2</sup>)**|
|**Heap** Sort| **O(n log n)**|
|**Merge** Sort| **O(n log n)**|
|**Quicksort** |**O(n log n)**|

### Bubble
><br>This is the simplest sorting algorithm
>
> Bubble Sort will go from **right** to **left** and perform **compare & swap** between the two consecutive values until all values are sorted.<br><br>

Here are some of the rules used in this sorting :
1. Every set of iteration will always start from **[last index]**
2. For vector {a, b} :
   - if **a &ge; b** : then the **index** will point to the **index of a**
   - if **a &lt; b** : then they will swap and become **{b, a}**

**PS :** of course, you can always go from **left** to **right** so long you know what you are doing.

#### Demo
><br>Here is a simple demo showing how **Bubble Sort** can sort an array [ 5, 9, 3, 1, 2, 8, 4, 7, 6 ] in **ascending order**.
><br>

1. [ 5, 9, 3, 1, 2, 8, 4, **7**, **6** ] &rarr; ( compare 7 , 6 )
2. [ 5, 9, 3, 1, 2, 8, **4**, **6**, *7* ] &rarr; ( since 6 < 7 &rarr; swap 7 , 6 ; compare 4, 6 )
3. [ 5, 9, 3, 1, 2, **8**, **4**, 6 , 7 ] &rarr; ( compare 8 , 4 )
4. [ 5, 9, 3, 1, **2**, **4**, *8*, 6, 7 ] &rarr; ( swap 8 , 4 ; compare 2 , 4 )
5. [ 5, 9, 3, **1**, **2**, 4, 8, 6, 7 ] &rarr; ( compare 1 , 2 )
6. [ 5, 9, **3**, **1**, 2, 4, 8, 6, 7 ] &rarr; ( compare 3 , 1 )
7. [ 5, **9**, **1**, *3*, 2, 4, *8*, 6, 7 ] &rarr; ( swap 3 , 1 ; compare 9 , 1 )
8. [ **5**, **1**, *9*, 3, 2, 4, *8*, 6, 7 ] &rarr; ( swap 9 , 1 ; compare 5 , 1 )
9. [ **1**, *5*, 9, 3, 2, 4, 8, 6, 7 ] &rarr; ( swap 5 , 1 ) **DONE first Round**

After this, do the same thing again until it reaches n = 1 index.

10. [ **1**, 5, 9, 3, 2, 4, 8, **6**, **7** ]
11. [ **1**, 5, 9, 3, 2, 4, **8**, **6**, 7 ]
12. [ **1**, 5, 9, 3, 2, **4**, **6**, *8*, 7 ]
13. [ **1**, 5, 9, 3, **2**, **4**, 6, 8, 7 ]
14. [ **1**, 5, 9, **3**, **2**, 4, 6, 8, 7 ]
15. [ **1**, 5, **9**, **2**, *3*, 4, 6, 8, 7 ]
16. [ **1**, **5**, **2**, *9*, 3, 4, 6, 8, 7 ]
17. [ **1**, **2**, *5*, 9, 3, 4, 6, 8, 7 ]
18. ... **skip to result**

By the end, we will get :
[ **1**, **2**, **3**, **4**, **5**, **6**, **7**, **8**, **9** ]

#### Source Code
<details>
<summary><b>CPP ( Iteration )</b></summary>

```cpp
using namespace std;
#include <vector>

// Iteration
void iteratorSort(vector<int> &nums) {
    // improved
    for (int i = 0 ; i < nums.size() - 1 ; i++) {
        bool swapped = false;
        for (int j = nums.size() - 1 ; j > i ; j--) {
            if (nums[j] < nums[j-1]) {
                swapped = true;
                swap(nums, j-1, j);
            }
        }
        if (!swapped) break;
    }

    static void swap(vector<int>& nums, int indexA, int indexB) {
        int a = nums[indexA];
        nums[indexA] = nums[indexB];
        nums[indexB] = a;
    }
}
```

</details>

<details>
<summary><b>CPP ( Recursive )</b></summary>

```CPP
static void recursiveSort(vector<int> &nums, int i = 1) {
    int t_index = nums.size() - 1;

    if (i == nums.size() - 1) return;

    bool swapped = false;

    for (int j = nums.size() - 1 ; j > i ; j--) {
        if (nums[j] < nums[j-1]) {
            swapped = true;
            swap(nums, j, j-1);
        }
    }

    if (!swapped) return;
    recursiveSort(nums, ++i);
}
```

</details>
<br>

><br>**NOTE :** A swap consist of 3 operations (**read**, **write**, **write**)
><br>

#### Time Complexity
##### Big-O
In **Bubble Sort** the **worst scenario** is when the array is in revsersed at the beginning.

This way, the loop will go through all values.

If we examine the code, we can see that it requires **2 for-loops** :
- **outer** loop :
  ```CPP
  for (int i = 0 ; i < nums.size() - 1 ; i++)
  ```
  This loop will run **n - 1** times, where **n** is the **size** of the array.

- **inner** loop :
  ```CPP
  for (int j = t_index - 1 ; j >= i ; j--)
  ```

  This loop will run **n - i - 1** times, where **i** is the value of the current **i** value from outer loop.

  Let's assume the size of the array is 9, the inner loop will run :
  |i|inner loop will run :|
  |:--:|--|
  | 0  |9 - 0 - 1 = 8 times   |
  | 1  |9 - 1 - 1 = 7 times   |
  | n - 1  | n - (n - 1) - 1 = 0 times  |

Since the inner loop is dependent on the outer loop value, the total number of runs performed will be equal to the sum of the number of time the inner loop runs.

Therefore, for array with size of n, the total loop count will be :
  $$(n - 1)+(n - 2)+(n - 3)+...+ 1 + 0$$

Using the sum equation :
  $$\sum_{x=1}^{n - 1}x = \frac{(n-1)(n)}{2} \approx \frac{n^2}{2}$$

As a result, the Big-O is **O( n<sup>2</sup> )**


### Selection
><br>**Selection Sort** is quite similar to **Bubble Sort**, where they both has a Big-O of **O( n<sup>2</sup> )**.<br>
>The main difference between the two is how many times **compare** and **swap** are performed.
><br>

In **Selection Sort**, instead of performing **compare and swap** for every value, it will do the followings:
1. Find the **minimum** from the *unsorted* array
2. Swap the **minimum** with the smallest index in the *sorted* array.

Simply continue performing step 1 and 2 till all values are sorted.

**PS :** if you are trying sort it into **decending** order, you can find the **maximum** instead.

#### Demo
><br>Here is a simple demo showing how **Selection Sort** can sort an array [ 5, 9, 3, 1, 2, 8, 4, 7, 6 ] in **ascending order**.
><br>

1. [ 5, 9, 3, **1**, 2, 8, 4, 7, 6 ] &rarr; (scan for **min**)
2. [ **1**, 9, 3, **5**, 2, 8, 4, 7, 6 ] &rarr; (swap with **index 0**)
3. [ **1**, 9, 3, 5, **2**, 8, 4, 7, 6 ] &rarr; (scan for **min**)
4. [ **1**, **2**, 3, 5, **9**, 8, 4, 7, 6 ] &rarr; (swap with **index 1**)
5. ... **skip to result**

By the end, we will get :
[ **1**, **2**, **3**, **4**, **5**, **6**, **7**, **8**, **9** ]

#### Source Code

<details>
<summary><b>C++</b></summary>

```CPP
void sort(vector<int> &nums, bool iteration) {
    for(int i = 0 ; i < nums.size() - 1 ; i++) {

        int min_index = i;

        for (int j = i + 1 ; j < nums.size() ; j++) {
            if (nums[j] < nums[min_index]) {
                min_index = j;
            }
        }

        if (min_index != i) {
            int min = nums[min_index];
            nums[min_index] = nums[i];
            nums[i] = min;
        }
    }
}
```

</details>

#### Big-O
In **Selection Sort**, there is no such thing as **worst scenario** because we need to scan **all remaining values** to find the minimum value no matter how pre-sorted the array is.

If you look at the code, you can see how it resemble **Bubble Sort**.

Again, we need to examine the **inner** and **outer** for-loop separately :
- **outer** loop :
  ```CPP
  for(int i = 0 ; i < nums.size() - 1 ; i++)
  ```
  This will run **n - 1** times, where **n** is the size of the array.
- **inner** loop :
  ```CPP
  for (int j = i + 1 ; j < nums.size() ; j++)
  ```
  This will run **n - ( i + 1 )** or **n - i - 1** times.

  Whereas the maximum of **i** is **n - 2** and minimum is **0**, making the minimum and maximum of the loop count to be : **1** and **n-1**, respectively.

Since the number of times the **inner** loop runs is dependent on the **i** value from the **outer** loop, the total number of times loops are performed is also calculated

$$\sum_{x = 1}^{n - 1} x = \frac{(n-1)(n)}{2} \approx \frac{n^2}{2}$$

Hence, the Big-O for **Selection Sort** is also **O( n<sup>2</sup> )**.

#### What has improved ?
Compared to **Bubble** Sort, **Selection** Sort has the following adventage :
- Less computation
  Instead of performing **compare and swap** for every values, **Selection Sort** also compare every values, but only perform **n** times of **swap** in total.

### Insertion
><br>**Insertion Sort** also has the same time complexity, a Big-O of **O ( n<sup>2</sup> )**, **Insertion Sort** as **Bubble Sort** and **Selection Sort**.
>
>An example when **Insertion Sort** is performed is the way we **sort cards while playing poker**.
><br>

Here is how **Insertion Sort** is done :
1. Select the value at **index 0** as the **sorted ( left )** array **initial** value.
2. Select the next value on the **right** and **insert** into the **left** array through **shift and insert**.


The rest of the process will simply repeat step **2** until the **right** array is empty.

><br>**NOTE :** both shifting and insert only perform a single *write*, which is more efficient than performing *swap*.
><br>

#### Demo
><br>Here is a simple demo showing how **Insertion Sort** can sort an array [ 5, 9, 3, 1, 2, 8, 4, 7, 6 ] in **ascending order**.
><br>

0. [ <u>**5**</u>, 9, 3, 1, 2, 8, 4, 7, 6 ] &rarr; ( pick **5** as the **inital** value )
2. [ <u>**5**</u>, **9**, 3, 1, 2, 8, 4, 7, 6 ] , [**9**] &rarr; ( holds 9 ; compare 9 and 5 )
3. [ <u>**5**</u>, <u>**9**</u>, **3**, 1, 2, 8, 4, 7, 6 ] , [**3**] &rarr; (holds 3 ; compare 3 with 9)
4. [ <u>**5**</u>, 9, <u>**9**</u>, 1, 2, 8, 4, 7, 6 ] , [**3**] &rarr; (since **9 > 3**, shift 9 to the right )
5. [ 5, <u>**5**</u>, <u>**9**</u>, 1, 2, 8, 4, 7, 6 ] , [**3**] &rarr; ( since **5 > 3** , shift 5 to the right )
6. [ <u>**3**</u>, <u>**5**</u>, <u>**9**</u>, 1, 2, 8, 4, 7, 6 ] &rarr; ( place 3 back to array )
7. [ <u>**3**</u>, <u>**5**</u>, <u>**9**</u>, **1**, 2, 8, 4, 7, 6 ] , [**1**] &rarr; (holds 1 ; compare 1 and 9)
8. ... **skip to result**

Now you should be able to understand how **Insertion Sort** is done, the final result should be [ **1**, **2**, **3**, **4**, **5**, **6**, **7**, **8**, **9** ]

#### PsudoCode
Here are something to keep in mind :
- The inital value of the sorted array is always **index 0**, hence we can **skip this index** when defining the sub-array.
- The outer loop should end when all values have been looped.
- The inner loop should end when value has inserted at the right location.

```CPP
int n = nums.size();

for i in 1 to n {
  int target = nums[i];
  int j = i - 1;
  while (j > 0 && nums[j] > t) {
    nums[j + 1] = nums[j];
    j--;
  }

  nums[j + 1] = target;
}
```


#### Source Code

<details>
<summary><b>C++ ( Iteration ) </b></summary>

```cpp
static void iterationSortWithWhile(vector<int>& nums) {
    for (int i = 1 ; i < nums.size() ; i++) {
      int t = nums[i];
      int j = i - 1;

      while (j > 0 && nums[j] > t) {
        nums[j + 1] = nums[j];
        j--;
      }

      nums[j + 1] = t;
    }
}

// you can also use for-loop instead :
static void iterationSort(vector<int>& nums) {
    for (int i = 1 ; i < nums.size() ; i++) {
        int t = nums[i];
        for (int j = i ; j > 0 ; j--) {
            if (nums[j - 1] > t) {
                nums[j] = nums[j - 1];
            } else {
                nums[j] = t;
                break;
            }

            if (j - 1 == 0) {
                nums[j - 1] = t;
                break;
            }
        }
    }
}
```

</details>

<details>
<summary><b>C++ ( Recursive )</b></summary>

```cpp
// Recursive can take care the first for-loop
static void recursiveSort(vector<int>& nums, int i = 1) {
    if (i == nums.size()) return;
    int t = nums[i];
    int j = i;

    while (j != 0) {
        if (nums[j - 1] > t) {
            nums[j] = nums[j - 1];
            j--;
        } else {
            break;
        }
    }

    nums[j] = t;

    recursiveSort(nums, i+1);
}
```

</details>

#### Big-O
If you look at the code, you can see that we still need 2 for-loops, and the number of time inner loop runs depends on the outer loop value ( **i** ).

- **outer** loop :
  ```CPP
  for (int i = 1 ; i < nums.size() ; i++)
  ```
- **inner** loop :
  ```CPP
  for (int j = i ; j > 0 ; j--)
  ```
Since the **inner** loop depends on **outer** loop, again, we need to add up the amount of time the second loop runs :
$$\sum_{x = 0}^{n-1} x = (n-1) + (n - 2) + ... + 0$$
$$= 1 + ... + (n-1) = \frac{(n)(n-1)}{2} \approx \frac{n^2}{2}$$

As a result, **Insertion Sort** also has a Big-O of **O ( n<sup>2</sup> )**.
#### What has improved ?
Even through **Insertion** also has the same time complexity as **Bubble** and **Selection** Sort, but it is much faster.

The reason has to do with the number of operations these algorithms need to go through.

In the worst case scenario :
- **Bubble Sort** needs to perform **n** times of **comparison** and **swap** for every value.
- **Selection Sort** needs to perform **n** times of **comparison** and a **single swap**.
- **Insertion Sort** needs to perform **n** times of **comparison** and **write**.

><br>**NOTE :** Recall **swap** requires 3 operations (**read**, **write**, **write**).
><br>

Since **Insertion Sort** only uses **write**, so **Insertion** is more *efficient* than **Bubble** and **Selection Sort**.

### Heap Sort
><br>A **Heap Sort** uses **heap** (a data structure) to sort the array.
><br>


#### What's a Heap ?
><br>A **Heap** is a special **Tree-based** data structure in which the tree is a **complete binary tree**. -- [geeksforgeeks](https://www.geeksforgeeks.org/heap-data-structure/)
>
> A **complete binary tree** is a special type of binary tree where **all the levels** of the tree are **filled completely** *except* **the lowest level nodes** which are *filled from as left as possible*. -- [geeksforgeeks](https://www.geeksforgeeks.org/complete-binary-tree/)
><br>

<figure>
<center>
<img src = "/images/posts/jekyll/algorithms/data_structure_heap.jpg" style="width:80%"/>
<br>Complete Binary Tree
</center>
</figure>

#### Demo
><br>In order to sort an array with a heap, we need to create a tree that has a root with the largest value.
>The rest of the nodes will be created from **left** to **right**, with values going from **largest** to **smallest**.
><br>

Let's try to sort an array [ 5, 9, 3, 1, 2, 8, 4, 7, 6 ] using **Heap Sort** :
1. [ 5, **9**, 3, 1, 2, 8, 7, 6 ] ( pick **9** as root )
   [ [ **9** ] ]
2. [ 5, **9**, 3, 1, 2, **8**, 7, 6 ] ( pick **8** as the next node )
   [ [ **9** ] , [ **8** ] ]
3. [ 5, **9**, 3, 1, 2, **8**, **7**, 6 ] ( pick **7** as the next node )
   [ [ **9** ] , [ **8**, **7** ] ]
4. [ 5, **9**, 3, 1, 2, **8**, **7**, **6** ] ( pick **6** as the next node at next level )
   [ [ **9** ] , [ **8**, **7** ] , [ **6** ] ]

#### Source Code

<details>
<summary><b>C++ </b></summary>

</details>

#### Big-O


### Merge

#### Demo


#### Source Code

<details>
<summary><b>C++ </b></summary>

</details>

#### Big-O

### Quicksort

#### Demo


#### Source Code

<details>
<summary><b>C++ </b></summary>

</details>

#### Big-O


### Counting Sort

#### Demo


#### Source Code

<details>
<summary><b>C++ </b></summary>

</details>

#### Big-O

### Radix Sort

#### Demo


#### Source Code

<details>
<summary><b>C++ </b></summary>

</details>

#### Big-O

### Bucket Sort

#### Demo


#### Source Code

<details>
<summary><b>C++ </b></summary>

</details>

#### Big-O



<br><br><br><br><br><br><br><br><br><br>
