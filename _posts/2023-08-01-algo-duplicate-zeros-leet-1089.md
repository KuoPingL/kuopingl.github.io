---
layout: post
title: Algorithm：LeetCode 1089 Duplicate Zeros
date: 2023-08-01 00:40:20 +08:00
categories: [Algorithm]
use_math: true
keywords: RecyclerView, git diff, Longest Common Subsequence, LCS
---

## [LeetCode 題目](https://leetcode.com/problems/duplicate-zeros/description/)

Given a fixed-length integer array `arr`, duplicate each occurrence of zero, shifting the remaining elements to the right.

Note that elements beyond the length of the original array are not written. Do the above modifications to the input array in place and do not return anything.

**Example 1**:

```kotlin
Input: arr = [1,0,2,3,0,4,5,0]
Output: [1,0,0,2,3,0,0,4]
Explanation: After calling your function, the input array is modified to: [1,0,0,2,3,0,0,4]
```

**Example 2**:

```kotlin
Input: arr = [1,2,3]
Output: [1,2,3]
Explanation: After calling your function, the input array is modified to: [1,2,3]
```

**Constraints**:

1 <= `arr.length` <= $10^4$
0 <= `arr[i]` <= 9

## 理解題目
這個題目其實很容易理解。

當我們在轉換 $A$ 至 $A'$ ($A \rightarrow A'$) 時，將遇到的 $0_i$ 都插入一個新的 $0_{i + 1}$，並將最後的值移除。 如同以下 ：

$$\begin{align} \nonumber
A_{0,7} &= [1,0,2,3,0,4,5,0]
\\\nonumber A'_{0,7} &= [1, 0, 0, 2, 3, 0, 0, 4]
\end{align}$$


## 解題

### Brutal Force

||Complexity|
|:--|:--|
|Space  | $O(2n)$ |
|Time   | $O(1)$   |

<br>

這是我最一開始的解法。 由於我們需要插入 $0$，但同時也需要保留後面的值來進行分析。 所以我就用 **Queue** 將被 $0$ 取代的值記錄下來，並通過取代與移除來將陣列的值往右移。

但當我們遇到 $0$ 時，我們會遇到兩種情況：

1. 遇到第一個 $0_i$ 時，我們只需要將後者 $A[i + 1]$ 記錄並將其改為 $0$ 。
2. 遇到之後的 $0$ 時，我們需要將 $A[i]$ 與 $A[i + 1]$ 兩者記錄並將兩者皆更改為 $0$。
<br>

根據這兩個規則，當我們對 `arr` 中的值逐步分析時會得到以下結果：

|i|`arr`|Queue|
|:--:|:--|:--|
|0   |  $[{\color{red}{1}},0,2,3,0,4,5,0]$ |  $[]$ <br>尚未遇到首個 $0$，因此不會記錄。|
|1   | $[1, {\color{red}{0}},2,3,0,4,5,0]$ <br> $[1, {\color{red}{0}},{\color{red}{0}},3,0,4,5,0]$ | $[2]$ <br>遇到首個 $0$ 所以將 $2$ 記錄並由 $0$ 取而代之。  |
|2   | ---  | 由於這是新增的 $0$ 所以略過  |
|3   | $[1,0,0, {\color{red}{3}},0,4,5,0]$ <br> $[1,0,0, {\color{red}{2}},0,4,5,0]$  | $[3]$ <br> 因為 Queue 中有值，所以我們通過 FIFO 取出 $2$ 並取代並記錄 $3$ 。  |
|4   | $[1,0,0, 2, {\color{red}{0}},4,5,0]$ <br> $[1,0,0, 2, {\color{red}{3}},4,5,0]$ | $[0]$ <br> 同上  |
|5   |$[1,0,0, 2, 3, {\color{red}{4}},5,0]$ <br> $[1,0,0, 2, 3, {\color{red}{0}},5,0]$ <br> $[1,0,0, 2, 3, {\color{red}{0}},{\color{red}{0}},0]$ | $[4, 5]$ <br> 由於 Queue 的下一個值為 $0$，所以我們需要將 $A[5]$ 與 $A[6]$ 的值記錄起來，並將他們由 $0$ 來取代。  |
|6   | ---  | 略過  |
|7   | $[1,0,0, 2, 3, 0, 0, {\color{red}{0}}]$ <br> $[1,0,0, 2, 3, 0, 0, {\color{red}{4}}]$| $[5, 0]$ <br>此時我們取出 $4$ 並將最後的 $0$ 記錄並取代掉。 |

