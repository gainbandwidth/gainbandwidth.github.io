---
title: "Hot 100 刷题笔记（二）：双指针与接雨水"
date: 2025-01-02
draft: false
categories:
  - LeetCode
tags:
  - LeetCode
  - 双指针
  - 接雨水
  - 算法
  - Python
summary: "LeetCode Hot 100 双指针专题：移动零、盛最多水的容器、三数之和、接雨水（四种解法）。"
ShowToc: true
---

## 双指针

双指针是数组类题目中非常核心的技巧，通过两个指针的协同移动来减少不必要的遍历，将时间复杂度优化到 O(n)。本节从简单到困难，逐步深入，最后以经典的"接雨水"四种解法收尾。

### 移动零（简单）

快慢指针的经典应用。

### 盛最多水的容器（中等）

左右双指针向中间收缩，每次移动较矮的那一侧，因为容器盛水量取决于较短的边。

```python
def maxArea(self, height: List[int]) -> int:
    n = len(height)
    l, r = 0, n - 1
    ans = 0
    while l < r:
        ans = max(ans, min(height[l], height[r]) * (r - l))
        if height[l] < height[r]:
            l += 1
        else:
            r -= 1
    return ans
```

### 三数之和（中等）

先排序，再固定一个数，用双指针找另外两个数。注意剪枝以避免重复。

```python
def threeSum(self, nums: List[int]) -> List[List[int]]:
    n = len(nums)
    nums.sort()
    ans = []
    for i in range(n - 2):
        x = nums[i]
        if i > 0 and x == nums[i - 1]: # 跳过重复数字
            continue
        # if x + nums[i + 1] + nums[i + 2] > 0:
        #     break
        # if x + nums[-2] + nums[-1] < 0:
        #     continue
        j = i + 1
        k = n - 1
        while j < k:
            s = x + nums[j] + nums[k]
            if s > 0:
                k -= 1
            elif s < 0:
                j += 1
            else:
                ans.append([x, nums[j], nums[k]])
                j += 1
                while j < k and nums[j] == nums[j - 1]: # 跳过重复数字
                    j += 1
                k -= 1
                while k > j and nums[k] == nums[k + 1]: # 跳过重复数字
                    k -= 1
    return ans
```

### （补充）最接近的三数之和

在三数之和的基础上，记录与 target 最接近的和，加入更多剪枝优化。

```python
def threeSumClosest(self, nums: List[int], target: int) -> int:
    nums.sort()
    n = len(nums)
    ans = inf
    for i in range(n - 2):
        x = nums[i]
        if i and x == nums[i - 1]:
            continue
        s = x + nums[i + 1] + nums[i + 2]
        if s > target:
            if s - target < abs(ans - target):
                ans = s
            break
        s = x + nums[-2] + nums[-1]
        if s < target:
            if target - s < abs(ans - target):
                ans = s
            continue
        j, k = i + 1, n - 1
        while j < k:
            s = x + nums[j] + nums[k]
            if s == target:
                return target
            if abs(s - target) < abs(ans - target):
                ans = s
            if s > target:
                k -= 1
            else:
                j += 1
    return ans
```

### 接雨水（困难）

接雨水是经典难题，这里给出四种解法：前后缀分解、双指针、单调栈、动态规划。

#### 解法一：前后缀分解（两个数组）

用两个数组分别存储每个位置左边和右边的最大高度。

```python
# 最简单，两个数组存储两边传递来的木板最大值（前后缀分解）
def trap(self, height):
    ans = 0
    n = len(height)
    pre_max = [height[0]] + [0] * (n - 1)
    for i in range(1, n):
        pre_max[i] = max(pre_max[i - 1], height[i])
    sur_max = [0] * (n - 1) + [height[-1]]
    for i in range(n - 2, -1, -1):
        sur_max[i] = max(sur_max[i + 1], height[i])

    for i in range(n):
        ans += min(pre_max[i], sur_max[i]) - height[i]
    return ans
```

#### 解法二：双指针

左右指针向中间收缩，维护两边的最大值，空间复杂度 O(1)。

```python
# 双指针
def trap(self, height):
    ans = 0
    l, r = 0, len(height) - 1
    leftMax = rightMax = 0
    while l < r:
        leftMax = max(leftMax, height[l])
        rightMax = max(rightMax, height[r])
        if height[l] < height[r]:
            ans += leftMax - height[l]
            l += 1
        else:
            ans += rightMax - height[r]
            r -= 1
    return ans
```

#### 解法三：单调栈

维护一个单调递减的栈，遇到比栈顶高的元素时，逐个弹出并计算积水。

```python
# 单调栈（单调递减的栈，遇到高的逐个弹出）
def trap(self, height: List[int]) -> int:
    ans = 0
    stack = list()
    n = len(height)

    for i, h in enumerate(height):
        while stack and h > height[stack[-1]]:
            top = stack.pop()
            if not stack:
                break
            left = stack[-1]
            currWidth = i - left - 1
            currHeight = min(height[left], height[i]) - height[top]
            ans += currWidth * currHeight
        stack.append(i)

    return ans
```

#### 解法四：动态规划

本质上与前后缀分解思路一致，写法更加简洁。

```python
# 动态规划
def trap(self, height: List[int]) -> int:
    if not height:
        return 0

    n = len(height)
    leftMax = [height[0]] + [0] * (n - 1)
    for i in range(1, n):
        leftMax[i] = max(leftMax[i - 1], height[i])

    rightMax = [0] * (n - 1) + [height[n - 1]]
    for i in range(n - 2, -1, -1):
        rightMax[i] = max(rightMax[i + 1], height[i])

    ans = sum(min(leftMax[i], rightMax[i]) - height[i] for i in range(n))
    return ans
```
