---
title: "Hot 100 刷题笔记（九）：二分查找"
date: 2025-01-09
draft: false
categories:
  - LeetCode
tags:
  - LeetCode
  - 二分查找
  - 旋转数组
  - 算法
  - Python
summary: "LeetCode Hot 100 二分查找专题：lower_bound 模板、搜索旋转排序数组、寻找两个正序数组的中位数。"
ShowToc: true
---

## 二分查找

本文整理 LeetCode Hot 100 中二分查找相关题目，从基础模板到经典变体，包括 lower_bound、旋转数组搜索以及寻找两个正序数组的中位数。

二分基础模板，>= x 问题

```python
# 左闭右闭区间，>= x 问题
# > x 转换为 >= x + 1
# < x 转换为 (>=x) - 1
# <= x 转换为 (>x) - 1 转换为 (>= x + 1) - 1
# nums 是非递减的，返回最小的满足 nums[i] >= target 的i，如果不存在，返回 len(nums)
def lower_bound(nums, target):
    left = 0
    right = len(nums) - 1
    while left <= right:
        mid = (left + right) // 2
        if nums[mid] >= target:
            right = mid - 1
        else:
            left = mid + 1
    return left
```

二分 `check` 条件模板——最大满足条件值

```python
while left <= right:
	mid = (left + right) // 2
	if check(mid):
		left = mid + 1
	else:
		right = mid - 1
	return right
```

搜索插入位置 简单

搜索二维矩阵 中等

```python
def searchMatrix(self, matrix: List[List[int]], target: int) -> bool:
    # a[i] = matrix[⌊i/n⌋][i mod n]
    m, n = len(matrix), len(matrix[0])
    left, right = 0, m * n - 1
    while left <= right:
        mid = (left + right) // 2
        x = matrix[mid // n][mid % n]
        if x == target:
            return True
        elif x < target:
            left = mid + 1
        else:
            right = mid - 1
    return False
```

在排序数组中查找元素的第一个和最后一个位置 中等

```python
# 活用上面的 lower_bound
def searchRange(self, nums: List[int], target: int) -> List[int]:
    start = lower_bound(nums, target) # >= x
    if start >= len(nums) or nums[start] != target:
        return [-1, -1]
    end = lower_bound(nums, target + 1) - 1 # <= x
    return [start, end]
```

**搜索旋转排序数组 中等**

```python
def search(self, nums: List[int], target: int) -> int:
    # 对于任意一个index，其左区间和右区间至少有一个是有序的，那么就可以根据这个区间的最大值和最小值来判断Target是否在该区间内，由此就可以确定新的查找区间为有序半区还是无序半区
    if not nums:
        return -1
    l, r = 0, len(nums) - 1
    while l <= r:
        mid = (l + r) // 2
        if nums[mid] == target:
            return mid
        if nums[0] <= nums[mid]:
            if nums[0] <= target < nums[mid]:
                r = mid - 1
            else:
                l = mid + 1
        else:
            if nums[mid] < target <= nums[-1]:
                l = mid + 1
            else:
                r = mid - 1
    return -1
```

寻找旋转排序数组中的最小值 中等

```python
def findMin(self, nums: List[int]) -> int:
    l, r = 0, len(nums) - 1
    while l <= r:
        m = (l + r) // 2
        if nums[m] <= nums[-1]:
            r = m - 1
        else:
            l = m + 1
    return nums[l]
```

**寻找两个正序数组的中位数 困难**（最难的一题）

```python
def findMedianSortedArrays(self, a: List[int], b: List[int]) -> float:
    # 查找第 k 小的数，其中 k=⌈(m + n)/2⌉
    def getKthElement(k):
        index1, index2 = 0, 0
        while True:
            if index1 == m:
                return b[index2 + k - 1]
            if index2 == n:
                return a[index1 + k - 1]
            if k == 1:
                return min(a[index1], b[index2])

            newIndex1 = min(index1 + k // 2 - 1, m - 1)
            newIndex2 = min(index2 + k // 2 - 1, n - 1)
            pivot1, pivot2 = a[newIndex1], b[newIndex2]
            if pivot1 <= pivot2:
                k -= newIndex1 - index1 + 1
                index1 = newIndex1 + 1
            else:
                k -= newIndex2 - index2 + 1
                index2 = newIndex2 + 1

    m, n = len(a), len(b)
    totalLength = m + n
    if totalLength % 2 == 1:
        return getKthElement((totalLength + 1) // 2)
    else:
        return (getKthElement(totalLength // 2) + getKthElement(totalLength // 2 + 1)) / 2
```