因為我們已經到陣列的 `length` 了，所以剩下的 $[5, 0]$ 都會被剔除掉。

以下便是執行以上流程的兩種寫法：

```cpp
class Solution {
public:
    void duplicateZeros(vector<int>& arr) {

        vector<int> storedValues;

        // find the first zero and do a simple replace
        for (int i = 0; i < arr.size(); i ++) {
            int currentValue = arr[i];

            if (currentValue == 0) {
                if (i + 1 < arr.size()) {
                    storedValues.push_back(arr[i + 1]);
                    arr[i + 1] = currentValue;
                    beginDuplicateZeros(arr, i + 2, storedValues);
                    break;
                }
            }
        }
    }

    void beginDuplicateZeros(vector<int>& arr, int index, vector<int>& storedValues) {

        if (index >= arr.size()) return;

        for (int i = index; i < arr.size(); i++) {
            int storedValue = pop_front(storedValues);

            if (storedValue == 0) {
                storedValues.push_back(arr[i]);
                arr[i] = storedValue;
                i++;
                if (i < arr.size()) {
                    storedValues.push_back(arr[i]);
                    arr[i] = storedValue;
                }
            } else {
                storedValues.push_back(arr[i]);
                arr[i] = storedValue;
            }
        }

    }

    int pop_front(vector<int> &v)
    {
        int result = -1;
        if (v.size() > 0) {
            result = v[0];
            v.erase(v.begin());
        }
        return result;
    }


};
```
<br>

以下是另一種寫法。 邏輯也是一樣，但增加了一個 Flag 來判斷何時找到第一個 $0$ 。


```kotlin
class Solution {
    fun duplicateZeros(arr: IntArray): Unit {
        val queue = LinkedList<Int>()
        var isFirstZeroFound = false
        var i = 0

        while (i < arr.size) {
            if (arr[i] == 0 && !isFirstZeroFound) {
                i++
                if (i >= arr.size) break;
                queue.add(arr[i])
                arr[i] = 0
                isFirstZeroFound = true
                i++
                continue;
            }

            if (!isFirstZeroFound) {
                i++
            } else {

                val cur = queue.remove()
                if (cur == 0) {
                    queue.add(arr[i])
                    arr[i] = cur
                    i++
                    if (i < arr.size) {
                        queue.add(arr[i])
                        arr[i] = cur
                        i++
                    }
                } else {
                    queue.add(arr[i])
                    arr[i] = cur
                    i++;
                }
            }
        }

    }
}
```

#### 複雜度

由於每次遇到 $0$ 都需要將後面兩個值都記錄起來 ( 除了首個 $0$ )，所以每次遇到 $0$ 記憶體就需要 $2$ 格。


因此，在最差的情況下：

$$A_{0,m} = [0, 0, 0, 0, 0 ... 0]$$

所需要的記憶體的大小為： $2n - 1$，$n$ 是陣列中 $0$ 的數量。

><br>
>
> **PS** : 減 $1$ 是因為第一個 $0$ 只需要記錄 $1$ 個值。
><br>

也就是說 **空間複雜度** 是 $O(2n)$。

另外， **時間複雜度** 則是 $O(m)$ ， $m$ 為列舉的長度，也可以寫成 $O(1)$ 。

當然，這個演算法真的不夠優化。 接下來就是看看我們如何優化他。

### 空間優化

