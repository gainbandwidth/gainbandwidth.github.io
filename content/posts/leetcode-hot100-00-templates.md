---
title: "Hot 100 刷题笔记（零）：重点清单与算法模板"
date: 2024-12-31
draft: false
categories:
  - LeetCode
tags:
  - LeetCode
  - 算法模板
  - 并查集
  - 线段树
  - 单调队列
  - ST表
  - Python
summary: "LeetCode Hot 100 刷题前的准备：重点题目清单、易错点提醒、ACM 模式输入输出，以及常用算法模板（二维前缀和、单调队列、懒删除堆、并查集、树状数组、线段树、ST 表）。"
ShowToc: true
---

## 重点

1. ==快速排序、归并排序、堆排序==。卡一下都不行，没事就写着玩看最快多久能写完

   选择、冒泡、插入、（希尔）

   快排、归并、堆排序、

   计数排序、桶排序、基数排序

2. 二叉树的前中后序、层序遍历、2字遍历。迭代法写，递归一共那几行谁不会啊

3. ==反转链表==。各种反，直接反，反一段，K个一组全反，K个一组不够K个不反。

4. 接雨水！！！单调栈

5. 股票买卖！机器人寻路！！动态规划！

6. ==回溯法==也很容易考，这是一种思想经常与树和其他结合

7. 字符串的题最多变，有双指针==滑动窗口==还有动态规划还有其他很烦

8. ==LRU==必须至少独立写出来10次freebug，LFU这个至少5次吧

> [!CAUTION]
>
> 容易出错的地方：
>
> - 链表记得向前进
> - While 循环注意边界，通常要加上 0 <= i < n 这样的约束
> - 分清位置`i` 还是字符串 / 数组的值 `s[i] / nums[i]`，在哈希表尤其要注意
> - 回溯三问：当前问题、子问题、下一个子问题
> - 回溯结束边界记得写、记得return（二叉树节点为空），回溯有的要恢复现场，回溯在主函数中要调用
> - nonlocal ans
> - 哈希表记录和更新答案的顺序，有时需要开始前加上`cnt[0] = 1`
> - 滑动窗口模板：记录窗口左边界，主循环`for i, x in enumerate(nums)`，循环中`while`更改左边界和答案
> - 无重复字符的最长子串、最小覆盖子串——滑动窗口；最长递增子序列、最大公共子序列、最长回文子串、最长回文子序列、最长有效括号——动态规划
> - 动态规划注意初始化，根据合理的状态定义数组长度（+1）

> [!NOTE]
>
> ACM 模式：
>
> ```python
> n = int(input())
> a, b = map(int, input().split())
> arr = list(map(int, input().split()))
> 
> # 读取多行（先读n，再读n行）
> n = int(input())
> lines = [input().strip() for _ in range(n)]
> 
> # 快速读取大量输入（可选，用于卡时间场景）
> data = sys.stdin.read().split()
> # 然后用指针逐个取 data[idx], idx += 1
> 
> # 逐行快速读取（推荐）
> import sys
> for line in sys.stdin:
>     line = line.strip()
>     if not line:  # 跳过空行
>         continue
> 	# 处理 line
> 
> readline = sys.stdin.readline
> T = int(readline())
> for _ in range(T):
>     n = int(readline())
>     a = list(map(int, readline().split()))
> ```

## 模板

### 二维前缀和
```python
class NumMatrix:
    def __init__(self, matrix: List[List[int]]):
        m, n = len(matrix), len(matrix[0])
        s = [[0] * (n + 1) for _ in range(m + 1)]
        for i, row in enumerate(matrix):
            for j, x in enumerate(row):
                s[i + 1][j + 1] = s[i + 1][j] + s[i][j + 1] - s[i][j] + x
        self.s = s

    # 返回左上角在 (r1, c1)，右下角在 (r2, c2) 的子矩阵元素和
    def sumRegion(self, r1: int, c1: int, r2: int, c2: int) -> int:
        s = self.s
        return s[r2 + 1][c2 + 1] - s[r2 + 1][c1] - s[r1][c2 + 1] + s[r1][c1]
```

