---
title: "Hot 100 刷题笔记（十三）：技巧与数学"
date: 2025-01-13
draft: false
categories: [LeetCode]
tags: [位运算, 数学, 快速幂, 组合数, 算法, Python]
summary: "LeetCode Hot 100 技巧专题：异或、Boyer-Moore 投票、荷兰国旗、快速幂、组合数计算、拒绝采样。"
ShowToc: true
---

本篇整理 LeetCode Hot 100 中技巧与数学相关的题目，包括位运算、投票算法、荷兰国旗问题，以及快速幂、组合数、拒绝采样等数学工具。

> **常用技巧：**
>
> 位运算 `n & (n - 1)` 消除数字 n 二进制表示中的最后一个 1
>
> 异或运算 `a ^ a = 0` `a ^ 0 = 0`

## 只出现一次的数字（简单）

位运算，异或。

## 多数元素（简单）

```python
# 投票
def majorityElement(self, nums: List[int]) -> int:
    count = 0
    candidate = None
    for num in nums:
        if count == 0:
            candidate = num
        count += (1 if num == candidate else -1)

    return candidate
```

## 颜色分类（中等）

```python
def sortColors(self, nums: List[int]) -> None:
    # two_i：先把房子全刷成蓝色（2）。
    # one_i：遇到 0 或 1 的房间，再刷一层黄色（1）。
    # zero_i：遇到 0 的房间，再刷成白色（0）。
    # 最后你看到的房子，就是前面全白，中间全黄，后面全蓝
    zero, one, two = 0, 0, 0
    for num in nums:
        nums[two] = 2
        two += 1           
        if num <= 1:
            nums[one] = 1
            one += 1
        if num == 0:
            nums[zero] = 0
            zero += 1
```

## 下一个排列（中等）

```python
def nextPermutation(self, nums: List[int]) -> None:
    n = len(nums)
    # 第一步：从右向左找到第一个小于右侧相邻数字的数 nums[i]
    i = n - 2
    while i >= 0 and nums[i] >= nums[i + 1]:
        i -= 1
    if i >= 0:
        j = n - 1
        while nums[j] <= nums[i]:
            j -= 1
        nums[i], nums[j] = nums[j], nums[i]

    # 第三步：反转 nums[i+1:]（如果上面跳过第二步，此时 i = -1）
    left, right = i + 1, n - 1
    while left < right:
        nums[left], nums[right] = nums[right], nums[left]
        left += 1
        right -= 1
```

## 寻找重复数（中等）

等价于环形链表。

```python
def findDuplicate(self, nums: List[int]) -> int:
    slow = fast = 0
    while True:
        slow = nums[slow]
        fast = nums[nums[slow]]
        if fast == slow:
            break

    head = 0
    while slow != head:
        slow = nums[slow]
        head = nums[head]
    return slow
```

## （补充）最大公约数

辗转相除法、更相减损术、Stein 算法。

```python
# gcd(x, y)=gcd(y, x mod y) (x >= y)
def gcd(x, y):
    if y == 0:
        return x
    return gcd(y, x % y)
def gcd(x, y):
    while y:
        x, y = y, x % y
    return x

def gcd(x, y):
    if x == y:
        return x
    if x < y:
        return gcd(y-x, x)
    return gcd(x-y, y)

def gcd(x, y):
    if x < y:
        x, y = y, x
    if y == 0:
        return x
    if x % 2 == 0 and y % 2 == 0:
        return 2 * gcd(x // 2, y // 2)
    if x % 2 == 0:
        return gcd(x // 2, y)
    if y % 2 == 0:
        return gcd(x, y // 2)
    return gcd((x - y) // 2, y)
```

## （补充）素数（埃氏筛）

```python
isPrime = [True] * n
for i in range(2, int(n ** 0.5) + 1):
    if isPrime[i]:
        for j in range(i * i, n, i):
            isPrime[j] = False
```

## （补充）快速幂 Pow(x, n)

```python
def myPow(self, x: float, n: int) -> float:
    # 递归
    def quickMul(N):
        if N == 0:
            return 1.0
        y = quickMul(N // 2)
        return y * y if N % 2 == 0 else y * y * x

    # 迭代
    def quickMul(N):
        ans = 1.0
        y = x
        while N:
            if N % 2 == 1:
                ans *= y
            y *= y
            N //= 2
        return ans
    
    return quickMul(n) if n >= 0 else 1.0 / quickMul(-n)
```

## （补充）模幂 a^n mod m

```python
a = a % m
r = 1
while n:
    if n & 1:
        r = (r * a) % m
    a = (a * a) % m
    n >>= 1
print(r)
```

### 除法取模

a 是 b 的倍数，b 和 p 互质时使用快速幂计算：

$$
\frac{a}{b} \text{mod} p = (a \cdot b^{p-2}) \text{mod} p
$$

## （补充）组合数计算（预处理阶乘及其逆元）

```python
MOD = 1_000_000_007
MX = 100_001  # 根据题目数据范围修改

fac = [0] * MX  # fac[i] = i!
fac[0] = 1
for i in range(1, MX):
    fac[i] = fac[i - 1] * i % MOD

inv_f = [0] * MX  # inv_f[i] = i!^-1
inv_f[-1] = pow(fac[-1], -1, MOD)
for i in range(MX - 1, 0, -1):
    inv_f[i - 1] = inv_f[i] * i % MOD

# 从 n 个数中选 m 个数的方案数
def comb(n: int, m: int) -> int:
    return fac[n] * inv_f[m] * inv_f[n - m] % MOD if 0 <= m <= n else 0

class Solution:
    def solve(self, nums: List[int]) -> int:
        # 预处理的逻辑写在 class 外面，这样只会初始化一次
```

## （补充）用 rand7() 实现 rand10()（拒绝采样+构造等概率）

```python
# 通用方法——生成等概率产生0和1的方法：rand2(), 用这个 rand2() 方法生成n的每个二进制位
def rand2(self):
    ans = rand7()
    return self.rand2() if ans == 7 else ans % 2
def rand10(self):
    ans = self.rand2()
    for _ in range(3):
        ans = (ans << 1) ^ self.rand2()
    return ans if 1 <= ans <= 10 else self.rand10()

# (randX() - 1)*Y + randY() 可以等概率的生成[1, X * Y]范围的随机数
def rand10(self):
    while True:
        num = (rand7() - 1) * 7 + rand7()
        # 如果在40以内，那就直接返回
        if num <= 40: return 1 + num % 10
        # 说明刚才生成的在41-49之间，利用随机数再操作一遍
        num = (num - 40 - 1) * 7 + rand7()
        if num <= 60: return 1 + num % 10
        # 说明刚才生成的在61-63之间，利用随机数再操作一遍
        num = (num - 60 - 1) * 7 + rand7()
        if num <= 20: return 1 + num % 10
```
