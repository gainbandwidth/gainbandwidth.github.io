---
title: "Hot 100 刷题笔记（十二）：动态规划"
date: 2025-01-12
draft: false
categories: [LeetCode]
tags: [动态规划, 背包, LIS, LCS, 编辑距离, 算法, Python]
summary: "LeetCode Hot 100 动态规划专题：打家劫舍、零钱兑换、LIS、LCS、编辑距离、最长回文子串、最长有效括号。"
ShowToc: true
---

本篇整理 LeetCode Hot 100 中动态规划相关的经典题目，涵盖背包 DP、线性 DP、区间 DP、多维 DP 等多种类型。

> **DP 分类总结：**
>
> 背包DP：0-1背包、子集背包、完全背包、目标和
>
> 线性DP：最长递增子序列（LIS）、最长公共子序列（LCS）、编辑距离
>
> 区间DP：最长回文子序列
>
> 状态机DP：买卖股票
>
> 树形DP：二叉树的直径、树上最大独立集、树上最小支配集
>
> 选与不选 / 枚举选哪个
>
> - 相邻无关子序列问题（比如 0-1 背包），适合「选或不选」。每个元素互相独立，只需依次考虑每个元素选或不选。
> - 相邻相关子序列问题（比如最长递增子序列），适合「枚举选哪个」。我们需要知道子序列中的相邻两个数的关系。

## 爬楼梯（简单）

## 杨辉三角（简单）

```python
def generate(self, numRows: int) -> List[List[int]]:
    ans = [[1] * (i + 1) for i in range(numRows)]
    for i in range(2, numRows):
        for j in range(1, i):
            ans[i][j] = ans[i - 1][j - 1] + ans[i - 1][j]
    return ans
```

## 打家劫舍

```python
def rob(self, nums):
    cur, pre = 0, 0
    for num in nums:
        cur, pre = max(pre + num, cur), cur
    return cur
    # f = [0] * (len(nums) + 2)
    # for i, x in enumerate(nums):
    #     f[i + 2] = max(f[i + 1], f[i] + x)
    # return f[-1]
```

### 二、数组首尾相连（拆分成两次）

```python
def rob(self, nums: List[int]) -> int:
    return max(nums[0] + self.rob1(nums[2:-1]), self.rob1(nums[1:])) # 分为选 nums[0] 和不选
```

### 三、二叉树上（DFS，设计好返回值）

```python
def rob(self, root: Optional[TreeNode]) -> int:
    # 两个返回值分别表示当前节点选/不选时的最大值
    # 转移方程
    # 选 = 左不选 + 右不选 + 当前节点
    # 不选 = Max（左选，左不选）+ Max（右选，右不选）
    def dfs(node):
        if node is None:
            return 0, 0
        l1, l2 = dfs(node.left)
        r1, r2 = dfs(node.right)
        ans1 = node.val + l2 + r2
        ans2 = max(l1, l2) + max(r1, r2)
        return ans1, ans2

    ans1, ans2 = dfs(root)
    return max(ans1, ans2)
```

### 四、k 个房间最大金额的最小值（最小化最大值 / 最大化最小值——二分）

```python
def minCapability(self, nums: List[int], k: int) -> int:
    # 二分 + DP
    def check(mx):
        # f[i] 表示从 nums[0] 到 nums[i] 中偷金额不超过 mx 的房屋，最多能偷多少间房屋
        # 如果 f[n−1]≥k 则表示答案至多为 mx，否则表示答案必须超过 mx
        pre = cur = 0
        for x in nums:
            if x > mx:
                pre = cur
            else:
                pre, cur = cur, max(cur, pre + 1)
        return cur >= k
    l, r = 0, max(nums)
    while l <= r:
        m = (l + r) // 2
        if check(m):
            r = m - 1
        else:
            l = m + 1
    return l
```

## 完全平方数（中等）

## 零钱兑换（中等）

```python
def coinChange(self, coins, amount):
    # dp 数组初始值为 amount + 1，保证不会达到也不会因为后序处理超出 int 边界
    f = [amount + 1] * (amount + 1)
    f[0] = 0
    for i in range(amount + 1):
        for coin in coins:
            if i >= coin:
                f[i] = min(f[i], f[i - coin] + 1)
    return -1 if f[-1] == amount + 1 else f[-1]
```

### （补充）零钱兑换 II（凑成 amount 的组合数）

```python
def change(self, amount: int, coins: List[int]) -> int:
    n = len(coins)
    f = [[0] * (amount + 1) for _ in range(n + 1)]
    f[0][0] = 1
    for i, c in enumerate(coins):
        for x in range(amount + 1):
            if x < c:
                f[i + 1][x] = f[i][x]
            else:
                f[i + 1][x] = f[i][x] + f[i + 1][x - c]
    return f[n][amount]
```

## 单词拆分（中等）