### 单调队列
```python
# 计算 nums 的每个长为 k 的窗口的最大值
# 时间复杂度 O(n)，其中 n 是 nums 的长度
def maxSlidingWindow(nums: List[int], k: int) -> List[int]:
    ans = [0] * (len(nums) - k + 1)  # 窗口个数
    q = deque()  # 双端队列

    for i, x in enumerate(nums):
        # 1. 右边入
        while q and nums[q[-1]] <= x:
            q.pop()  # 维护 q 的单调性
        q.append(i)  # 注意保存的是下标，这样下面可以判断队首是否离开窗口

        # 2. 左边出
        left = i - k + 1  # 窗口左端点
        if q[0] < left:  # 队首离开窗口
            q.popleft()

        # 3. 在窗口左端点处记录答案
        if left >= 0:
            # 由于队首到队尾单调递减，所以窗口最大值就在队首
            ans[left] = nums[q[0]]

    return ans
```

### 懒删除堆
```python
class LazyHeap:
    def __init__(self):
        self.heap = []  # 最小堆（最大堆可以把数字取反或重载 __lt__）
        self.remove_cnt = defaultdict(int)  # 每个元素剩余需要删除的次数
        self.size = 0  # 堆的实际大小

    # 删除
    def remove(self, x: Any) -> None:
        self.remove_cnt[x] += 1  # 懒删除
        self.size -= 1

    # 正式执行删除操作
    def _apply_remove(self) -> None:
        while self.heap and self.remove_cnt[self.heap[0]] > 0:
            self.remove_cnt[self.heap[0]] -= 1
            heappop(self.heap)

    # 查看堆顶
    def top(self) -> Any:
        self._apply_remove()
        return self.heap[0]  # 真正的堆顶

    # 出堆
    def pop(self) -> Any:
        self._apply_remove()
        self.size -= 1
        return heappop(self.heap)

    # 入堆
    def push(self, x: Any) -> None:
        if self.remove_cnt[x] > 0:
            self.remove_cnt[x] -= 1  # 抵消之前的删除
        else:
            heappush(self.heap, x)
        self.size += 1
```

### 并查集
```python
class UnionFind:
    def __init__(self, n: int):
        # 一开始有 n 个集合 {0}, {1}, ..., {n-1}
        # 集合 i 的代表元是自己，大小为 1
        self._fa = list(range(n))  # 代表元
        self._size = [1] * n  # 集合大小
        self.cc = n  # 连通块个数

    # 返回 x 所在集合的代表元
    # 同时做路径压缩，也就是把 x 所在集合中的所有元素的 fa 都改成代表元
    def find(self, x: int) -> int:
        fa = self._fa
        # 如果 fa[x] == x，则表示 x 是代表元
        if fa[x] != x:
            fa[x] = self.find(fa[x])  # fa 改成代表元
        return fa[x]

    # 判断 x 和 y 是否在同一个集合
    def is_same(self, x: int, y: int) -> bool:
        # 如果 x 的代表元和 y 的代表元相同，那么 x 和 y 就在同一个集合
        # 这就是代表元的作用：用来快速判断两个元素是否在同一个集合
        return self.find(x) == self.find(y)

    # 把 from 所在集合合并到 to 所在集合中
    # 返回是否合并成功
    def merge(self, from_: int, to: int) -> bool:
        x, y = self.find(from_), self.find(to)
        if x == y:  # from 和 to 在同一个集合，不做合并
            return False
        self._fa[x] = y  # 合并集合。修改后就可以认为 from 和 to 在同一个集合了
        self._size[y] += self._size[x]  # 更新集合大小（注意集合大小保存在代表元上）
        # 无需更新 _size[x]，因为我们不用 _size[x] 而是用 _size[find(x)] 获取集合大小，但 find(x) == y，我们不会再访问 _size[x]
        self.cc -= 1  # 成功合并，连通块个数减一
        return True

    # 返回 x 所在集合的大小
    def get_size(self, x: int) -> int:
        return self._size[self.find(x)]  # 集合大小保存在代表元上
```

