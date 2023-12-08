---
layout: post
title: "Leet 278: First Bad Version"
date: 2023-12-04 20:07:53 +0800
use_math: true
categories: [LeetCode, Algorithm, Easy Algorithm]
keywords: algorithm
---

# 題目

[278. First Bad Version](https://leetcode.com/problems/first-bad-version)

You are a product manager and currently leading a team to develop a new product. Unfortunately, the latest version of your product fails the quality check. Since each version is developed based on the previous version, all the versions after a bad version are also bad.

Suppose you have n versions `[1, 2, ..., n]` and you want to find out the first bad one, which causes all the following ones to be bad.

You are given an API bool isBadVersion(version) which returns whether version is bad. Implement a function to find the first bad version.

> <b>You should minimize the number of calls to the API.</b>

```
Example 1:

Input: n = 5, bad = 4
Output: 4
Explanation:
call isBadVersion(3) -> false
call isBadVersion(5) -> true
call isBadVersion(4) -> true
Then 4 is the first bad version.


Example 2:

Input: n = 1, bad = 1
Output: 1
```

# 暸解題目

首先，從題目我們知道，版本的存放可以視為 Array 結構。

我的第一個想法應該就是使用 **Linear Search** :

1. 先查看當下 `n` 的 `isBadVersion(n)` 是否為 `true`
2. 如果是 `true` 那就往`n - 1` 或左走直到遇到 `isBadVersion(m) == false` 並回傳 `n + 1`
3. 否則就先要往 `n + 1` 或右走，直到遇到 `isBadVersion(m) == true` 並回傳 `n`

```java
/* The isBadVersion API is defined in the parent class VersionControl.
      boolean isBadVersion(int version); */

public class Solution extends VersionControl {
    // You should minimize the number of calls to the API.
    public int firstBadVersion(int n) {

        boolean isInitBadVersion = isBadVersion(n);
        int step = isInitBadVersion ? -1 : 1;
        int finalIndex = n + step;

        while (isBadVersion(finalIndex) == isInitBadVersion) {
            finalIndex += step;
        }

        return isInitBadVersion ? finalIndex + 1 : finalIndex;
    }
}
```

但這種寫法其實會很耗時。 其中一個範例就是當 :

```
n = 2126753390
bad = 1702766719
```

Leetcode 便會嫌慢了。所以我們需要更快的演算法。

由於這個題目的資料結構只是 Array， 而且我們只需要找到 `true` 與 `false` 的臨界點。

```
[<-- 皆是 False --- F, T --- 皆是 True --->]
```

所以我們可以把這個想成是已排序好的 Array， 也就是說這時我們就可以使用 **Binary Search**。

## Binary Search

Binary Search 的概念很單純，我們只是不斷地將資料減半直到找到我們要找的即可。

> 問題在 : 如何減半？

假設我們有以下資料 :

```
[ F, T, T, T, T, T, T, T, T ]
```

當給我們的資料是 : `input = 8 ; bad = 1` 時， 我們可以把資料分成兩份 :

```
[ F, T, T, T, T, T, T, T, T ]
<-----------------------|--->
  [0 ... 7]               [8]
```

因為 `isBadVersion(8) == true`，所以我們只需要看 `[0 ... 7]` 的部分。

此時我們會從 [0 , 7] 取得中間值 : 4 並查看 `isBadVersion(4)`。

通過這個行為，我們便可以將資料減半 :

```
[ F, T, T, T, T, T, T, T, T ]
<-----------|---------->|
[0, 3]          [4, 7]
```

由於 `isBadVersion(4) == true` 所以我們知道 `bad` 會在左邊，也就是 [0, 3]。

我們再次取得 [0, 3] 的中間值 : 2

```
[ F, T, T, T, T, T, T, T , T ]
<--------|->|
[0, 2]   [3, 3]
```

然後當我們再次將 [0, 2] 分成兩份 [0, 1] 與 [2, 2]。 當然很顯然我們會再次將 [0, 1] 分成 [0] 與 [1]。最後便會找到 `bad` 的位置為 `1`。

相較於線性搜尋法， 二分搜尋法成功地將步數減少一半。 他們的空間複雜度是 :

| Space Complexity | Best   | Average     | Worst       |
| :--------------- | :----- | :---------- | :---------- |
| 線性搜尋法       | $O(n)$ | $O(n)$      | $O(n)$      |
| 二分搜尋法       | $O(1)$ | $O(log(n))$ | $O(log(n))$ |

## 實作

```java
/* The isBadVersion API is defined in the parent class VersionControl.
      boolean isBadVersion(int version); */

public class Solution extends VersionControl {
    // You should minimize the number of calls to the API.
    public int firstBadVersion(int n) {
        boolean isInitBadVersion = isBadVersion(n);

        if (isInitBadVersion && !isBadVersion(n - 1)) return n;

        int start = 0;
        int end = n - 1

        return performBinarySearch(start, end)
    }

    public int performBinarySearch(int start, int end) {
        int mid = (start + end) / 2 ; // start + (end - start) / 2;
        boolean isMidBadVersion = isBadVersion(mid);
        boolean isLeftBadVersion = isBadVersion(mid - 1);
        boolean isRightBadVersion = isBadVersion(mid + 1);

        // we find the index when the adjacent version has opposite index
        if (isMidBadVersion && !isLeftBadVersion) {
            return mid;
        } else if (!isMidBadVersion && isRightBadVersion) {
            return mid + 1;
        }

        if (isMidBadVersion) {
            return performBinarySearch(start, mid - 1);
        } else {
            return performBinarySearch(mid, end);
        }
    }
}
```

### Stackoverflow

如果我們 submit 以上的程式時，在

```
input = 2126753390
bad = 1702766719
```

的情況下，Leetcode 會出現 **Time Limit Exceeded**。

> 你能看出問題出在哪嗎？

如果我們將 `mid` 的值列出來，你會發現 :

```
n = 2126753390;	max = 2147483647

mid = 1063376694
mid = -552418606
mid = 787167391
mid = -690523258...
```

這是因為當我們進行 `(start + end)` 時，結果超過 `Integer.MAX_VALUE`，對 32 bits 而言 Signed Integer 最大值就是 2_147_483_647。

想要避免這個問題，我們有兩種方法 :

```java
int mid = start / 2 + end / 2;
// 或一般寫法
int mid = start + (end - start) / 2;
```

最後結果變成 :

```java
/* The isBadVersion API is defined in the parent class VersionControl.
      boolean isBadVersion(int version); */

public class Solution extends VersionControl {
    // You should minimize the number of calls to the API.
    public int firstBadVersion(int n) {
        boolean isInitBadVersion = isBadVersion(n);

        if (isInitBadVersion && !isBadVersion(n - 1)) return n;

        int start = 0;
        int end = n - 1

        return performBinarySearch(start, end)
    }

    public int performBinarySearch(int start, int end) {
        int mid = start + (end - start) / 2;
        boolean isMidBadVersion = isBadVersion(mid);
        boolean isLeftBadVersion = isBadVersion(mid - 1);
        boolean isRightBadVersion = isBadVersion(mid + 1);

        // we find the index when the adjacent version has opposite index
        if (isMidBadVersion && !isLeftBadVersion) {
            return mid;
        } else if (!isMidBadVersion && isRightBadVersion) {
            return mid + 1;
        }

        if (isMidBadVersion) {
            return performBinarySearch(start, mid - 1);
        } else {
            return performBinarySearch(mid, end);
        }
    }
}
```

|             |   值    | %      |
| :---------- | :-----: | ------ |
| **Runtime** |  29ms   | 9.25%  |
| **Memory**  | 38.58MB | 97.79% |

上面是使用遞迴的寫法，我們可以將他改為 loop 的寫法 :

```java
/* The isBadVersion API is defined in the parent class VersionControl.
      boolean isBadVersion(int version); */

public class Solution extends VersionControl {
    // You should minimize the number of calls to the API.
    public int firstBadVersion(int n) {
        boolean isInitBadVersion = isBadVersion(n);

        if (isInitBadVersion && !isBadVersion(n - 1)) return n;

        int start = 0;
        int end = n - 1

        return performBinarySearch(start, end)
    }

    public int performBinarySearch(int start, int end) {
        int mid = start + (end - start) / 2;
        boolean isMidBadVersion = isBadVersion(mid);
        boolean isLeftBadVersion = isBadVersion(mid - 1);
        boolean isRightBadVersion = isBadVersion(mid + 1);

        while (!((isMidBadVersion && !isLeftBadVersion) || (!isMidBadVersion && isRightBadVersion))) {
            if (isMidBadVersion) {
                end = mid - 1;
            } else {
                start = mid;
            }

            mid = start + (end - start) / 2;

            isMidBadVersion = isBadVersion(mid);
            isLeftBadVersion = isBadVersion(mid - 1);
            isRightBadVersion = isBadVersion(mid + 1);
        }

        // we find the index when the adjacent version has opposite index
        if (isMidBadVersion) {
            return mid;
        } else {
            return mid + 1;
        }
    }
}
```

|             |   值    | %      |
| :---------- | :-----: | ------ |
| **Runtime** |  41ms   | 9.25%  |
| **Memory**  | 39.17MB | 49.39% |

### 優化

看看那可悲的 Runtime 我的程式必須要好好進行修改。尤其是以下幾點 :

其一，我們上面寫得實在是太冗長了。
其二，我們調用 `isBadVersion` 太多次了 (每個 Loop 都調用 3 次)

所以我們直接參考大神的吧 XD ~

```java
/* The isBadVersion API is defined in the parent class VersionControl.
      boolean isBadVersion(int version); */

public class Solution extends VersionControl {
    // You should minimize the number of calls to the API.
    public int firstBadVersion(int n) {
        // 因為如果 n = 1，那 [0] 必須是 false
        if (n == 1) return 1;

        return performBinarySearch(0, n - 1)
    }

    public int performBinarySearch(int start, int end) {

        while (start <= end) {
            int mid = start + (end - start) / 2;
            if (isBadVersion(mid)) {
                end = mid - 1;
            } else {
                start = mid + 1;
            }
        }

        return start;
    }
}
```

|             |   值    | %      |
| :---------- | :-----: | ------ |
| **Runtime** |  13ms   | 83.56% |
| **Memory**  | 39.38MB | 31.63% |

快多了吧。

> Memory 的部分因為挺隨機的，所以就忽略吧。

<br><br><br><br><br><br><br>