```python
def wordBreak(self, s: str, wordDict: List[str]) -> bool:
    n = len(s)
    f = [True] + [False] * n
    for i in range(n):
        for j in range(i + 1, n + 1):
            if f[i] and s[i:j] in wordDict:
                f[j] = True
    return f[-1]
```

## 最长递增子序列（中等）

动态规划 / 贪心+二分查找。

```python
# 动态规划 O(N^2)
def lengthOfLIS(self, nums: List[int]) -> int:
    dp = [1] * len(nums)
    for i in range(len(nums)):
        for j in range(i):
            if nums[i] > nums[j]:
                dp[i] = max(dp[i], dp[j] + 1)
    return max(dp)
# 二分查找 O(NlogN)
def lengthOfLIS(self, nums):
    g = []
    for x in nums:
        j = bisect_left(g, x)
        if j == len(g): # >=x 的 g[j] 不存在
            g.append(x)
        else:
            g[j] = x
    return len(g)
```

## 乘积最大子数组（中等）

最大子数组和的乘法版本。

```python
def maxProduct(self, nums: List[int]) -> int:
    ans = -inf  # 注意答案可能是负数
    f_max = f_min = 1
    for x in nums:
        f_max, f_min = max(f_max * x, f_min * x, x), \
                       min(f_max * x, f_min * x, x)
        ans = max(ans, f_max)
    return ans
```

## 分割等和子集（中等）

恰好装满 0-1 背包。

```python
# 0-1背包
# dp[i][w] 对于前 i 个物品，当前背包容量为 w 可装下的最大价值
def knapsack(W: int, wt: List[int], val: List[int]):
    N = len(wt)
    dp = [[0] * (W + 1) for _ in range(N + 1)]
    for i in range(1, N + 1):
        for w in range(1, W + 1):
            if w - wt[i - 1] < 0:
                dp[i][w] = dp[i - 1][w]
            else:
                dp[i][w] = max(dp[i - 1][w - wt[i - 1]] + val[i - 1], dp[i - 1][w])
    return dp[N][W]
```

```python
def canPartition(self, nums: List[int]) -> bool:
    Sum = sum(nums)
    if Sum % 2 != 0:
        return False
    # 转为 0-1 背包问题
    # 第i个物品的容量和价值都是nums[i]，而背包总容量是sum/2
    W = Sum // 2
    N = len(nums)
    # f[i+1][j] 定义为 nums[0,...,i] 中是否能选出一个和恰好为 j 的子序列
    # 状态转移方程 f[i+1][j] = f[i][j−nums[i]] ∨ f[i][j]
    f = [[False] * (W + 1) for _ in range(N + 1)]
    f[0][0] = True
    for i, x in enumerate(nums):
        for j in range(W + 1):
            f[i + 1][j] = j >= x and f[i][j - x] or f[i][j]
    return f[N][W]
```

## 最长有效括号（困难）

```python
# 贪心
# 从左到右遍历字符串，对于遇到的每个 '('，我们增加 left 计数器，对于遇到的每个 ')' ，我们增加 right 计数器。每当 left 计数器与 right 计数器相等时，我们计算当前有效字符串的长度，并且记录目前为止找到的最长子字符串。当 right 计数器比 left 计数器大时，我们将 left 和 right 计数器同时变回 0。
def longestValidParentheses(self, s):
    left, right, ans = 0, 0, 0
    for i in range(len(s)):
        if s[i] == '(':
            left += 1
        else:
            right += 1
        if left == right:
            ans = max(ans, 2 * right)
        elif right > left:
            left = right = 0

    left = right = 0
    for i in range(len(s) - 1, -1, -1):
        if s[i] == '(':
            left += 1
        else:
            right += 1
        if left == right:
            ans = max(ans, 2 * left)
        elif left > right:
            left = right = 0

    return ans
# 栈
# 始终保持栈底元素为当前已经遍历过的元素中「最后一个没有被匹配的右括号的下标」
# 对于遇到的每个 '(' ，将它的下标放入栈中
# 对于遇到的每个 ')' ，先弹出栈顶元素表示匹配了当前右括号：
# 如果栈为空，说明当前的右括号为没有被匹配的右括号，将其下标放入栈中来更新「最后一个没有被匹配的右括号的下标」
# 如果栈不为空，当前右括号的下标减去栈顶元素即为「以该右括号为结尾的最长有效括号的长度」
def longestValidParentheses(self, s):
    ans = 0
    stack = []
    stack.append(-1)
    for i in range(len(s)):
        if s[i] == '(':
            stack.append(i)
        else:
            stack.pop()
            if not stack:
                stack.append(i)
            else:
                ans = max(ans, i - stack[-1])
    return ans
# 动态规划
def longestValidParentheses(self, s):
    dp = [0] * len(s)
    ans = 0
    for i in range(1, len(s)):
        if s[i] == ')':
            if s[i - 1] == '(':
                dp[i] = dp[i - 2] + 2 if i >= 2 else 2
            elif i - dp[i - 1] > 0 and s[i - dp[i - 1] - 1] == '(':
                dp[i] = dp[i - 1] + (dp[i - dp[i - 1] - 2] if i - dp[i - 1] >= 2 else 0) + 2
            ans = max(ans, dp[i])

    return ans
```