### 带权并查集
```python
class UnionFind:
    def __init__(self, n: int):
        # 一开始有 n 个集合 {0}, {1}, ..., {n-1}
        # 集合 i 的代表元是自己，自己到自己的距离是 0
        self.fa = list(range(n))  # 代表元
        self.dis = [0] * n  # dis[x] 表示 x 到（x 所在集合的）代表元的距离

    # 返回 x 所在集合的代表元
    # 同时做路径压缩
    def find(self, x: int) -> int:
        fa = self.fa
        if fa[x] != x:
            root = self.find(fa[x])
            self.dis[x] += self.dis[fa[x]]  # 递归更新 x 到其代表元的距离
            fa[x] = root
        return fa[x]

    # 判断 x 和 y 是否在同一个集合（同普通并查集）
    def same(self, x: int, y: int) -> bool:
        return self.find(x) == self.find(y)

    # 计算从 from 到 to 的相对距离
    # 调用时需保证 from 和 to 在同一个集合中，否则返回值无意义
    def get_relative_distance(self, from_: int, to: int) -> int:
        self.find(from_)
        self.find(to)
        # to-from = (x-from) - (x-to) = dis[from] - dis[to]
        return self.dis[from_] - self.dis[to]

    # 合并 from 和 to，新增信息 to - from = value
    # 其中 to 和 from 表示未知量，下文的 x 和 y 也表示未知量
    # 如果 from 和 to 不在同一个集合，返回 True，否则返回是否与已知信息矛盾
    def merge(self, from_: int, to: int, value: int) -> bool:
        x, y = self.find(from_), self.find(to)
        dis = self.dis
        if x == y:  # from 和 to 在同一个集合，不做合并
            # to-from = (x-from) - (x-to) = dis[from] - dis[to] = value
            return dis[from_] - dis[to] == value
        #    x --------- y
        #   /           /
        # from ------- to
        # 已知 x-from = dis[from] 和 y-to = dis[to]，现在合并 from 和 to，新增信息 to-from = value
        # 由于 y-from = (y-x) + (x-from) = (y-to) + (to-from)
        # 所以 y-x = (to-from) + (y-to) - (x-from) = value + dis[to] - dis[from]
        dis[x] = value + dis[to] - dis[from_]  # 计算 x 到其代表元 y 的距离
        self.fa[x] = y
        return True
```

### 树状数组
```python
class FenwickTree:
    def __init__(self, n: int):
        self.tree = [0] * (n + 1)  # 使用下标 1 到 n

    # a[i] 增加 val
    # 1 <= i <= n
    # 时间复杂度 O(log n)
    def update(self, i: int, val: int) -> None:
        t = self.tree
        while i < len(t):
            t[i] += val
            i += i & -i

    # 计算前缀和 a[1] + ... + a[i]
    # 1 <= i <= n
    # 时间复杂度 O(log n)
    def pre(self, i: int) -> int:
        t = self.tree
        res = 0
        while i > 0:
            res += t[i]
            i &= i - 1
        return res

    # 计算区间和 a[l] + ... + a[r]
    # 1 <= l <= r <= n
    # 时间复杂度 O(log n)
    def query(self, l: int, r: int) -> int:
        if r < l:
            return 0
        return self.pre(r) - self.pre(l - 1)
```

