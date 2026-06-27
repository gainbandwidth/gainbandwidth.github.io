---
title: "Hot 100 刷题笔记（十四）：排序算法与补充题"
date: 2025-01-14
draft: false
categories: [LeetCode]
tags: [排序, 快速排序, 归并排序, 堆排序, 逆序对, 算法, Python]
summary: "经典排序算法实现（快排、堆排、归并）+ 逆序对、字符串乘法、基本计算器等补充题。"
ShowToc: true
---

本篇整理经典排序算法的手写实现（快速排序、堆排序、归并排序），以及逆序对、字符串乘法、基本计算器等补充题目。

## 快速排序

```python
def quick_sort(nums):
    """
    快速排序（随机基准版，避免最坏情况）
    :param nums: 待排序数组
    :return: 有序数组
    """
    def partition(low, high):
        # 随机选基准（面试可简化为选low/high，但提一句随机基准更优）
        pivot_idx = random.randint(low, high)
        nums[pivot_idx], nums[high] = nums[high], nums[pivot_idx]
        pivot = nums[high]  # 基准放到末尾
        
        # 分区：小于基准的放左边，大于的放右边
        i = low - 1  # 小于基准区的最后一个位置
        for j in range(low, high):
            if nums[j] <= pivot:
                i += 1
                nums[i], nums[j] = nums[j], nums[i]
        # 基准放到正确位置（i+1）
        nums[i+1], nums[high] = nums[high], nums[i+1]
        return i+1

    def sort(low, high):
        if low < high:
            pivot_pos = partition(low, high)
            sort(low, pivot_pos - 1)  # 递归排序左半区
            sort(pivot_pos + 1, high) # 递归排序右半区

    if not nums:
        return []
    import random  # 面试时可提前导入
    sort(0, len(nums)-1)
    return nums
```

## 堆排序

```python
def heap_sort(nums):
    """
    堆排序（大顶堆实现）
    :param nums: 待排序数组
    :return: 有序数组
    """
    def heapify(arr, n, i):
        """
        调整大顶堆（以i为根节点）
        :param arr: 堆数组
        :param n: 堆的大小
        :param i: 根节点索引
        """
        largest = i  # 初始化最大元素为根
        left = 2 * i + 1  # 左子节点
        right = 2 * i + 2 # 右子节点

        # 找最大元素（根/左/右）
        if left < n and arr[left] > arr[largest]:
            largest = left
        if right < n and arr[right] > arr[largest]:
            largest = right

        # 若最大元素不是根，交换并递归调整
        if largest != i:
            arr[i], arr[largest] = arr[largest], arr[i]
            heapify(arr, n, largest)

    if not nums:
        return []
    n = len(nums)

    # 1. 构建大顶堆（从最后一个非叶子节点开始）
    for i in range(n // 2 - 1, -1, -1):
        heapify(nums, n, i)

    # 2. 逐个取出堆顶元素（最大），放到数组末尾
    for i in range(n-1, 0, -1):
        nums[i], nums[0] = nums[0], nums[i]  # 堆顶与当前最后元素交换
        heapify(nums, i, 0)  # 调整剩余元素为大顶堆

    return nums
```

## 归并排序

```python
def merge_sort(nums):
    """
    归并排序（分治+合并有序数组）
    :param nums: 待排序数组
    :return: 有序数组
    """
    def merge(left, right):
        """合并两个有序数组"""
        res = []
        i = j = 0
        # 双指针合并
        while i < len(left) and j < len(right):
            if left[i] <= right[j]:
                res.append(left[i])
                i += 1
            else:
                res.append(right[j])
                j += 1
        # 追加剩余元素
        res.extend(left[i:])
        res.extend(right[j:])
        return res

    # 递归终止条件：数组长度≤1
    if len(nums) <= 1:
        return nums
    # 分治：拆分为左右子数组
    mid = len(nums) // 2
    left_sorted = merge_sort(nums[:mid])
    right_sorted = merge_sort(nums[mid:])
    # 合并
    return merge(left_sorted, right_sorted)
```

## 交易逆序对的总数

归并排序（分治思想）/ 线段树。

```python
def reversePairs(self, record):
    def merge_sort(l, r):
        if l >= r: return 0
        m = (l + r) // 2
        res = merge_sort(l, m) + merge_sort(m + 1, r)
        i, j = l, m + 1
        tmp[l:r + 1] = record[l:r + 1]
        for k in range(l, r + 1):
            if i == m + 1:
                record[k] = tmp[j]
                j += 1
            elif j == r + 1 or tmp[i] <= tmp[j]:
                record[k] = tmp[i]
                i += 1
            else:
                record[k] = tmp[j]
                j += 1
                res += m - i + 1 # 统计逆序对
        return res
    tmp = [0] * len(record)
    return merge_sort(0, len(record) - 1)
```

