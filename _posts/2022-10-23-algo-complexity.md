---
layout: post
title:  "Algorithm - Complexity"
date:   2022-10-23 17:38:00+08:00
categories: [algorithm, beginner]
---

### Background
><br>Whenever we are evaluating an algorithm, we always compare their **complexity** or the **Big O**.
><br>

But, do you know what is a Big O ? and what it means ?

In this article, we are going to discuss about that.

### Complexity
><br> **Complexity** is defined as the **amount of computational resources** (time and memory) that is required to run the algorithm.
> <br>In another word, it defines the **efficiency of the algorithm**.
><br>

We can examine **computational complexity** through  **Time** and **Space Complexity**.

><br>**Time Complexity** is the amount of computational time it takes for the algorithm to complete.
>
>**Space Complexity** is the amount of memory storage it takes to run the algorithm.
><br>

but most of the time, we will see algorithms being compared using **time complexity**.

<u>**So how can we describe complexities ? Or how can we describe the algorithm's efficiency ?**</u>
<br>This is where **Asymptotic Analysis** comes in.


### Asymptotic Analysis
><br>In mathematical analysis, **asymptotic analysis**, also known as <b>*asymptotics*</b>, is a method of describing **limiting behavior**. ---[wiki](https://en.wikipedia.org/wiki/Asymptotic_analysis)
><br>

The popular **notations** in asymptotic analysis are :
- **Big O** notation ( **O (expression)** )
- **Omega** notation ( **Ω (expression)** )
- **Theta** notation ( **Θ (expression)** )

><br>**Big-O** is used to describe the **worst case** complexity, which explains the maximum amount of time an algorithm requires considering all input values [ [geeksforgeeks](https://www.geeksforgeeks.org/analysis-of-algorithms-set-2-asymptotic-analysis/) ].
><br>

><br>**Omega** describes the **best case** senario.
><br>

and

><br>**Theta** defines the **average case**.
><br>

**<u>Now that we know the definition, how do we find the limit ?</u>**

### Calculating the Time Complexity of Insertion Sort

In this section, we will take **Insertion Sort** as an example.

Before examining the psudocode, there are something that you need to know.

Every line of code that gets proceeded require certain amount of time, usually we would assume this is a constant time or **cost** ( **c <sub>i</sub>** ).

To find the total cost of a certain line of code, we simply need to find the number of time the code will run, **n**, and multiply it by the **cost** :
$$\begin{aligned}
TotalTime = n\times{c_i}
\end{aligned}
$$

where **i** is the line of the code

Now, let's take a look at the psudocode :
```CPP
1 int n = nums.size();
2
3 for i in 2 to n { // array[1:n]
4  int target = nums[i];
5  int j = i - 1;
6  while (j > 0 && nums[j] > t) {
7    nums[j + 1] = nums[j];
8    j--;
9  }
10 nums[j + 1] = target;
11}
```

Here is the cost and number of time each line will run :

|line #|code|cost|# of run|
|:--:|--|:--:|:--:|
|1   | int n = nums.size( );   | c<sub>1</sub> | 1  |
|3   | for i in 1 to n  |  c<sub>3</sub>  |  n |
|4   | int target = nums[ i ];  | c<sub>4</sub>  | n - 1  |
|5   |  int j = i - 1; | c<sub>5</sub>  |  n - 1 |
|6   |  while ( j > 0 && nums[ j ] > t) | c<sub>6</sub>   |  $\sum_{i = 2}^{n}m_i$ |
|7   |  nums [ j + 1 ] = nums[ j ]; | c<sub>7</sub>  | $\sum_{i = 2}^{n}(m_i - 1)$  |
|8   | j--;  | c<sub>8</sub>  | $\sum_{i = 2}^{n}(m_i - 1)$  |
|10   | nums[ j + 1 ] = target;  | c<sub>10</sub>   |  n - 1 |

><br>since the inner loop depends on the **i** value, therefore, we uses **m <sub>i</sub>** to represent the number of time the code will run.
><br>

Here is a [great tutorial](https://www.youtube.com/watch?v=tmKUHLs21PU) that explains why the number of runs are the way they are.

><br>If you are curious on why are the loops run **n** times while their contents run **n - 1** times, it's simply because loops need **an extra step** to exit the loop.
><br>

So if we sums up the total time it takes to run a **Insertion Sort**, we get :
$$T(n) = c_1 + c_3n + c_4(n - 1)+c_5(n-1)+c_6\sum_{i = 2}^{n}m_i\\+c_7\sum_{i = 2}^{n}(m_i - 1)+c_8\sum_{i = 2}^{n}(m_i - 1)+c_{10}(n - 1)$$

Now with the formula prepared, we can examine its performance in **best-case**, **average-case** and **worst-case** scenario.

#### Best Case
><br>The best case scenario is obviously when the array is already sorted.
><br>

In this case, the inner loop will never enter thus $m_i = 1$ :
$$T(n)=c_1+c_3n + c_4(n-1) + c_5(n-1)\\+ c_6(n-1)+c_{10}(n-1)$$
$$T(n)= (c_3+c_4+c_5+c_6+c_{10})n \\+ (c_1+c_3-c_4-c_5-c_6-c_{10})$$

Hence, in the best scenario, the **time complexity** is **Ω ( n )**, aka **linear function of n** [ [ref](https://github.com/Mcdonoughd/CS2223/blob/master/Books/Algorithhms%204th%20Edition%20by%20Robert%20Sedgewick,%20Kevin%20Wayne.pdf) ].


#### Worst Case
><br>In the worst case scenario, the array is in the **reversed order**.
>ie [ 6, 5, 4, 3, 2, 1 ] if we wish to get a accending order.
><br>

In this case, all values need to be shifted by $i$ times :

|array|actions|
|:--|:--|
| [ **6**, 5, 4, 3, 2, 1 ]  |  select **6 ( nums[ 0 ] )** as the inital value |
| [ **5**, **6**, 4, 3, 2, 1 ]  | insert **5** ( **nums[ 1 ]** ) by <br>shifting **6** by 1 step  |
| [ **4**, **5**, **6**, 3, 2, 1 ]  | insert **4** ( **nums[ 2 ]** ) by <br>shifting **5, 6** by 2 steps  |

And so forth.

Hence, $m_i = i$ [ [ref](https://github.com/Mcdonoughd/CS2223/blob/master/Books/Algorithhms%204th%20Edition%20by%20Robert%20Sedgewick,%20Kevin%20Wayne.pdf) ].:

$$\sum_{i = 2}^{n}m_i = \sum_{i = 2}^{n}i = \sum_{i = 1}^{n}i - 1\\\sum_{i = 1}^{n}i - 1 = \frac{(n + 1)(n)}{2} - 1 $$

Similarly [ [ref](https://github.com/Mcdonoughd/CS2223/blob/master/Books/Algorithhms%204th%20Edition%20by%20Robert%20Sedgewick,%20Kevin%20Wayne.pdf) ].:

$$\sum_{i = 2}^{n}(m_i - 1) = \sum_{i = 2}^{n}(i - 1) = \sum_{i = 1}^{n - 1}(i - 1)\\\sum_{i = 1}^{n - 1}(i - 1) = \frac{((n - 1 + 1)(n - 1))}{2} = \frac{(n)(n - 1)}{2}$$

As a result :

$$T(n) = c_1+c_3n + c_4(n-1) + c_5(n-1)\\+ c_6(\frac{(n+1)(n)}{2} -1)+c_7\frac{(n)(n-1)}{2}+\\c_8\frac{(n)(n-1)}{2} + c_{10}(n-1)$$

$$T(n) = n^2(\frac{c_6}{2} + \frac{c_7}{2} + \frac{c_8}{2}) \\+ n(c_3+c_4+\frac{c_6}{2} - \frac{c_7}{2} - \frac{c_8}{2} + c_{10})   \\+( c_1 - c_4 - c_5 - c_6 - c_{10})$$

From this, we can express the worst case runtime as a **quadratic fucntion**:

$$an^2+bn+c$$

As a result, in the **worst-case** scenario, **Insertion Sort** has a Big-O of **O ( n<sup>2</sup> )**.


><br>**Worst case** scenario is usually what we focused because it gives us an **upper-bound** to the runtime for *any* inputs, which is essential for real-time computation [ [ref](https://github.com/Mcdonoughd/CS2223/blob/master/Books/Algorithhms%204th%20Edition%20by%20Robert%20Sedgewick,%20Kevin%20Wayne.pdf) ].
><br>

#### Average Case
><br>**Average case** analysis is *seldom* discussed because usually it is *as bad as* the **worst case** scenario.
><br>

This is true in the case of **Insertion Sort**.
On average, half the of values are sorted, thus :

$$m_i = \frac{i}{2}$$

which will also give a time complexity of **Θ ( n<sup>2</sup> )**.

### Space Complexity of Insertion Sort
Since **Insertion Sort** is a *in-place* algorithm, no extra space is needed, thus it will have a **space complexity** of **O ( 1 )**.

### Summary
I hope you get a simple understanding on how complexity is calculated.

### Further Readings
Coming Soon

<br><br><br>
