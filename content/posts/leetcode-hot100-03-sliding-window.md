---
title: "Hot 100 刷题笔记（三）：滑动窗口与子串"
date: 2025-01-03
draft: false
categories:
  - LeetCode
tags:
  - 滑动窗口
  - 单调队列
  - 前缀和
  - 算法
  - Python
summary: "LeetCode Hot 100 滑动窗口与子串专题：无重复字符最长子串、滑动窗口最大值、最小覆盖子串等。"
ShowToc: true
---

## 滑动窗口

滑动窗口是处理连续子数组/子串问题的重要技巧，本质上是双指针的变体。通过维护一个窗口区间 [left, right]，在满足条件时扩展右边界，不满足时收缩左边界，实现 O(n) 的时间复杂度。

### 无重复字符的最长子串（中等）

用哈希表记录窗口内字符的出现次数，当出现重复字符时收缩左边界。

```python
def lengthOfLongestSubstring(self, s: str) -> int:
    ans = 0
    left = 0
    cnt = defaultdict(int)
    for right, c in enumerate(s):
        cnt[c] += 1
        while cnt[c] > 1:
            cnt[s[left]] -= 1 # 注意是 s[left] 需要的是字符串对应位置的字符
            left += 1
        ans = max(ans, right - left + 1)
    return ans
```

### 找到字符串中所有字母异位词（中等）

给出两种滑窗实现：定长滑窗和不定长滑窗（不定长更优）。

```python
def findAnagrams(self, s: str, p: str) -> List[int]:
    # 定长滑窗
	cnt_p = Counter(p)
    cnt_s = Counter(s)
    ans = []
    for right, c in enumerate(s):
        cnt_s[c] += 1
        left = right - len(p) + 1
        if left < 0:
            continue
        if cnt_s == cnt_p:
            ans.append(left)
        cnt_s[s[left]] -= 1
    return ans
    # 不定长滑窗（更优）
    cnt = Counter(p)
    ans = []
    left = 0
    for right, c in enumerate(s):
        cnt[c] -= 1
        while cnt[c] < 0:
            cnt[s[left]] += 1
            left += 1
        if right - left + 1 == len(p):
            ans.append(left)
    return ans
```

## 子串

子串类问题常结合前缀和、哈希表等技巧，下面几道题目涉及前缀和优化和单调队列等进阶知识点。

### 和为 K 的子数组（中等）

前缀和优化：用哈希表记录前缀和出现的次数，注意初始情况 `cnt[0] = 1`。关键在于"先累加答案，再更新计数"的顺序。

```python
def subarraySum(self, nums: List[int], k: int) -> int:
    cnt = defaultdict(int)
    cnt[0] = 1 # 注意初始情况
    ans = s = 0
    for x in nums:
        s += x
        ans += cnt[s - k]
        cnt[s] += 1
    return ans
    '''
    cnt = defaultdict(int)
    ans = s = 0
    for x in nums:
        cnt[s] += 1
        s += x
        ans += cnt[s - k]
    return ans
    '''
```

### 滑动窗口最大值（困难）

单调队列的经典应用：及时去掉无用数据，保证双端队列有序。队列中存储的是下标而非值。

```python
def maxSlidingWindow(self, nums, k):
    ans = [0] * (len(nums) - k + 1)
    q = deque()
    for i, x in enumerate(nums):
        while q and nums[q[-1]] <= x:
            q.pop()
        q.append(i)

        left = i - k + 1
        if q[0] < left:
            q.popleft()
        if left >= 0:
            ans[left] = nums[q[0]]
    return ans
```

### 最小覆盖子串（困难）

用哈希表计数 + 变量 `less` 记录还未满足条件的字符种类数，当 `less == 0` 时尝试收缩左边界。

```python
def minWindow(self, s: str, t: str) -> str:
    cnt = Counter(t)
    less = len(cnt)
    ans_left, ans_right = -1, len(s)
    left = 0
    for right, c in enumerate(s):
        cnt[c] -= 1 # 右端点字母移入子串
        if cnt[c] == 0: # 原来窗口内 c 的出现次数比 t 的少，现在一样多
            less -= 1
        while less == 0: # 涵盖：所有字母的出现次数都是 >=
            if right - left < ans_right - ans_left:
                ans_left, ans_right = left, right
            x = s[left]
            if cnt[x] == 0:
                less += 1
            cnt[x] += 1 # 左端点字母移出子串
            left += 1
    return "" if ans_left < 0 else s[ans_left: ans_right + 1]
```