## 多维动态规划

### 不同路径（中等）

二维数组 / 组合数。

```python
def uniquePaths(self, m: int, n: int) -> int:
    f = [[1] * n for _ in range(m)]
    for i in range(1, m):
        for j in range(1, n):
            f[i][j] = f[i - 1][j] + f[i][j - 1]
    return f[-1][-1]
    
    def comb(n, k):
        k = min(k, n - k)
        res = 1
        for i in range(1, k + 1):
            res = res * (n + 1 - i) // i
        return res
    ans = comb(m + n - 2, m - 1)
```

### 最小路径和（中等）

二维数组。

```python
def minPathSum(self, grid: List[List[int]]) -> int:
    g = grid[:]
    m, n = len(grid), len(grid[0])
    for i in range(1, m):
        g[i][0] += g[i - 1][0]
    for j in range(1, n):
        g[0][j] += g[0][j - 1]
    for i in range(1, m):
        for j in range(1, n):
            g[i][j] += min(g[i - 1][j], g[i][j - 1])
    return g[-1][-1]
```

### 最长回文子串（中等）

动态规划 / 中心扩散法。

```python
def longestPalindrome(self, s):
    n = len(s)
    max_len = 1
    ans = s[0]
    # dp[i][j]表示字符i-j(包括)，即s[i:j+1]
    dp = [[False] * n for _ in range(n)]
    for i in range(n - 1, -1, -1):
        dp[i][i] = True
        for j in range(i + 1, n):
            if (s[i] == s[j]) and (j - i <= 2 or dp[i + 1][j - 1]):
                dp[i][j] = True
                if j - i + 1 > max_len:
                    max_len = j - i + 1
                    ans = s[i:j+1]
    return ans

def longestPalindrome(self, s: str) -> str:
    n = len(s)
    ans_left = ans_right = 0

    for i in range(2 * n - 1):
        l, r = i // 2, (i + 1) // 2
        while l >= 0 and r < n and s[l] == s[r]:
            l -= 1
            r += 1
        # 循环结束后，s[l+1] 到 s[r-1] 是回文串
        if r - l - 1 > ans_right - ans_left:
            ans_left, ans_right = l + 1, r  # 左闭右开区间

    return s[ans_left: ans_right]
```

### （补充）最长回文子序列

可以转为原数组和反转后的数组的最长公共子序列。

```python
def longestPalindromeSubseq(self, s: str) -> int:
    n = len(s)
    f = [[0] * n for _ in range(n)]
    for i in range(n-1, -1, -1):
        f[i][i] = 1
        for j in range(i+1, n):
            if s[i] == s[j]:
                f[i][j] = f[i+1][j-1] + 2
            else:
                f[i][j] = max(f[i+1][j], f[i][j-1])
    return f[0][n-1]
```

### 最长公共子序列（中等）

分类讨论即可。

```python
def longestCommonSubsequence(self, s, t):
    n = len(s)
    m = len(t)
    f = [[0] * (m + 1) for _ in range(n + 1)]
    for i, x in enumerate(s):
        for j, y in enumerate(t):
            if x == y:
                f[i + 1][j + 1] = f[i][j] + 1
            else:
                f[i + 1][j + 1] = max(f[i][j + 1], f[i + 1][j])
    return f[n][m]
```

### （补充）最长重复子数组

注意和子序列的区分，子数组是连续的。

```python
def findLength(self, nums1: List[int], nums2: List[int]) -> int:
    # f[i+1][j+1] 表示以 nums1[i] 结尾的子数组和以 nums2[j] 结尾的子数组的最长公共子数组的长度
    n, m = len(nums1), len(nums2)
    f = [[0] * (m + 1) for _ in range(n + 1)]
    for i, x in enumerate(nums1):
        for j, y in enumerate(nums2):
            if x == y:
                f[i + 1][j + 1] = f[i][j] + 1
    return max(map(max, f)) # 所有 f[i][j] 的最大值
```

### 编辑距离（中等）

```python
def minDistance(self, word1: str, word2: str) -> int:
    n = len(word1)
    m = len(word2)
    # dp[i][j] 代表 word1 中前 i 个字符，变换到 word2 中前 j 个字符，最短需要操作的次数；
    # dp[i-1][j-1] 表示替换操作，dp[i-1][j] 表示删除操作，dp[i][j-1] 表示插入操作
    f = [[0] * (m + 1) for _ in range(n + 1)]
    f[0] = list(range(m + 1))

    for i, x in enumerate(word1):
        f[i + 1][0] = i + 1
        for j, y in enumerate(word2):
            if x == y:
                f[i + 1][j + 1] = f[i][j]
            else:
                f[i + 1][j + 1] = min(f[i][j], f[i+1][j], f[i][j+1]) + 1

    return f[n][m]
```
