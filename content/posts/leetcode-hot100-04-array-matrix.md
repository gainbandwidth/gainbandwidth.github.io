---
title: "Hot 100 刷题笔记（四）：数组与矩阵"
date: 2025-01-04
draft: false
categories:
  - LeetCode
tags:
  - LeetCode
  - 数组
  - 矩阵
  - 前缀和
  - 算法
  - Python
summary: "LeetCode Hot 100 数组与矩阵专题：最大子数组和、合并区间、螺旋矩阵、旋转图像等。"
ShowToc: true
---

## 普通数组

数组是最基础的数据结构，但其中蕴含的算法思想非常丰富——分治、动态规划、前缀和、原地哈希等，都需要扎实掌握。

### 最大子数组和（中等）

给出两种解法：前缀和与动态规划。前缀和思路是维护当前前缀和与历史最小前缀和的差值。

```python
def maxSubArray(self, nums):
    # 前缀和
    ans = -inf
    min_pre = pre = 0
    for x in nums:
        pre += x
        ans = max(ans, pre - min_pre)
        min_pre = min(min_pre, pre)
    return ans
    # 动态规划
    ans = -inf
    f = 0
    for x in nums:
        f = max(f, 0) + x
        ans = max(ans, f)
    return ans
```

### 合并区间（中等）

先按区间左端点排序，再逐个合并重叠区间。注意排序函数的使用：`key=lambda x: x[0]`。

```python
def merge(self, intervals: List[List[int]]) -> List[List[int]]:
    intervals.sort(key=lambda p: p[0])
    ans = []
    for p in intervals:
        if ans and p[0] <= ans[-1][1]:
            ans[-1][1] = max(ans[-1][1], p[1])
        else:
            ans.append(p)
    return ans
```

### 轮转数组（中等）

通过三次反转实现原地轮转：先反转整个数组，再反转前 k 个，最后反转剩余部分。

```python
def rotate(self, nums: List[int], k: int) -> None:
    # 原地做法：反转数组
    def reverse(i, j):
        while i < j:
            nums[i], nums[j] = nums[j], nums[i]
            i += 1
            j -= 1
    n = len(nums)
    k %= n
    reverse(0, n - 1)
    reverse(0, k - 1)
    reverse(k, n - 1)
```

### 除了自身以外数组的乘积（中等）

利用前缀积和后缀积的思想，不使用除法即可在 O(n) 时间内完成。

### 缺失的第一个正数（困难）

原地哈希的经典题目：将每个正整数放到它应该在的位置上（即 `nums[i]` 放到下标 `nums[i] - 1`），然后扫描找出第一个位置不匹配的数。

```python
def firstMissingPositive(self, nums: List[int]) -> int:
  	n = len(nums)
    for i in range(n):
         # 如果当前学生的学号在 [1,n] 中，但（真身）没有坐在正确的座位上
        while 1 <= nums[i] <= n and nums[nums[i] - 1] != nums[i]:
            # 那么就交换 nums[i] 和 nums[j]，其中 j 是 i 的学号
            j = nums[i] - 1
            nums[i], nums[j] = nums[j], nums[i]
    # 找第一个学号与座位编号不匹配的学生
    for i in range(n):
        if nums[i] != i + 1:
            return i + 1
    return n + 1
```

## 矩阵

矩阵类题目重点考察二维坐标的操控能力，常见技巧包括：利用第一行/列作为标记、方向数组、削皮算法、矩阵变换等。

### 矩阵置零（中等）

空间优化：用矩阵的第一行和第一列来记录哪些行/列需要置零，额外只需两个布尔变量。

