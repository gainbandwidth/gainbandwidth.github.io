---
title: "Hot 100 刷题笔记（八）：回溯法"
date: 2025-01-08
draft: false
categories:
  - LeetCode
tags:
  - 回溯
  - 排列
  - 组合
  - 子集
  - N皇后
  - 算法
  - Python
summary: "LeetCode Hot 100 回溯专题：全排列、N 皇后、子集、组合总和、括号生成、单词搜索。"
ShowToc: true
---

## 回溯（子集、组合、排列）

本文整理 LeetCode Hot 100 中回溯法相关经典题目，涵盖排列型、子集型、组合型三大类回溯问题。

> 回溯三问：当前操作、子问题、下一个子问题
>
> - 排列型：通常答案长度固定，可以函数参数设置为当前位置，使用集合不需要恢复现场 / visited数组要恢复——全排列、N 皇后（可选集合作为函数参数 / 另外定义 visited 数组）
> - 子集型：path 记录选出的子集，需要 `path.append` 和  `path.pop`，有时虽然个数固定，但用子集的形式定义更方便（剪枝综合考虑目标总和 & 剩余个数）——子集、分割回文串（选或不选 / 枚举选哪个）、组合总和
> - 组合型：path 记录选出的组合（组合总数固定），需要对可行性剪枝——组合、括号生成（选或不选 / 枚举选哪个）

### 排列型

**全排列 中等**（回溯函数参数为当前位置，记录`on_path` 或者 使用集合 `dfs(i, s)`）

```python
def permute(self, nums):
    n = len(nums)
    path = [0] * n
    on_path = [False] * n
    ans = []

    # 枚举 path[i] 填 nums 的哪个数
    def dfs(i):
        if i == n:
            ans.append(path.copy())
            return
        for j, on in enumerate(on_path):
            if not on:
                path[i] = nums[j]
                on_path[j] = True
                dfs(i + 1)
                on_path[j] = False

    dfs(0)
    return ans
```

（补充）全排列 II（在全排列基础上，对于重复元素要判断之前的时候on_path，如果是直接跳过）

```python
def permuteUnique(self, nums: List[int]) -> List[List[int]]:
    nums.sort()
    n = len(nums)
    path = [0] * n
    on_path = [False] * n
    ans = []
    def dfs(i):
        if i == n:
            ans.append(path.copy())
            return
        for j, on in enumerate(on_path):
            if on or j > 0 and nums[j] == nums[j - 1] and not on_path[j - 1]:
                continue
            path[i] = nums[j]
            on_path[j] = True
            dfs(i + 1)
            on_path[j] = False
    dfs(0)
    return ans
```

**N 皇后 困难**（集合（不需要恢复现场））

```python
def solveNQueens(self, n: int) -> List[List[str]]:
    ans = []
    col = [0] * n

    def valid(r, c):
        for R in range(r):
            C = col[R]
            if r + c == R + C or r - c == R - C:
                return False
        return True

    def dfs(r, s):
        if r == n:
            ans.append(['.'*c + 'Q' + '.'*(n-1-c) for c in col])
            return
        for c in s:
            if valid(r, c):
                col[r] = c
                dfs(r + 1, s - {c})

    dfs(0, set(range(n)))
    return ans
```

电话号码的字母组合 中等

```python
def letterCombinations(self, digits: str) -> List[str]:        
    n = len(digits)
    strs = ["", "", "abc", "def", "ghi", "jkl", "mno", "pqrs", "tuv", "wxyz"]
    ans = []
    path = [''] * n
    def dfs(i):
        if i == n:
            ans.append(''.join(path))
            return
        for c in strs[int(digits[i])]:
            path[i] = c
            dfs(i + 1)
    dfs(0)
    return ans
```

### 子集型

**子集 中等**（`path.append` 和  `path.pop`）

```python
def subsets(self, nums: List[int]) -> List[List[int]]:
    n = len(nums)
    ans = []
    path = []
    # 选或不选
    def dfs(i):
        if i == n:
            ans.append(path.copy())
            return
        dfs(i + 1)
        path.append(nums[i])
        dfs(i + 1)
        path.pop()
    # 枚举选哪个
    def dfs(i):
        ans.append(path.copy())
        if i == n:
            return
        for j in range(i, n):
            path.append(nums[j])
            dfs(j + 1)
            path.pop()
    dfs(0)
    return ans
```

**分割回文串 中等**

```python
def partition(self, s: str) -> List[List[str]]:
    n = len(s)
    ans = []
    path = []
    # 枚举选哪个的视角，考虑 s[i:] 怎么分割
    def dfs(i):
        if i == n:
            ans.append(path.copy())
            return
        for j in range(i, n): # 枚举子串的结束位置
            t = s[i: j + 1] # 分割出子串 t
            if t == t[::-1]: # 判断 t 是不是回文串
                path.append(t)
                # 子问题：考虑剩余的 s[j+1:] 怎么分割
                dfs(j + 1)
                path.pop()

    dfs(0)
    return ans
```

**（补充）复原 IP 地址**（在分割回文串基础上加上分割段数的限制）