## 字符串转换整数（atoi）

```python
def myAtoi(self, s: str) -> int:
    n = len(s)
    i = 0
    while i < n and s[i] == ' ':
        i += 1
    sign = 1
    if i < n and s[i] in "+-":
        sign = 1 if s[i] == '+' else -1
        i += 1

    MX = (1 << 31) - 1
    num = 0
    while i < n and '0' <= s[i] <= '9':
        num = 10 * num + int(s[i])
        if num > MX:
            return MX if sign > 0 else -(1 << 31)
        i += 1

    return sign * num
```

## 比较版本号

```python
def compareVersion(self, version1: str, version2: str) -> int:
    a = map(int, version1.split('.'))
    b = map(int, version2.split('.'))
    for ver1, ver2 in zip_longest(a, b, fillvalue=0):
        if ver1 != ver2:
            return -1 if ver1 < ver2 else 1
    return 0
```

## 字符串相乘

字符串相加 / 做乘法 / FFT。

```python
def multiply(self, num1: str, num2: str) -> str:
    if num1 == "0" or num2 == "0":
        return "0"
    m, n = len(num1), len(num2)
    ans = [0] * (m + n)
    for i in range(m - 1, -1, -1):
        x = int(num1[i])
        for j in range(n - 1, -1, -1):
            ans[i + j + 1] += x * int(num2[j])

    for i in range(m + n - 1, 0, -1):
        ans[i - 1] += ans[i] // 10
        ans[i] %= 10

    index = 1 if ans[0] == 0 else 0
    ans = "".join(str(x) for x in ans[index:])
    return ans
```

## 分发糖果

```python
def candy(self, ratings: List[int]) -> int:
    # 前后缀分解（更容易理解）
    n = len(ratings)
    left = [1] * n
    right = [1] * n
    # 从左至右遍历
    for i in range(1, n):
        if ratings[i] > ratings[i - 1]:
            left[i] = left[i - 1] + 1
    ans = left[-1]
    # 从右至左遍历
    for i in range(n - 2, -1, -1):
        if ratings[i] > ratings[i + 1]:
            right[i] = right[i + 1] + 1
        ans += max(left[i], right[i])
    return ans
    
    # 分组循环（只需要 O(1) 额外空间）
    # 外层循环负责遍历组之前的准备工作（记录开始位置），和遍历组之后的统计工作（更新答案）。
    # 内层循环负责遍历组，找出这一组最远在哪结束。
    ans = n = len(ratings)
    i = 0
    while i < n:
        # i-1 前一组的最后一个数（谷底会被两组共享）
        start = i - 1 if i > 0 and ratings[i - 1] < ratings[i] else i

        # 找严格递增段
        while i + 1 < n and ratings[i] < ratings[i + 1]:
            i += 1
        top = i # 峰顶
        # 找严格递减段
        while i + 1 < n and ratings[i] > ratings[i + 1]:
            i += 1

        inc = top - start # 从 start 到 top−1
        dec = i - top # 从 top+1 到 i 
        ans += (inc * (inc - 1) + dec * (dec - 1)) // 2 + max(inc, dec)
        i += 1

    return ans
```

## 基本计算器

```python
def calculate(self, s: str) -> int:
    ops = [1]
    sign = 1

    ret = 0
    n = len(s)
    i = 0
    while i < n:
        if s[i] == ' ':
            i += 1
        elif s[i] == '+':
            sign = ops[-1]
            i += 1
        elif s[i] == '-':
            sign = -ops[-1]
            i += 1
        elif s[i] == '(':
            ops.append(sign)
            i += 1
        elif s[i] == ')':
            ops.pop()
            i += 1
        else:
            num = 0
            while i < n and s[i].isdigit():
                num = num * 10 + ord(s[i]) - ord('0')
                i += 1
            ret += num * sign

    return ret
```

## 反转字符串中的单词

```python
def reverseWords(self, s):
    s = s.strip() # 删除首尾空格
    # 分割字符串函数 strs = s.split()
    i = j = len(s) - 1
    res = []
    while i >= 0:
        while i >= 0 and s[i] != ' ': # 搜索首个空格
            i -= 1
        res.append(s[i + 1: j + 1]) # 添加单词
        while i >= 0 and s[i] == ' ': # 跳过单词间空格
            i -= 1
        j = i
    return ' '.join(res)
```