```python
def setZeroes(self, matrix: List[List[int]]) -> None:
    m, n = len(matrix), len(matrix[0])
    first_row_has_zero = 0 in matrix[0]  # 记录第一行是否包含 0
    first_col_has_zero = any(row[0] == 0 for row in matrix)  # 记录第一列是否包含 0

    # 用第一列 matrix[i][0] 保存 row_has_zero[i]
    # 用第一行 matrix[0][j] 保存 col_has_zero[j]
    for i in range(1, m):  # 无需遍历第一行
        for j in range(1, n):  # 无需遍历第一列
            if matrix[i][j] == 0:
                matrix[i][0] = 0  # 相当于 row_has_zero[i] = True
                matrix[0][j] = 0  # 相当于 col_has_zero[j] = True

    for i in range(1, m):  # 跳过第一行
        for j in range(1, n):  # 跳过第一列
            if matrix[i][0] == 0 or matrix[0][j] == 0:  # i 行或 j 列有 0
                matrix[i][j] = 0

    # 如果第一列一开始就包含 0，那么把第一列全变成 0
    if first_col_has_zero:
        for row in matrix:
            row[0] = 0

    # 如果第一行一开始就包含 0，那么把第一行全变成 0
    if first_row_has_zero:
        for j in range(n):
            matrix[0][j] = 0
```

### 螺旋矩阵（中等）

两种实现：方向数组法（记录方向，减少步数）和削皮算法（收集第一行 → 去除第一行 → 逆时针旋转矩阵）。

```python
def spiralOrder(self, matrix: List[List[int]]) -> List[int]:
    # 记录方向
    DIRS = (0, 1), (1, 0), (0, -1), (-1, 0)  # 右下左上
    m, n = len(matrix), len(matrix[0])
    size = m * n
    ans = []
    i, j, di = 0, -1, 0
    while len(ans) < size:
        dx, dy = DIRS[di]
        for _ in range(n):
            i += dx
            j += dy
            ans.append(matrix[i][j])
        di = (di + 1) % 4
        n, m = m - 1, n # 减少后面的循环次数（步数）
    return ans
    # 削皮算法
    # 循环：
    # （1）收集第一行
    # （2）去除第一行
    # （3）逆时针旋转矩阵
    ans = []
    while matrix:
        m = len(matrix)
        n = len(matrix[0])
        for i in range(n):
            ans.append(matrix[0][i])
        matrix_new = [[0] * (m - 1) for _ in range(n)]
        for i in range(n):
            for j in range(m - 1):
                matrix_new[i][j] = matrix[j + 1][n - 1 - i]
        matrix[:] = matrix_new
    return ans
```

### 旋转图像（中等）

矩阵旋转的核心变换关系：

- 矩阵转置：`matrix[i][j] ↔ matrix[j][i]`
- 矩阵水平镜像：`matrix[i][j] ↔ matrix[i][n-1-j]`
- 矩阵垂直镜像：`matrix[i][j] ↔ matrix[m-1-i][j]`
- 顺时针旋转 90 度：`new_matrix[j][n-1-i] = matrix[i][j]`，相当于垂直镜像 + 转置
- 顺时针旋转 180 度：`new_matrix[n-1-i][n-1-j] = matrix[i][j]`，相当于水平镜像 + 垂直镜像
- 逆时针旋转 90 度：`new_matrix[n-1-j][i] = matrix[i][j]`，相当于水平镜像 + 转置

```python
# matrix[:] = [list(row) for row in zip(*matrix[::-1])]
```

### 搜索二维矩阵 II（中等）

利用二叉搜索树的思想，从矩阵右上角开始搜索。

### （补充）对角线遍历

对于每条对角线，行号 i 加列号 j 是一个定值。偶数条从左下到右上，奇数条从右上到左下。

```python
def findDiagonalOrder(self, mat: List[List[int]]) -> List[int]:
    m, n = len(mat), len(mat[0])
    ans = []
    # 对于每条对角线，行号 i 加列号 j 是一个定值，从左上到右下，一条一条地枚举对角线
    for k in range(m + n - 1):
        min_j = max(k - m + 1, 0)
        max_j = min(k, n - 1)
        if k % 2 == 0: # 偶数从左到右
            for j in range(min_j, max_j + 1):
                ans.append(mat[k - j][j])
        else: # 奇数从右到左
            for j in range(max_j, min_j - 1, -1):
                ans.append(mat[k - j][j])
    return ans
```