[參考資料](https://home.gamer.com.tw/artwork.php?sn=5101808)

||Complexity|
|:--|:--:|
|Space   | $O(m)$ 或 $O(1)$  |
|Time   |   |

按照題目的要求，當我們將新的 $0$ 插入後，多出來的值都會被拋棄。
所以 $A$ 與最後陣列 $A'$ 的關係如下：

$$\begin{align}\nonumber
A &= [a_0, a_1, a_2, ... , a_m]
\\\nonumber &= A_{0,last\_index} + A_{last\_index + 1, m}
\\\nonumber A' &= A_{0,last\_index} + 0_{m - last\_index}
\end{align}$$


也就是說，$A'$ 其實是由 $A_{0,last\_index}$ 以及新的 $0$ 所組成的。

所以我們節省空間的作法便是通過計算出 $last\_index$ 來省去 **Queue** 的使用。

$last\_index$ 的值要算出來也很簡單。因為我們知道 $0$ 需要佔用 $2$ 格，而其他數字則佔用 $1$ 格。而當 **( 所需的格子總和 )**  $\ge (m + 1)$ 時，我們就找到 $last\_index$ 了。

例子：

$$\begin{align}
\nonumber A_{0,7} &= [1,0, 2,3,0,4,5,0]
\\ \nonumber blocks &= 1_1 + 2_0 + 1_2 + 1_3 + 2_0 + 1_4 = 8
\end{align}$$

所以對 $A_{0,7}$ 而言， $last\_index = 5$ 因為當我們遇到 $A[5]$ 時，**總和** $\ge 8$ 。

如此一來，我們新的演算法有兩個步驟：
1. 算出 $last\_index$
2. 將新的 $0$ 插入正確的位置

問題又來了，我們要如何插入 $0$ 呢？

其實也就只有兩種方法：
1. 從 $A[0]$ 開始，逐步讀取與分析。這就如同之前方法。
2. 從 $A[m]$ 開始，從 $A[last\_index]$ 開始讀取與分析，並從 $m$ 開始進行取代。

第一種做法其實就跟 [Brutal Force](#brutal-force) 的作法一樣。 但第二種作法卻可以讓我們省去 **Queue**，如下：

|i|$last\_index$|$A$|
|:--:|:--|:-|
|7   |5  |$[1,0,2,3,0,{\color{red}4},5,{\color{green}0}]$ <br><br> $A[i] = A[last\_index]$ <br><br>$[1,0,2,3,0,4,5,{\color{gray}4}]$ |
|6   |4   | $[1,0,2,3,{\color{red}0}, 4,{\color{green}5},{\color{gray}4}]$ <br><br> 由於遇到 $0$ 所以 $A[i]$ 與 $A[i - 1]$ 都需要變成 $0$ 。 <br><br> $[1,0,2,3,0, {\color{gray}0},{\color{gray}0},{\color{gray}4}]$|
|5   |  4 | 略過  |
|4   | 3  |  $[1,0,2,{\color{red}3},{\color{green}0}, {\color{gray}0},{\color{gray}0},{\color{gray}4}]$<br><br> $[1,0,2,3, {\color{gray}3}, {\color{gray}0},{\color{gray}0},{\color{gray}4}]$ |
| 3  | 2  | $[1,0,{\color{red}2},{\color{green}3},{\color{gray}3}, {\color{gray}0},{\color{gray}0},{\color{gray}4}]$<br><br> $[1,0,2,{\color{gray}2},{\color{gray}3}, {\color{gray}0},{\color{gray}0},{\color{gray}4}]$  |
|2   |1   | $[1,{\color{red}0},{\color{green}2},{\color{gray}2},{\color{gray}3}, {\color{gray}0},{\color{gray}0},{\color{gray}4}]$ <br><br> $[1,{\color{gray}0},{\color{gray}0},{\color{gray}2},{\color{gray}3}, {\color{gray}0},{\color{gray}0},{\color{gray}4}]$ |
|1   | 1  |  由於 $i \le last\_index$ 所以結束 |


但我們還需要考慮到另一種可能性，也就是若 $A[last\_index] = 0$ 但在 $A'$ 中，複製出來的 $0$ 會被推擠出去。 如同以下範例：

$$\begin{align}\nonumber
A &= [1,0,2,3,0,0,5,0]
\\\nonumber blocks &= 1_1 + 2_0 + 1_2 + 1_3 + 2_0 + 2_0 \gt 8
\end{align}$$

此時，我們發現 **總和** $= 9$，而 $A'$ 則是：

$$\begin{align}\nonumber
A' &= [1, 0, 0, 2, 3, 0, 0, {\color{red}0}]
\end{align}$$

而 $A[5]$ 的 $0$ 所複製出來的 $0$ 並不在 $A'$ 中。

考量到這個問題，我們只需要在 $last\_index \gt m$ 時，將 $A[m]$ 改為 $0$ 並將 $last\_index = last\_index - 1$ ：

$$\begin{align}\nonumber
A &= [1,0,2,3,0,0,5,{\color{gray}0}]
\end{align}$$


以下便是這次的演算法：

```kotlin
class Solution {
    fun duplicateZeros(arr: IntArray): Unit {
        var blocks = 0
        var lastIndex = -1
        var targetIndex = arr.size - 1

        for (i in 0 until arr.size) {
            if (arr[i] == 0) {
                blocks += 2
            } else {
                blocks ++
            }

            if (blocks >= arr.size) {
                lastIndex = i
                if (blocks > arr.size) {
                    arr[targetIndex] = 0
                    lastIndex --
                    targetIndex --
                }
                break;
            }
        }

        while(lastIndex >= 0) {

            val target = arr[lastIndex]
            if (target == 0) {
                arr[targetIndex--] = target
                arr[targetIndex--] = target
            } else {
                arr[targetIndex--] = target
            }
            lastIndex --
        }

    }
}
```

#### 複雜度

這次的演算法與 [Brutal Force](#brutal-force) 寫法主要差在空間複雜度上。

由於我們這次不是將 $0$ 後面所需要移動的值存放在 $Queue$ 中，所以省下不少空間。

不僅如此，我們只使用了幾個參數來記錄所需資料。 也就是說這次的空間複雜度為 $O(1)$，而時間複雜度維持 $O(1)$。


### 優化寫法

我們上面是通過計算所需的格子數量來找出 $last\_index$。

那我們有沒有其他作法呢？ 我朋友 George 就找到另一個解法。

我們可以通過計算 `arr` 中有多少 $0$ 來找到 $last\_index$。

由於每當有一個 $0$ 就表示我們會需要額外 $1$ 個空格來放複製出來的 $0$。

所以我們可以有以下設定：
- 我們所需要的空格數量預設為 $0$
- $last\_index$ 預設則為 $m$。

如果我們從 $A[0]$ 開始逐步讀取，每次遇到 $0$ 的時候 $last\_index$ 就會減 $1$。 這個流程會持續到 $last\_index$ 與 $i$ 重疊才會停。

範例：

|$i$|lastIndex|zeroCounter|$A$|
|:--:|:--:|:--:|:--|
|--- | 7 | 0  |$[1,0, 2,3,0,4,5,0]$|
|  0 | 7  | 0  |$[{\color{red}1},0, 2,3,0,4,5,0]$|
|  1 | 6  | 1  |$[1,{\color{red}0}, 2,3,0,4,5,0]$|
|  2 | 6  | 1  |$[1,0, {\color{red}2},3,0,4,5,0]$|
|  3 | 6  | 1  |$[1,0, 2,{\color{red}3},0,4,5,0]$|
|  4 | 5  | 2  |$[1,0, 2,3,{\color{red}0},4,5,0]$|
|  5 | 5  | 2  |$[1,0, 2,3,0,{\color{red}4},5,0]$|

當 $i == last\_index$ 時，我們就找到 $last\_index$ 了。

那如果 $A = [1,0,2,3,0,0,5,0]$ 呢？

|$i$|lastIndex|zeroCounter|$A$|
|:--:|:--:|:--:|:--|
|--- | 7 | 0  |$[1,0,2,3,0,0,5,{\color{green}0}]$|
|  0 | 7  | 0  |$[{\color{red}1},0, 2,3,0,0,5,{\color{green}0}]$|
|  1 | 6  | 1  |$[1,{\color{red}0}, 2,3,0,0,{\color{green}5},0]$|
|  2 | 6  | 1  |$[1,0, {\color{red}2},3,0,0,{\color{green}5},0]$|
|  3 | 6  | 1  |$[1,0, 2,{\color{red}3},0,0,{\color{green}5},0]$|
|  4 | 5  | 2  |$[1,0, 2,3,{\color{red}0},{\color{green}0},5,0]$|
|  5 | 4  | 3  |$[1,0, 2,3,{\color{green}0},{\color{red}0},5,0]$|

與最後結果來比較：

$$A' = [1, 0, 0, 2, 3, 0, 0, 0]$$

從這個範例看到，最後的 $0$ 會讓 $last\_index \lt i$。這是因為這個 $0$ 所複製的 $0$ 會被拋棄。

也就是說，當 $last\_index \lt i$ 發生時，我們就需要將 $A[m] = 0$。而 `zeroCounter` 則需要減 $1$。

所以此時的 $last\_index = 4$ 和 $zeroCounter = 2$，接下來就是利用這些資訊來插入 $0$ 了。

|$last\_index$|zeroCounter|$A$|
|:--:|:--:|:--|
|4   | 2  | $[1,0,2,3,{\color{red}0},0,5,{\color{gray}0}]$ <br><br> 此時將 $A[i + zeroCounter]$ 設為 $A[i]$ 的值。 但因為遇到的是 $0$ 所以需要執行 `zeroCounter - 1`， 並再次將 $A[i + zeroCounter]$ 設為 $A[i]$ 。<br><br> $[1,0,2,3,{\color{red}0},{\color{green}0},{\color{green}0},{\color{gray}0}]$ |
|3   |1   | $[1,0,2,{\color{red}3},0,{\color{gray}0},{\color{gray}0},{\color{gray}0}]$ <br><br> 這次因為遇到的並非為 $0$ 所以只需要將 $A[i + zeroCounter]$ 設為 $A[i]$ 的值即可。 <br><br> $[1,0,2,{\color{red}3},{\color{green}3},{\color{gray}0},{\color{gray}0},{\color{gray}0}]$|
|2   |1   | $[1,0,{\color{red}2},3,{\color{gray}3},{\color{gray}0},{\color{gray}0},{\color{gray}0}]$ <br><br> 以此類推 <br><br> $[1,0,{\color{red}2},{\color{green}2},{\color{gray}3},{\color{gray}0},{\color{gray}0},{\color{gray}0}]$ |

<br>


```kotlin
class Solution {
    fun duplicateZeros(arr: IntArray): Unit {
        var lastIndex = arr.size - 1
        var zeroCounter = 0
        var i = 0

        while(i <= lastIndex) {
            if(arr[i] == 0) {
                lastIndex --
                zeroCounter ++
            }
            i++
        }

        // 這才是正確 i
        i --

        if (lastIndex < i) {
          arr[arr.size - 1] = 0
          zeroCounter --
        }

        while(zeroCounter > 0) {
          if (arr[lastIndex] == 0) {
            arr[lastIndex + zeroCounter] = arr[lastIndex]
            zeroCounter --
            arr[lastIndex + zeroCounter] = arr[lastIndex]
          } else {
            arr[lastIndex + zeroCounter] = arr[lastIndex]
          }

          lastIndex--
        }
    }
}
```

以上的寫法也可以有以下兩種寫法：

```kotlin
class Solution {
    fun duplicateZeros(arr: IntArray): Unit {
        var lastIndex = arr.size - 1
        var zeroCounter = 0
        var i = 0

        while(i <= lastIndex) {
            if(arr[i] == 0) {
                lastIndex --
                zeroCounter ++

                if (lastIndex < i) {
                  arr[arr.size - 1] = 0
                  zeroCounter --
                  break
                } else if (lastIndex == i) {
                  break
                }
            }
            i++
        }

        while(zeroCounter > 0) {
          if (arr[lastIndex] == 0) {
            arr[lastIndex + zeroCounter] = arr[lastIndex]
            zeroCounter --
            arr[lastIndex + zeroCounter] = arr[lastIndex]
          } else {
            arr[lastIndex + zeroCounter] = arr[lastIndex]
          }

          lastIndex--
        }
    }
}
```

或：

```kotlin
class Solution {
    fun duplicateZeros(arr: IntArray): Unit {
        var lastIndex = arr.size - 1
        var zeroCounter = 0
        var i = 0

        while(i <= lastIndex - zeroCounter) {
            if(arr[i] == 0) {
                if (i == lastIndex - zeroCounter) {
                    arr[lastIndex] = 0;
                    lastIndex --;

                    break;
                }
                zeroCounter++;
            }
            i++
        }

        val last = lastIndex - zeroCounter
        for (i in last downTo 0) {
            if (arr[i] == 0) {
                arr[i + zeroCounter] = 0
                zeroCounter --
                arr[i + zeroCounter] = 0
            } else {
                arr[i + zeroCounter] = arr[i]
            }
        }
    }
}
```



<br><br><br><br><br><br><br>