### 线段树（无区间更新）
```python
# 线段树有两个下标，一个是线段树节点的下标，另一个是线段树维护的区间的下标
# 节点的下标：从 1 开始，如果你想改成从 0 开始，需要把左右儿子下标分别改成 node*2+1 和 node*2+2
# 区间的下标：从 0 开始
class SegmentTree:
    def __init__(self, arr, default=0):
        # 线段树维护一个长为 n 的数组（下标从 0 到 n-1）
        # arr 可以是 list 或者 int
        # 如果 arr 是 int，视作数组大小，默认值为 default
        if isinstance(arr, int):
            arr = [default] * arr
        n = len(arr)
        self._n = n
        self._tree = [0] * (2 << (n - 1).bit_length())
        self._build(arr, 1, 0, n - 1)

    # 合并两个 val
    def _merge_val(self, a: int, b: int) -> int:
        return max(a, b)  # **根据题目修改**

    # 合并左右儿子的 val 到当前节点的 val
    def _maintain(self, node: int) -> None:
        self._tree[node] = self._merge_val(self._tree[node * 2], self._tree[node * 2 + 1])

    # 用 a 初始化线段树
    # 时间复杂度 O(n)
    def _build(self, a: List[int], node: int, l: int, r: int) -> None:
        if l == r:  # 叶子
            self._tree[node] = a[l]  # 初始化叶节点的值
            return
        m = (l + r) // 2
        self._build(a, node * 2, l, m)  # 初始化左子树
        self._build(a, node * 2 + 1, m + 1, r)  # 初始化右子树
        self._maintain(node)

    def _update(self, node: int, l: int, r: int, i: int, val: int) -> None:
        if l == r:  # 叶子（到达目标）
            # 如果想直接替换的话，可以写 self._tree[node] = val
            self._tree[node] = self._merge_val(self._tree[node], val)
            return
        m = (l + r) // 2
        if i <= m:  # i 在左子树
            self._update(node * 2, l, m, i, val)
        else:  # i 在右子树
            self._update(node * 2 + 1, m + 1, r, i, val)
        self._maintain(node)

    def _query(self, node: int, l: int, r: int, ql: int, qr: int) -> int:
        if ql <= l and r <= qr:  # 当前子树完全在 [ql, qr] 内
            return self._tree[node]
        m = (l + r) // 2
        if qr <= m:  # [ql, qr] 在左子树
            return self._query(node * 2, l, m, ql, qr)
        if ql > m:  # [ql, qr] 在右子树
            return self._query(node * 2 + 1, m + 1, r, ql, qr)
        l_res = self._query(node * 2, l, m, ql, qr)
        r_res = self._query(node * 2 + 1, m + 1, r, ql, qr)
        return self._merge_val(l_res, r_res)

    # 更新 a[i] 为 _merge_val(a[i], val)
    # 时间复杂度 O(log n)
    def update(self, i: int, val: int) -> None:
        self._update(1, 0, self._n - 1, i, val)

    # 返回用 _merge_val 合并所有 a[i] 的计算结果，其中 i 在闭区间 [ql, qr] 中
    # 时间复杂度 O(log n)
    def query(self, ql: int, qr: int) -> int:
        return self._query(1, 0, self._n - 1, ql, qr)

    # 获取 a[i] 的值
    # 时间复杂度 O(log n)
    def get(self, i: int) -> int:
        return self._query(1, 0, self._n - 1, i, i)
```

