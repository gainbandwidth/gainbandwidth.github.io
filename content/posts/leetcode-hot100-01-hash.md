---
title: "Hot 100 刷题笔记（一）：哈希表"
date: 2025-01-01
draft: false
categories:
  - LeetCode
tags:
  - LeetCode
  - 哈希表
  - 算法
  - Python
summary: "LeetCode Hot 100 哈希表专题：两数之和、字母异位词分组、最长连续序列。"
ShowToc: true
---

## 哈希表

哈希表是算法面试中最基础也最常用的数据结构之一。核心思想是通过键值对实现快速查找，将时间复杂度从 O(n) 降低到 O(1)。本节涵盖三道经典哈希表题目。

### 两数之和（简单）

使用哈希表存储已遍历过的值及其下标，遇到 `target - x` 已在表中时直接返回结果。

```python
def twoSum(self, nums: List[int], target: int) -> List[int]:
    mp = {}
    for i, x in enumerate(nums):
        if target - x in mp:
            return [mp[target - x], i]
        mp[x] = i
```

### 字母异位词分组（中等）

将每个字符串的字母计数数组转为元组作为哈希表的 key，相同字母组成的字符串会被分到同一组。注意哈希的一些重要函数：`.values()`、`.keys()`、`.items()`。

```python
def groupAnagrams(self, strs: List[str]) -> List[List[str]]:
    mp = defaultdict(list)

    for st in strs:
        counts = [0] * 26
        for ch in st:
            counts[ord(ch) - ord('a')] += 1
        mp[tuple(counts)].append(st)

    return list(mp.values())
```

### 最长连续序列（中等）

将数组放入集合中，只从连续序列的起点（即 `x - 1` 不在集合中的数）开始向后扩展，并加入剪枝优化。

```python
def longestConsecutive(self, nums: List[int]) -> int:
    st = set(nums)
    ans = 0
    for x in nums:
        if x - 1 in st:
            continue
        y = x + 1
        while y in st:
            y += 1
        ans = max(ans, y - x)
        if ans * 2 >= len(nums):
            break
    return ans
```
