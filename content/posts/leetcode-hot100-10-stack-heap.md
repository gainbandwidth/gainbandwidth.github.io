---
title: "Hot 100 刷题笔记（十）：栈与堆"
date: 2025-01-10
draft: false
categories:
  - LeetCode
tags:
  - LeetCode
  - 栈
  - 堆
  - 单调栈
  - TopK
  - 算法
  - Python
summary: "LeetCode Hot 100 栈与堆专题：最小栈、每日温度、柱状图最大矩形、第 K 大元素、数据流中位数。"
ShowToc: true
---

## 栈

本文整理 LeetCode Hot 100 中栈与堆相关的经典题目，包括单调栈、最小栈、柱状图最大矩形，以及堆相关的 Top-K 问题和数据流中位数。

有效的括号 简单（字典存储括号对）

**最小栈 中等**

```python
class MinStack:
    def __init__(self):
        # 栈中除了保存添加的元素，还保存前缀最小值
        # 初始化的时候，在栈底加一个 ∞ 哨兵，作为 preMin[−1]
        self.st = [(0, inf)]       

    def push(self, val: int) -> None:
        # preMin[i]=min(preMin[i−1],nums[i])
        self.st.append((val, min(self.st[-1][1], val)))       

    def pop(self) -> None:
        self.st.pop()       

    def top(self) -> int:
        return self.st[-1][0]

    def getMin(self) -> int:
        return self.st[-1][1]
```

字符串解码 中等（栈 / 递归）

```python
def decodeString(self, s: str) -> str:
    # 栈
    stack, res, multi = [], "", 0
    for c in s:
        if c == '[':
            stack.append([multi, res])
            res, multi = "", 0
        elif c == ']':
            cur_multi, last_res = stack.pop()
            res = last_res + cur_multi * res
        elif '0' <= c <= '9':
            multi = multi * 10 + int(c)
        else:
            res += c
    return res
    
    # 递归
    def dfs(s, i):
        res, multi = "", 0
        while i < len(s):
            if '0' <= s[i] <= '9':
                multi = multi * 10 + int(s[i])
            elif s[i] == '[':
                i, tmp = dfs(s, i + 1)
                res += multi * tmp
                multi = 0
            elif s[i] == ']':
                return i, res
            else:
                res += s[i]
            i += 1
        return res

    return dfs(s, 0)
```

**每日温度 中等**（单调栈两种写法：及时去掉无用数据，保证栈中数据有序）

```python
def dailyTemperatures(self, temperatures: List[int]) -> List[int]:
    n = len(temperatures)
    ans = [0] * n
    st = []
    # 从右向左
    for i in range(n-1, -1, -1):
        while st and temperatures[st[-1]] <= temperatures[i]:
            st.pop()
        if st:
            ans[i] = st[-1] - i
        st.append(i)
    # 从左往右
    for i, t in enumerate(temperatures):
        while st and temperatures[st[-1]] < t:
            j = st.pop()
            ans[j] = i - j
        st.append(i)
    return ans
```

**柱状图中最大的矩形 困难**

```python
def largestRectangleArea(self, heights: List[int]) -> int:
    # 单调栈
    # 在一维数组中对每一个数找到第一个比自己小的元素
    n = len(heights)
    left = [-1] * n
    right = [n] * n
    s = []
    for i, h in enumerate(heights):
        while s and heights[s[-1]] > h:
            j = s.pop()
            right[j] = i
        if s:
            left[i] = s[-1]
        s.append(i)
    ans = max(heights[i] * (right[i] - left[i] - 1) for i in range(n))
    return ans
```

（补充）移调 k 位数字（单调栈，有不少坑）

```python
def removeKdigits(self, num: str, k: int) -> str:
    s = []
    # 构建单调递增的数字串
    for digit in num:
        while k and s and s[-1] > digit:
            s.pop()
            k -= 1
        s.append(digit)
    # 如果 K > 0，删除末尾的 K 个字符
    s = s[:-k] if k else s
    # 抹去前导零
    return "".join(s).lstrip('0') or "0"
```