### Lazy 线段树（有区间更新）
```python
class Node:
    __slots__ = 'val', 'todo'

class LazySegmentTree:
    # 懒标记初始值
    _TODO_INIT = 0  # **根据题目修改**

    def __init__(self, arr, default=0):
        # 线段树维护一个长为 n 的数组（下标从 0 到 n-1）
        # arr 可以是 list 或者 int
        # 如果 arr 是 int，视作数组大小，默认值为 default
        if isinstance(arr, int):
            arr = [default] * arr
        n = len(arr)
        self._n = n
        self._tree = [Node() for _ in range(2 << (n - 1).bit_length())]
        self._build(arr, 1, 0, n - 1)

    # 合并两个 val
    def _merge_val(self, a: int, b: int) -> int:
        return a + b  # **根据题目修改**

    # 合并两个懒标记
    def _merge_todo(self, a: int, b: int) -> int:
        return a + b  # **根据题目修改**

    # 把懒标记作用到 node 子树（本例为区间加）
    def _apply(self, node: int, l: int, r: int, todo: int) -> None:
        cur = self._tree[node]
        # 计算 tree[node] 区间的整体变化
        cur.val += todo * (r - l + 1)  # **根据题目修改**
        cur.todo = self._merge_todo(todo, cur.todo)

    # 把当前节点的懒标记下传给左右儿子
    def _spread(self, node: int, l: int, r: int) -> None:
        todo = self._tree[node].todo
        if todo == self._TODO_INIT:  # 没有需要下传的信息
            return
        m = (l + r) // 2
        self._apply(node * 2, l, m, todo)
        self._apply(node * 2 + 1, m + 1, r, todo)
        self._tree[node].todo = self._TODO_INIT  # 下传完毕

    # 合并左右儿子的 val 到当前节点的 val
    def _maintain(self, node: int) -> None:
        self._tree[node].val = self._merge_val(self._tree[node * 2].val, self._tree[node * 2 + 1].val)

    # 用 a 初始化线段树
    # 时间复杂度 O(n)
    def _build(self, a: List[int], node: int, l: int, r: int) -> None:
        self._tree[node].todo = self._TODO_INIT
        if l == r:  # 叶子
            self._tree[node].val = a[l]  # 初始化叶节点的值
            return
        m = (l + r) // 2
        self._build(a, node * 2, l, m)  # 初始化左子树
        self._build(a, node * 2 + 1, m + 1, r)  # 初始化右子树
        self._maintain(node)

    def _update(self, node: int, l: int, r: int, ql: int, qr: int, f: int) -> None:
        if ql <= l and r <= qr:  # 当前子树完全在 [ql, qr] 内
            self._apply(node, l, r, f)
            return
        self._spread(node, l, r)
        m = (l + r) // 2
        if ql <= m:  # 更新左子树
            self._update(node * 2, l, m, ql, qr, f)
        if qr > m:  # 更新右子树
            self._update(node * 2 + 1, m + 1, r, ql, qr, f)
        self._maintain(node)

    def _query(self, node: int, l: int, r: int, ql: int, qr: int) -> int:
        if ql <= l and r <= qr:  # 当前子树完全在 [ql, qr] 内
            return self._tree[node].val
        self._spread(node, l, r)
        m = (l + r) // 2
        if qr <= m:  # [ql, qr] 在左子树
            return self._query(node * 2, l, m, ql, qr)
        if ql > m:  # [ql, qr] 在右子树
            return self._query(node * 2 + 1, m + 1, r, ql, qr)
        l_res = self._query(node * 2, l, m, ql, qr)
        r_res = self._query(node * 2 + 1, m + 1, r, ql, qr)
        return self._merge_val(l_res, r_res)

    # 用 f 更新 [ql, qr] 中的每个 a[i]
    # 0 <= ql <= qr <= n-1
    # 时间复杂度 O(log n)
    def update(self, ql: int, qr: int, f: int) -> None:
        self._update(1, 0, self._n - 1, ql, qr, f)

    # 返回用 _merge_val 合并所有 a[i] 的计算结果，其中 i 在闭区间 [ql, qr] 中
    # 0 <= ql <= qr <= n-1
    # 时间复杂度 O(log n)
    def query(self, ql: int, qr: int) -> int:
        return self._query(1, 0, self._n - 1, ql, qr)
```

### ST 表
```python
class SparseTable:
    # 时间复杂度 O(n * log n)
    def __init__(self, nums: List[int], op: Callable[[int, int], int]):
        n = len(nums)
        w = n.bit_length()
        st = [[0] * n for _ in range(w)]
        st[0] = nums[:]
        for i in range(1, w):
            for j in range(n - (1 << i) + 1):
                st[i][j] = op(st[i - 1][j], st[i - 1][j + (1 << (i - 1))])
        self.st = st
        self.op = op

    # [l, r) 左闭右开，下标从 0 开始
    # 返回 op(nums[l:r])
    # 时间复杂度 O(1)
    def query(self, l: int, r: int) -> int:
        k = (r - l).bit_length() - 1
        return self.op(self.st[k][l], self.st[k][r - (1 << k)])


# 使用方法举例
nums = [3, 1, 4, 1, 5, 9, 2, 6]
st = SparseTable(nums, max)
print(st.query(0, 5))  # max(nums[0:5]) = 5
print(st.query(1, 1))  # 错误：必须保证 l < r
```
