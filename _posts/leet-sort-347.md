---
layout: post
title:  "LeetCode Search - 374. Guess Number Higher or Lower"
date:   2022-11-14 15:27:40 +08:00
use_math: true
categories: [leetCode, search, easy]
---

### 374. Guess Number Higher or Lower
We are playing the Guess Game. The game is as follows:

I pick a number from 1 to n. You have to guess which number I picked.

Every time you guess wrong, I will tell you whether the number I picked is higher or lower than your guess.

You call a pre-defined API int guess (int num), which returns three possible results:

- **-1**: Your guess is higher than the number I picked (i.e. **num > pick**).
- **1**: Your guess is lower than the number I picked (i.e. **num < pick**).
- **0**: your guess is equal to the number I picked (i.e. **num == pick**).

Return the number that I picked.

### Approach
Obviously, we need to perform a **Binary Search** from 1 to n.