（补充）用栈实现队列

```python
class MyQueue:
    def __init__(self):
        self.s1 = []
        self.s2 = []      

    def push(self, x: int) -> None:
        self.s1.append(x)      

    # 删除头元素并返回
    def pop(self) -> int:
        # 先调用 peek 保证 s2 非空
        self.peek()
        return self.s2.pop()

    # 返回队头元素
    def peek(self) -> int:
        if not self.s2:
            while self.s1:
                self.s2.append(self.s1.pop())
        return self.s2[-1]
        
    def empty(self) -> bool:
        return not self.s1 and not self.s2
```

## 堆

数组中的第K个最大元素 中等（快速选择，分治 / 小顶堆实现，超出个数即删除）

```python
def findKthLargest(self, nums: List[int], k: int) -> int:
    # 基准数的索引正好是 N−k ，则意味着它就是第 k 大的数字。此时直接返回，无需继续递归
    def quickselect(nums, k):
        pivot = random.choice(nums)
        big, equal, small = [], [], []
        for num in nums:
            if num > pivot:
                big.append(num)
            elif num < pivot:
                small.append(num)
            else:
                equal.append(num)
        if k <= len(big):
            # 第 k 大元素在 big 中，递归划分
            return quickselect(big, k)
        if len(nums) - len(small) < k:
            # 第 k 大元素在 small 中，递归划分
            return quickselect(small, k - len(nums) + len(small))
        # 第 k 大元素在 equal 中，直接返回 pivot
        return pivot
    return quickselect(nums, k)

def findKthLargest(self, nums, k):
    # 小顶堆，维护所有元素
    min_heap = []
    for num in nums:
        heapq.heappush(min_heap, num)
    for i in range(len(nums) - k):
        heapq.heappop(min_heap)
    return min_heap[0]
    # Top-K 问题，小顶堆，维护 k 个元素
    min_heap = []
    for num in nums:
        heapq.heappush(min_heap, num)
        if len(min_heap) > k:
            heapq.heappop(min_heap)
    return min_heap[0]
```

前 K 个高频元素 中等

```python
def topKFrequent(self, nums: List[int], k: int) -> List[int]:
    # 1. 桶排序
    # 第一步：统计每个元素的出现次数
    cnt = Counter(nums)
    max_cnt = max(cnt.values())
    # 第二步：把出现次数相同的元素，放到同一个桶中
    buckets = [[] for _ in range(max_cnt + 1)]
    for x, c in cnt.items():
        buckets[c].append(x)
    # 第三步：倒序遍历 buckets，把出现次数前 k 大的元素加入答案
    ans = []
    for bucket in reversed(buckets):
        ans += bucket
        if len(ans) == k:
            return ans

    # 2. 堆
    mp = defaultdict(int)
    for num in nums:
        mp[num] += 1

    minheap = []
    for num, freq in mp.items():
        heappush(minheap, (freq, num))
        if len(minheap) > k:
            heappop(minheap)

    return [num for _, num in minheap]
```

**数据流的中位数 困难**（数据分左右两组，一个大顶堆、一个小顶堆，大顶堆通过负数实现）

```python
class MedianFinder:
    # 保证 left 的大小和 right 的大小尽量相等。规定：在有奇数个数时，left 比 right 多 1 个数。
    # 保证 left 的所有元素都小于等于 right 的所有元素。
    def __init__(self):
        self.left = []
        self.right = []

    def addNum(self, num: int) -> None:
        if len(self.left) == len(self.right):
            # heappush(self.left, -heappushpop(self.right, num))
            heappush(self.right, num)
            heappush(self.left, -heappop(self.right))
        else:
            # heappush(self.right, -heappushpop(self.left, -num))
            heappush(self.left, -num)
            heappush(self.right, -heappop(self.left))

    def findMedian(self) -> float:
        if len(self.left) > len(self.right):
            return -self.left[0]
        return (self.right[0] - self.left[0]) / 2
```
