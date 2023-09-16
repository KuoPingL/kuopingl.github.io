---
layout: post
title:  "GG Algorithm Problem - Floyd Warshall"
date:   2022-08-08 16:15:07 +0800
categories: [algorithm problem]
---

difficulty : medium

### Floyd Warshall
><br>The problem is to find the **shortest distances** between every pair of vertices in a given edge-weighted directed graph. <br>The graph is represented as an adjacency matrix of size $n\times n$. <br><br>$\text{Matrix}[i][j]$ denotes the weight of the edge from i to j. <br>If $\text{Matrix}[i][j]=-1$, it means there is no edge from i to j. <br><br>Do it **in-place**.
><br><br/>

[source](https://practice.geeksforgeeks.org/problems/implementing-floyd-warshall2042/1)

#### Examples
<b><u>Example 1</u></b>
**Input**: matrix = $\{\{0,25\},\{-1,0\}\}$

||0|1|
|:--:|:--:|:--:|
|0   | 0  |  25 |
|1   |  -1 |  0 |

**Output**: $\{\{0,25\},\{-1,0\}\}$

||0|1|
|:--:|:--:|:--:|
|0   | 0  |  25 |
|1   |  -1 |  0 |

**Explanation**:
The shortest distance between every pair is already given(if it exists).

<b><u>Example 2</u></b>
**Input**: matrix = $\{\{0,1,43\},\{1,0,6\},\{-1,-1,0\}\}$

||0|1|2|
|:--:|:--:|:--:|:--:|
|0   | 0  |  1 |43|
|1   |  1 |  0 |6|
|2   |-1   |-1   |0|

**Output**: $\{\{0,1,7\},\{1,0,6\},\{-1,-1,0\}\}$

||0|1|2|
|:--:|:--:|:--:|:--:|
|0   | 0  |  1 |7|
|1   |  1 |  0 |6|
|2   |-1   |-1   |0|

**Explanation**:
We can reach 2 from 0 as **0 &rarr; 1 &rarr; 2** and the cost will be 1+6=7 which is less than 43.


#### Tasks
><br>You don't need to read, return or print anything. Your task is to complete the function shortest_distance() which takes the matrix as input parameter and modifies the distances for every pair in-place.
>
>Expected Time Complexity: $O(n^3)$
>Expected Space Complexity: $O(1)$
>
>Constraints:
>$1 <= n <= 100$
>$-1 <= matrix[ i ][ j ] <= 1000$
><br><br/>





<br><br><br><br><br><br><br><br><br><br><br>
