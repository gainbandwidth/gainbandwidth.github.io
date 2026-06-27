---
title: "Hot 100 刷题笔记（十一）：贪心算法"
date: 2025-01-11
draft: false
categories: [LeetCode]
tags: [贪心, 买卖股票, 跳跃游戏, 算法, Python]
summary: "LeetCode Hot 100 贪心专题：买卖股票系列（一次/多次/冷冻期/k 次）、跳跃游戏、划分字母区间。"
ShowToc: true
---

本篇整理 LeetCode Hot 100 中贪心算法相关的经典题目，涵盖买卖股票系列的多种变体、跳跃游戏以及划分字母区间等问题。

## 买卖股票的最佳时机

### 一、一次买卖（贪心）

### 二、多次买卖（贪心）——进阶 含冷冻期（初始化和递推有变化）

```python
def maxProfit(self, prices: List[int]) -> int:
    n = len(prices)
    f = [[0] * 2 for _ in range(n + 1)]
    f[0][1] = -inf
    for i, p in enumerate(prices):
        f[i + 1][0] = max(f[i][0], f[i][1] + p)
        f[i + 1][1] = max(f[i][1], f[i][0] - p)
    return f[n][0]
# 冷冻期
def maxProfit(self, prices: List[int]) -> int:
    n = len(prices)
    dp = [[0] * 2 for _ in range(n + 1)]
    dp[0][1] = -inf
    dp[1][1] = -prices[0]
    for i in range(2, n + 1):
        dp[i][0] = max(dp[i-1][0], dp[i-1][1] + prices[i-1])
        # 冷冻期体现在 dp[i-2][0]
        dp[i][1] = max(dp[i-1][1], dp[i-2][0] - prices[i-1])
    return dp[n][0]
```

### 三、最多两笔（动态规划）

### 四、k 次交易（动态规划）

```python
def maxProfit(self, k: int, prices: List[int]) -> int:
    n = len(prices)
    f = [[[-inf] * 2 for _ in range(k + 2)] for _ in range(n + 1)]
    for j in range(1, k + 2):
        f[0][j][0] = 0

    for i in range(1, n + 1):
        for j in range(1, k + 2):
            f[i][j][0] = max(f[i-1][j][0], f[i-1][j-1][1] + prices[i-1])
            f[i][j][1] = max(f[i-1][j][1], f[i-1][j][0] - prices[i-1])
    return f[n][k+1][0]
    
    # 空间优化
    f = [[-inf] * 2 for _ in range(k + 2)]
    for j in range(1, k + 2):
        f[j][0] = 0
    for p in prices:
        for j in range(k + 1, 0, -1):
            f[j][0] = max(f[j][0], f[j][1] + p)
            f[j][1] = max(f[j][1], f[j-1][0] - p)
    return f[-1][0]

    # 恰好 k 次
    f[0][1][0] = 0  # 只需改这里
    for i, p in enumerate(prices):
        for j in range(1, k + 2):
            f[i + 1][j][0] = max(f[i][j][0], f[i][j][1] + p)
            f[i + 1][j][1] = max(f[i][j][1], f[i][j - 1][0] - p)
    return f[-1][-1][0]

    # 至少 k 次
    f[0][0][0] = 0
    for i, p in enumerate(prices):
        f[i + 1][0][0] = max(f[i][0][0], f[i][0][1] + p)
        f[i + 1][0][1] = max(f[i][0][1], f[i][0][0] - p)  # 无限次
        for j in range(1, k + 1):
            f[i + 1][j][0] = max(f[i][j][0], f[i][j][1] + p)
            f[i + 1][j][1] = max(f[i][j][1], f[i][j - 1][0] - p)
    return f[-1][-1][0]
```

## 跳跃游戏（中等）

```python
def canJump(self, nums: List[int]) -> bool:
    n = len(nums)
    rightmost = 0
    for i in range(n):
        if i <= rightmost:
            rightmost = max(rightmost, i + nums[i])
    return rightmost >= n - 1
```

## 跳跃游戏 II（中等）

判断每一轮最远距离 `i == end`。

```python
def jump(self, nums: List[int]) -> int:
    # 判断每一轮最远位置
    n = len(nums)
    maxPos, end, step = 0, 0, 0
    for i in range(n - 1):
        if maxPos >= i: # 鲁棒性
            maxPos = max(maxPos, i + nums[i])
            if i == end:
                end = maxPos
                step += 1
    return step
    # 二重循环
    n = len(nums)
    rightmost = 0
    cnt = 0
    while rightmost < n - 1:
        for i in range(rightmost + 1):
            rightmost = max(rightmost, i + nums[i])
        cnt += 1
    return cnt
```

## 划分字母区间（中等）

判断每一轮最远位置 `i == end`。

```python
def partitionLabels(self, s: str) -> List[int]:
    last = [0] * 26
    for i, ch in enumerate(s):
        last[ord(ch) - ord('a')] = i

    partition = []
    start = end = 0
    for i, ch in enumerate(s):
        end = max(end, last[ord(ch) - ord('a')])
        if i == end:
            partition.append(end - start + 1)
            start = end + 1

    return partition
```