```python
# 枚举选哪个（枚举子串右端点）
def restoreIpAddresses(self, s: str) -> List[str]:
    n = len(s)
    ans = []
    path = [''] * 4
    # 分割 s[i] 到 s[n-1]，现在在第 j 段（j 从 0 开始）
    def dfs(i, j):
        if not 4 - j <= n - i <= (4 - j) * 3:
            return
        if i == n: # 此时由于上面的剪枝，必然有 j = 4
            ans.append('.'.join(path))
            return
        # 子串左端点为 i, 枚举子串右端点 right
        ip_val = 0
        for right in range(i, n):
            ip_val = ip_val * 10 + int(s[right])
            if ip_val > 255:
                break
            path[j] = s[i: right + 1]
            dfs(right + 1, j + 1)
            if ip_val == 0: # 前导零
                break
    dfs(0, 0)
    return ans
```

组合总和 中等

```python
def combinationSum(self, candidates: List[int], target: int) -> List[List[int]]:
    # 选或不选（子集型回溯）
    # 减枝优化：把 candidates 从小到大排序，如果递归中发现 target<candidates[i]，可以直接返回
    candidates.sort()
    n = len(candidates)
    ans = []
    path = []

    def dfs(i, target):
        if target == 0:
            ans.append(path.copy())
            return
        if i >= n or target < candidates[i]: # 注意这边要加上边界条件 i >= n
            return           
        # 选
        path.append(candidates[i])
        dfs(i, target - candidates[i])
        path.pop()
        # 不选
        dfs(i + 1, target)

    dfs(0, target)
    return ans
```

（补充）组合总和 II（对于不选的情况，需要去重）

```python
def combinationSum2(self, candidates: List[int], target: int) -> List[List[int]]:
    candidates.sort()
    n = len(candidates)
    ans = []
    path = []
    def dfs(i, target):
        if target == 0:
            ans.append(path.copy())
            return
        if i >= n or candidates[i] > target:
            return
        # 选 x
        x = candidates[i]
        path.append(x)
        dfs(i + 1, target - x)
        path.pop()
        # 不选 x，那么后面所有等于 x 的数都不选
        i += 1
        while i < n and candidates[i] == x:
            i += 1
        dfs(i, target)
    dfs(0, target)
    return ans
```

（补充）组合总和 III（设计剪枝，包含目标总和 & 剩余选择个数）

```python
def combinationSum3(self, k: int, n: int) -> List[List[int]]:
    ans = []
    path = []
    def dfs(i, target):
        d = k - len(path)
        if target < 0 or (2 * i - d + 1) * d // 2 < target:
            return
        if d == 0:
            ans.append(path.copy())
            return
        if i > d:
            dfs(i - 1, target)
        path.append(i)
        dfs(i - 1, target - i)
        path.pop()
    dfs(9, n)
    return ans
```

（补充）组合总和 Ⅳ（动态规划，本质是爬楼梯）

```python
def combinationSum4(self, nums: List[int], target: int) -> int:
    # @cache
    # def dfs(i):
    #     if i == 0:
    #         return 1
    #     return sum(dfs(i - x) for x in nums if x <= i)
    # return dfs(target)
    f = [1] + [0] * target
    for i in range(1, target + 1):
        f[i] = sum(f[i - x] for x in nums if x <= i)
    return f[target]
```

### 组合型

**（补充）组合 中等**（path记录选出的组合，需要对可行性剪枝）

```python
def combine(self, n: int, k: int) -> List[List[int]]:
    ans = []
    path = []
    # 枚举选哪个：在 1 到 i 中选一个数，加到 path 末尾
    def dfs(i):
        d = k - len(path)
        if d == 0:
            ans.append(path.copy())
            return
        for j in range(i, d - 1, -1):
            path.append(j)
            dfs(j - 1)
            path.pop()
    # 选或不选：讨论 i 是否加入 path
    def dfs(i):
        d = k - len(path)
        if d == 0:
            ans.append(path.copy())
            return
        # 不选 i
        if i > d:
            dfs(i - 1)
        # 选 i
        path.append(i)
        dfs(i - 1)
        path.pop()

    dfs(n)
    return ans
```

括号生成 中等（约束：对于括号字符串的任意前缀，右括号的个数不能超过左括号的个数）

```python
def generateParenthesis(self, n: int) -> List[str]:
    ans = []
    m = 2 * n
    path = [''] * m
    # 枚举选哪个，i 表示当前位置，open 表示目前左括号个数
    def dfs(i, open):
        if i == m:
            ans.append(''.join(path))
            return
        if open < n:
            path[i] = '('
            dfs(i + 1, open + 1)
        if i - open < open:
            path[i] = ')'
            dfs(i + 1, open)
    dfs(0, 0)
    return ans
```

**单词搜索 中等**（网格图 DFS、可行性剪枝、顺序剪枝）

```python
def exist(self, board: List[List[str]], word: str) -> bool:
    cnt = Counter(c for row in board for c in row)
    if not cnt >= Counter(word):  # 优化一
        return False
    if cnt[word[-1]] < cnt[word[0]]:  # 优化二
        word = word[::-1]

    m, n = len(board), len(board[0])
    def dfs(i, j, k):
        if board[i][j] != word[k]:
            return False
        if k == len(word) - 1:
            return True
        board[i][j] = '' # 标记访问过
        for x, y in (i, j - 1), (i, j + 1), (i - 1, j), (i + 1, j):
            if 0 <= x < m and 0 <= y < n and dfs(x, y, k + 1):
                return True
        board[i][j] = word[k]
        return False

    return any(dfs(i, j, 0) for i in range(m) for j in range(n))
```
