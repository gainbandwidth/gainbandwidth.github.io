---
title: "Hot 100 刷题笔记（七）：图论"
date: 2025-01-07
draft: false
categories: [LeetCode]
tags: [图论, DFS, BFS, 拓扑排序, Dijkstra, 并查集, 算法, Python]
summary: "LeetCode Hot 100 图论专题：岛屿数量、拓扑排序、Trie、二分图判定、最小生成树、Dijkstra 最短路径。"
ShowToc: true
---

图论是算法面试中的重点和难点。本专题从 DFS/BFS 经典题（岛屿数量、腐烂的橘子）出发，经过拓扑排序、Trie 前缀树、并查集，一直延伸到最小生成树（Kruskal/Prim）和 Dijkstra 最短路径。

## 图论

**岛屿数量 中等**（<u>递归入口、递归方向、递归边界</u>——学会与二叉树类比）

```python
def numIslands(self, grid: List[List[str]]) -> int:
    m, n = len(grid), len(grid[0])

    def dfs(i, j):
        if i < 0 or i >= m or j < 0 or j >= n or grid[i][j] != '1':
            return
        grid[i][j] = '2'
        dfs(i, j - 1)
        dfs(i, j + 1)
        dfs(i + 1, j)
        dfs(i - 1, j)

    ans = 0
    for i, row in enumerate(grid):
        for j, c in enumerate(row):
            if c == '1':
                dfs(i, j)
                ans += 1
    return ans
```

（补充）岛屿的最大面积（也可以使用栈 / 队列，模拟DFS / BFS）

```python
def maxAreaOfIsland(self, grid: List[List[int]]) -> int:
    m, n = len(grid), len(grid[0])
    ans = 0
    def dfs(i, j):
        if grid[i][j] != 1:
            return
        nonlocal cnt
        cnt += 1
        grid[i][j] = 2
        for x, y in (i - 1, j), (i + 1, j), (i, j + 1), (i, j - 1):
            if 0 <= x < m and 0 <= y < n:
                dfs(x, y)

    for i in range(m):
        for j in range(n):
            if grid[i][j] == 1:
                cnt = 0
                dfs(i, j)
                ans = max(ans, cnt)
    return ans
```

**腐烂的橘子 中等**（队列记录每一轮腐烂的橘子）

```python
def orangesRotting(self, grid: List[List[int]]) -> int:
    m, n = len(grid), len(grid[0])
    q = []
    fresh = 0
    for i, row in enumerate(grid):
        for j, x in enumerate(row):
            if x == 1:
                fresh += 1
            elif x == 2:
                q.append((i, j))

    ans = 0
    while q and fresh:
        ans += 1
        tmp = q
        q = []
        for x, y in tmp:
            for i, j in (x + 1, y), (x - 1, y), (x, y + 1), (x, y - 1):
                if 0 <= i < m and 0 <= j < n and grid[i][j] == 1:
                    grid[i][j] = 2
                    fresh -= 1
                    q.append((i, j))

    return -1 if fresh else ans
```

**课程表 中等**——拓扑排序（判断有向图是否有环 DFS / BFS）

```python
# BFS
def canFinish(self, numCourses, prerequisites):
    g = [[] for _ in range(numCourses)]
    indegree = [0] * numCourses
    for a, b in prerequisites:
        g[b].append(a)
        indegree[a] += 1
    q = deque([i for i in range(numCourses) if indegree[i] == 0])
    cnt = 0
    while q:
        x = q.popleft()
        cnt += 1
        for y in g[x]:
            indegree[y] -= 1
            if indegree[y] == 0:
                q.append(y)
    return cnt == numCourses

# DFS
def canFinish(self, numCourses, prerequisites):
    g = [[] for _ in range(numCourses)]
    for a, b in prerequisites:
        g[b].append(a)
    # 0：节点 x 尚未被访问到。
    # 1：节点 x 正在访问中，dfs(x) 尚未结束。
    # 2：节点 x 已经完全访问完毕。注意这还说明从 x 出发无法找到环。所以当我们遇到状态值为 2 的节点 x 时，无需递归 x。
    colors = [0] * numCourses

    # 返回 True 表示找到了环
    def dfs(x):
        colors[x] = 1
        for y in g[x]:
            # 情况一：colors[y] == 1，表示发生循环依赖，找到了环
            # 情况二：colors[y] == 0，未知，继续递归 y 获取信息
            # 情况三：colors[y] == 2，继续递归 y 只会重蹈覆辙，和之前一样无法找到环
            if colors[y] == 1 or colors[y] == 0 and dfs(y):
                return True
        colors[x] = 2
        return False

    for i, c in enumerate(colors):
        if c == 0 and dfs(i):
            return False
    return True
```

**实现 Trie (前缀树) 中等**

```python
class Trie:
    def __init__(self):
        self.children = [None] * 26
        self.isEnd = False
    
    def searchPrefix(self, prefix: str) -> "Trie":
        node = self
        for ch in prefix:
            ch = ord(ch) - ord("a")
            if not node.children[ch]:
                return None
            node = node.children[ch]
        return node

    def insert(self, word: str) -> None:
        node = self
        for ch in word:
            ch = ord(ch) - ord("a")
            if not node.children[ch]:
                node.children[ch] = Trie()
            node = node.children[ch]
        node.isEnd = True

    def search(self, word: str) -> bool:
        node = self.searchPrefix(word)
        return node is not None and node.isEnd

    def startsWith(self, prefix: str) -> bool:
        return self.searchPrefix(prefix) is not None
```

（补充）单词替换（字典树的应用）

```python
class Trie:
    def __init__(self):
        self.next = defaultdict(Trie)
        self.end = False
    def add(self, word):
        cur = self
        for c in word:
            cur = cur.next[c]
            if cur.end:
                return
        cur.end = True
    def contain_prefix(self, word):
        cur = self
        prefix = ''
        for c in word:
            if c in cur.next:
                cur = cur.next[c]
                prefix += c
            else:
                return False
            if cur.end:
                return prefix
        return False

class Solution:
    def replaceWords(self, dictionary: List[str], sentence: str) -> str:
        trie = Trie()
        for word in dictionary:
            trie.add(word)
        ans = []
        for word in sentence.split():
            if prefix := trie.contain_prefix(word):
                ans.append(prefix)
            else:
                ans.append(word)
        return ' '.join(ans)
```

（补充）单词接龙 困难

```python
def ladderLength(self, beginWord: str, endWord: str, wordList: List[str]) -> int:
    def addWord(word):
        if word not in wordId:
            nonlocal nodeNum
            wordId[word] = nodeNum
            nodeNum += 1

    def addEdge(word):
        addWord(word)
        id1 = wordId[word]
        chars = list(word)
        # n 个虚拟节点（比如对于单词 hit，我们创建三个虚拟节点 *it、h*t、hi*）
        for i in range(len(chars)):
            tmp = chars[i]
            chars[i] = "*"
            newWord = "".join(chars)
            addWord(newWord)
            id2 = wordId[newWord]
            edge[id1].append(id2)
            edge[id2].append(id1)
            chars[i] = tmp

    wordId = dict()
    edge = collections.defaultdict(list)
    nodeNum = 0

    for word in wordList:
        addEdge(word)

    addEdge(beginWord)
    if endWord not in wordId:
        return 0

    '''
    dis = [inf] * nodeNum
    beginId, endId = wordId[beginWord], wordId[endWord]
    dis[beginId] = 0

    q = deque([beginId])
    while q:
        x = q.popleft()
        if x == endId:
            return dis[endId] // 2 + 1
        for it in edge[x]:
            if dis[it] == inf:
                dis[it] = dis[x] + 1
                q.append(it)
    '''
    # 双向广度优先搜索
    disBegin = [inf] * nodeNum
    beginId = wordId[beginWord]
    disBegin[beginId] = 0
    queBegin = deque([beginId])

    disEnd = [inf] * nodeNum
    endId = wordId[endWord]
    disEnd[endId] = 0
    queEnd = deque([endId])

    while queBegin or queEnd:
        queBeginSize = len(queBegin)
        for _ in range(queBeginSize):
            nodeBegin = queBegin.popleft()
            if disEnd[nodeBegin] != inf:
                return (disBegin[nodeBegin] + disEnd[nodeBegin]) // 2 + 1
            for it in edge[nodeBegin]:
                if disBegin[it] == inf:
                    disBegin[it] = disBegin[nodeBegin] + 1
                    queBegin.append(it)

        queEndSize = len(queEnd)
        for _ in range(queEndSize):
            nodeEnd = queEnd.popleft()
            if disBegin[nodeEnd] != inf:
                return (disBegin[nodeEnd] + disEnd[nodeEnd]) // 2 + 1
            for it in edge[nodeEnd]:
                if disEnd[it] == inf:
                    disEnd[it] = disEnd[nodeEnd] + 1
                    queEnd.append(it)

    return 0
```

## 图论补充

### 环检测、拓扑排序（BFS/DFS）

一、BFS （基于入度表，Kahn 算法）

1. 统计每个节点的入度，初始化入度为 0 的节点入队；
2. 依次弹出队列节点，将其邻接节点入度减 1，若邻接节点入度变为 0 则入队；
3. 最终若拓扑排序结果长度等于节点总数 → 无环；否则 → 有环。

```python
from collections import deque

def topological_sort_bfs(n, edges):
    """
    :param n: 节点总数（节点编号建议 0~n-1）
    :param edges: 边列表，格式 [[u1, v1], [u2, v2], ...] 表示 u→v
    :return: (是否有环, 拓扑排序结果)
    """
    # 1. 构建邻接表 + 入度表
    adj = [[] for _ in range(n)]  # 邻接表
    in_degree = [0] * n           # 入度表
    for u, v in edges:
        adj[u].append(v)
        in_degree[v] += 1
    
    # 2. 初始化队列（入度为 0 的节点）
    q = deque([i for i in range(n) if in_degree[i] == 0])
    topo_order = []
    
    # 3. BFS 核心
    while q:
        u = q.popleft()
        topo_order.append(u)
        for v in adj[u]:
            in_degree[v] -= 1
            if in_degree[v] == 0:
                q.append(v)
    
    # 4. 环检测：拓扑排序长度 < 节点数 → 有环
    has_cycle = len(topo_order) != n
    return has_cycle, topo_order

# 测试示例
if __name__ == "__main__":
    n = 4
    edges = [[0,1], [1,2], [2,3], [1,3]]  # 无环
    # edges = [[0,1], [1,2], [2,1]]       # 有环（1→2→1）
    has_cycle, topo = topological_sort_bfs(n, edges)
    print(f"是否有环：{has_cycle}")
    print(f"拓扑排序：{topo}")
```

二、DFS （基于访问状态标记）

1. 标记节点状态：0=未访问，1=访问中（递归栈中），2=已访问；
2. 递归遍历节点，若遇到状态为 1 的节点 → 有环；
3. 递归结束后将节点加入拓扑排序（结果需反转）。

```python
def topological_sort_dfs(n, edges):
    """
    :param n: 节点总数（节点编号 0~n-1）
    :param edges: 边列表 [[u1, v1], [u2, v2], ...] 表示 u→v
    :return: (是否有环, 拓扑排序结果)
    """
    # 1. 构建邻接表
    adj = [[] for _ in range(n)]
    for u, v in edges:
        adj[u].append(v)
    
    # 状态：0=未访问，1=访问中，2=已访问
    state = [0] * n
    topo_order = []
    has_cycle = False
    
    def dfs(u):
        nonlocal has_cycle
        if has_cycle:  # 提前终止
            return
        state[u] = 1   # 标记为访问中
        for v in adj[u]:
            if state[v] == 0:
                dfs(v)
            elif state[v] == 1:  # 遇到递归栈中的节点 → 环
                has_cycle = True
                return
        state[u] = 2   # 标记为已访问
        topo_order.append(u)  # 后序加入
    
    # 2. 遍历所有节点（处理非连通图）
    for i in range(n):
        if state[i] == 0 and not has_cycle:
            dfs(i)
    
    # 3. 拓扑排序结果需反转（后序遍历 → 正序拓扑）
    topo_order.reverse()
    return has_cycle, topo_order

# 测试示例
if __name__ == "__main__":
    n = 4
    edges = [[0,1], [1,2], [2,3], [1,3]]  # 无环
    # edges = [[0,1], [1,2], [2,1]]       # 有环
    has_cycle, topo = topological_sort_dfs(n, edges)
    print(f"是否有环：{has_cycle}")
    print(f"拓扑排序：{topo}")
```

1. **BFS 特点**：基于入度表，逻辑直观，无需递归，适合处理大规模图；拓扑排序结果是"按入度为 0 依次弹出"的顺序。
2. **DFS 特点**：基于递归和状态标记，能同时完成环检测和拓扑排序；拓扑排序需反转后序遍历结果。
3. **环检测核心**：BFS 看拓扑长度是否等于节点数，DFS 看是否遇到"访问中"的节点。

注意事项：

- 节点编号若不是 0~n-1，可先通过字典映射为连续整数；
- 无向图环检测需额外处理"父节点"（避免误判 u-v-u 为环），但拓扑排序仅适用于有向图。

### 二分图判定（BFS/DFS）

二分图的核心判定规则：**能否用两种颜色对图中所有节点染色，使得相邻节点颜色不同**（无奇数长度环）。

一、BFS （基于队列+染色）

1. 初始化颜色数组（-1 表示未染色，0/1 为两种颜色）；
2. 遍历未染色节点，将其染色并加入队列；
3. 依次弹出队列节点，对邻接节点染色：若未染色则染相反色并入队，若已染色且颜色相同 → 非二分图。

```python
from collections import deque

def is_bipartite_bfs(n, edges):
    """
    :param n: 节点总数（编号 0~n-1）
    :param edges: 边列表 [[u1, v1], [u2, v2], ...]（无向图）
    :return: 是否为二分图
    """
    # 1. 构建邻接表（无向图：u→v 且 v→u）
    adj = [[] for _ in range(n)]
    for u, v in edges:
        adj[u].append(v)
        adj[v].append(u)
    
    # 颜色数组：-1=未染色，0/1=两种颜色
    color = [-1] * n
    
    # 处理非连通图（遍历所有未染色节点）
    for i in range(n):
        if color[i] == -1:
            q = deque()
            q.append(i)
            color[i] = 0  # 初始染为0
            
            while q:
                u = q.popleft()
                for v in adj[u]:
                    if color[v] == -1:
                        # 邻接节点染相反色
                        color[v] = color[u] ^ 1
                        q.append(v)
                    elif color[v] == color[u]:
                        # 相邻节点颜色相同 → 非二分图
                        return False
    return True

# 测试示例
if __name__ == "__main__":
    n1, edges1 = 4, [[0,1], [1,2], [2,3], [3,0]]  # 是二分图（偶环）
    n2, edges2 = 3, [[0,1], [1,2], [2,0]]          # 非二分图（奇环）
    print(is_bipartite_bfs(n1, edges1))  # True
    print(is_bipartite_bfs(n2, edges2))  # False
```

二、DFS （基于递归+染色）

1. 初始化颜色数组（-1 未染色）；
2. 递归遍历节点，对当前节点染色后，递归处理邻接节点：
   - 邻接节点未染色 → 染相反色并递归；
   - 邻接节点已染色且颜色相同 → 非二分图。

```python
def is_bipartite_dfs(n, edges):
    """
    :param n: 节点总数（编号 0~n-1）
    :param edges: 边列表 [[u1, v1], [u2, v2], ...]（无向图）
    :return: 是否为二分图
    """
    # 1. 构建邻接表（无向图）
    adj = [[] for _ in range(n)]
    for u, v in edges:
        adj[u].append(v)
        adj[v].append(u)
    
    # 颜色数组：-1=未染色，0/1=两种颜色
    color = [-1] * n
    
    def dfs(u, c):
        color[u] = c  # 给当前节点染色
        for v in adj[u]:
            if color[v] == -1:
                # 邻接节点未染色 → 染相反色并递归
                if not dfs(v, c ^ 1):
                    return False
            elif color[v] == c:
                # 相邻节点颜色相同 → 非二分图
                return False
        return True
    
    # 处理非连通图
    for i in range(n):
        if color[i] == -1:
            if not dfs(i, 0):
                return False
    return True

# 测试示例
if __name__ == "__main__":
    n1, edges1 = 4, [[0,1], [1,2], [2,3], [3,0]]  # 是二分图
    n2, edges2 = 3, [[0,1], [1,2], [2,0]]          # 非二分图
    print(is_bipartite_dfs(n1, edges1))  # True
    print(is_bipartite_dfs(n2, edges2))  # False
```

总结

1. **核心逻辑**：二分图判定的本质是"双色染色"，相邻节点必须颜色不同，存在奇数长度环则无法染色。
2. **BFS 特点**：迭代实现，队列控制遍历顺序，适合处理大规模图，避免递归栈溢出。
3. **DFS 特点**：递归实现，代码更简洁，逻辑直观，但需注意递归深度（节点数过多时可手动设置递归深度）。
4. **通用适配**：若节点编号非 0~n-1，可先通过字典映射为连续整数；无向图需在邻接表中双向添加边。

### 并查集模板

```python
class UnionFind:
    def __init__(self, n: int):
        # 一开始有 n 个集合 {0}, {1}, ..., {n-1}
        # 集合 i 的代表元是自己
        self._fa = list(range(n))  # 代表元
        self.cc = n  # 连通块个数

    # 返回 x 所在集合的代表元
    # 同时做路径压缩，也就是把 x 所在集合中的所有元素的 fa 都改成代表元
    def find(self, x: int) -> int:
        # 如果 fa[x] == x，则表示 x 是代表元
        if self._fa[x] != x:
            self._fa[x] = self.find(self._fa[x])  # fa 改成代表元
        return self._fa[x]

    # 把 from 所在集合合并到 to 所在集合中
    # 返回是否合并成功
    def merge(self, from_: int, to: int) -> bool:
        x, y = self.find(from_), self.find(to)
        if x == y:  # from 和 to 在同一个集合，不做合并
            return False
        self._fa[x] = y  # 合并集合。修改后就可以认为 from 和 to 在同一个集合了
        self.cc -= 1  # 成功合并，连通块个数减一
        return True
```

### 最小生成树（Kruskal/Prim）
最小生成树（MST）适用于**无向连通图**，核心目标是用最小总权重连接所有节点且无环。以下是简洁通用的模板，分别实现 Kruskal（按边排序+并查集）和 Prim（贪心选点+BFS/堆）。

一、Kruskal 算法模板（按边贪心 + 并查集）

1. 将所有边按权重从小到大排序；
2. 用并查集（Union-Find）维护连通性，依次遍历边：
   - 若边的两个节点不在同一连通分量 → 加入 MST，合并分量；
   - 若已连通 → 跳过（避免环）；
3. 直到 MST 包含 `n-1` 条边（n 为节点数）。

```python
class UnionFind:
    """并查集辅助类（路径压缩 + 按秩合并）"""
    def __init__(self, n):
        self.parent = list(range(n))  # 父节点
        self.rank = [1] * n           # 秩（树高度）
    
    def find(self, x):
        """查找根节点（路径压缩）"""
        if self.parent[x] != x:
            self.parent[x] = self.find(self.parent[x])
        return self.parent[x]
    
    def union(self, x, y):
        """合并两个集合（按秩合并）"""
        fx, fy = self.find(x), self.find(y)
        if fx == fy:
            return False  # 已连通，合并失败
        # 矮树挂到高树下
        if self.rank[fx] < self.rank[fy]:
            self.parent[fx] = fy
        else:
            self.parent[fy] = fx
            if self.rank[fx] == self.rank[fy]:
                self.rank[fx] += 1
        return True

def kruskal(n, edges):
    """
    :param n: 节点总数（编号 0~n-1）
    :param edges: 边列表 [[u1, v1, w1], [u2, v2, w2], ...]（u,v为节点，w为权重）
    :return: (MST的总权重, MST的边列表)；若图不连通返回 (None, None)
    """
    # 1. 按边权重升序排序
    edges_sorted = sorted(edges, key=lambda x: x[2])
    uf = UnionFind(n)
    mst_weight = 0
    mst_edges = []
    
    # 2. 遍历边，贪心选边
    for u, v, w in edges_sorted:
        if uf.union(u, v):
            mst_weight += w
            mst_edges.append([u, v, w])
            # 已选够 n-1 条边，提前终止
            if len(mst_edges) == n - 1:
                break
    
    # 检查是否连通（所有节点是否在同一集合）
    root = uf.find(0)
    for i in range(1, n):
        if uf.find(i) != root:
            return None, None  # 图不连通，无MST
    
    return mst_weight, mst_edges

# 测试示例
if __name__ == "__main__":
    n = 4
    # 边列表：[u, v, weight]
    edges = [[0,1,1], [0,2,3], [1,2,1], [1,3,5], [2,3,2]]
    weight, mst_edges = kruskal(n, edges)
    if weight is None:
        print("图不连通，无最小生成树")
    else:
        print(f"MST总权重：{weight}")  # 预期输出：4（0-1,1-2,2-3）
        print(f"MST边列表：{mst_edges}")
```

二、Prim 算法模板（按点贪心 + 最小堆）

1. 从任意节点（如 0）开始，维护一个"已选节点集合"；
2. 用最小堆存储"已选集合 → 未选节点"的边，每次选权重最小的边；
3. 将选中的节点加入已选集合，更新堆中边的信息；
4. 直到已选集合包含所有节点。

```python
import heapq

def prim(n, edges):
    """
    :param n: 节点总数（编号 0~n-1）
    :param edges: 边列表 [[u1, v1, w1], [u2, v2, w2], ...]（u,v为节点，w为权重）
    :return: (MST的总权重, MST的边列表)；若图不连通返回 (None, None)
    """
    # 1. 构建邻接表（无向图：u→(v,w) 且 v→(u,w)）
    adj = [[] for _ in range(n)]
    for u, v, w in edges:
        adj[u].append((v, w))
        adj[v].append((u, w))
    
    visited = [False] * n  # 标记已选节点
    mst_weight = 0
    mst_edges = []
    heap = []
    
    # 2. 从节点0开始初始化堆
    visited[0] = True
    for v, w in adj[0]:
        heapq.heappush(heap, (w, 0, v))  # (权重, 已选节点, 未选节点)
    
    # 3. 贪心选边
    while heap and len(mst_edges) < n - 1:
        w, u, v = heapq.heappop(heap)
        if visited[v]:
            continue  # 节点v已选，跳过
        # 选中这条边，加入MST
        visited[v] = True
        mst_weight += w
        mst_edges.append([u, v, w])
        # 将v的邻接边加入堆
        for nv, nw in adj[v]:
            if not visited[nv]:
                heapq.heappush(heap, (nw, v, nv))
    
    # 检查是否所有节点都被访问（图是否连通）
    if not all(visited):
        return None, None
    
    return mst_weight, mst_edges

# 测试示例
if __name__ == "__main__":
    n = 4
    edges = [[0,1,1], [0,2,3], [1,2,1], [1,3,5], [2,3,2]]
    weight, mst_edges = prim(n, edges)
    if weight is None:
        print("图不连通，无最小生成树")
    else:
        print(f"MST总权重：{weight}")  # 预期输出：4
        print(f"MST边列表：{mst_edges}")
```

总结

1. **Kruskal 特点**：
   - 核心是**并查集**，按边排序后贪心选边，适合**边少的稀疏图**；
   - 易实现，且能快速判断图是否连通。
2. **Prim 特点**：
   - 核心是**最小堆**，按点扩展贪心选边，适合**节点少的稠密图**；
   - 时间效率更优（堆优化后为 O(E log V)）。
3. **通用注意**：
   - 输入需为**无向连通图**，否则返回 `None`；
   - 节点编号若非 0~n-1，可先映射为连续整数；
   - 两者最终生成的 MST 总权重一致，边列表可能因选边顺序不同略有差异。

### 最短路径（Dijkstra 算法）
Dijkstra 算法适用于**有向/无向图**、**边权重非负**的场景，核心目标是求从起点到所有其他节点的最短路径。以下是基于**最小堆（优先队列）** 优化的简洁模板，兼顾可读性和效率。

1. 初始化距离数组：起点距离为 0，其余节点为无穷大；
2. 用最小堆存储（当前最短距离，节点），初始时将起点入堆；
3. 每次弹出堆顶（距离最小的节点），遍历其邻接节点：若通过当前节点到达邻接节点的距离更短 → 更新距离，并将邻接节点入堆；
4. 重复直到堆为空，最终距离数组即为起点到各节点的最短路径。

```python
import heapq

def dijkstra(n, edges, start):
    """
    :param n: 节点总数（编号 0~n-1）
    :param edges: 边列表，格式 [[u1, v1, w1], [u2, v2, w2], ...]
                  - 有向图：u→v，权重w；无向图需添加 [v, u, w]
    :param start: 起点编号
    :return: 距离数组 dist（dist[i] 表示起点到i的最短距离，不可达则为inf）
    """
    # 1. 构建邻接表：adj[u] = [(v, w)] 表示u→v的边，权重w
    adj = [[] for _ in range(n)]
    for u, v, w in edges:
        adj[u].append((v, w))
    
    # 2. 初始化距离数组：inf表示不可达，起点距离为0
    INF = float('inf')
    dist = [INF] * n
    dist[start] = 0
    
    # 3. 最小堆：存储 (当前最短距离, 节点)，初始入堆起点
    heap = []
    heapq.heappush(heap, (0, start))
    
    # 4. 核心遍历
    while heap:
        current_dist, u = heapq.heappop(heap)
        # 剪枝：若当前距离大于已记录的最短距离，直接跳过
        if current_dist > dist[u]:
            continue
        # 遍历邻接节点
        for v, w in adj[u]:
            # 松弛操作：更新最短距离
            if dist[v] > dist[u] + w:
                dist[v] = dist[u] + w
                heapq.heappush(heap, (dist[v], v))
    
    return dist

# 测试示例
if __name__ == "__main__":
    # 示例1：有向图
    n1 = 5
    edges1 = [[0,1,2], [0,2,5], [1,2,1], [1,3,4], [2,3,2], [3,4,3]]
    start1 = 0
    dist1 = dijkstra(n1, edges1, start1)
    print("有向图最短距离：", dist1)  # 预期：[0, 2, 3, 5, 8]
    
    # 示例2：无向图（需双向加边）
    n2 = 4
    edges2 = [[0,1,1], [0,2,4], [1,2,2], [1,3,5], [2,3,1]]
    # 无向图需补充反向边
    edges2 += [[v, u, w] for u, v, w in edges2]
    start2 = 0
    dist2 = dijkstra(n2, edges2, start2)
    print("无向图最短距离：", dist2)  # 预期：[0, 1, 3, 4]
```

进阶：记录最短路径（可选）

若需要输出从起点到目标节点的具体路径，可在模板中增加**前驱节点数组**：

```python
import heapq

def dijkstra_with_path(n, edges, start, end):
    """
    扩展版：返回起点到终点的最短距离 + 具体路径
    :param end: 终点编号
    :return: (最短距离, 路径列表)；不可达则返回 (inf, [])
    """
    adj = [[] for _ in range(n)]
    for u, v, w in edges:
        adj[u].append((v, w))
    
    INF = float('inf')
    dist = [INF] * n
    dist[start] = 0
    # 前驱节点数组：prev[v] 表示到达v的最短路径中，v的前一个节点
    prev = [-1] * n
    
    heap = []
    heapq.heappush(heap, (0, start))
    
    while heap:
        current_dist, u = heapq.heappop(heap)
        if current_dist > dist[u]:
            continue
        for v, w in adj[u]:
            if dist[v] > dist[u] + w:
                dist[v] = dist[u] + w
                prev[v] = u  # 记录前驱
                heapq.heappush(heap, (dist[v], v))
    
    # 回溯构建路径
    path = []
    if dist[end] == INF:
        return dist[end], path
    # 从终点倒推到起点，再反转
    node = end
    while node != -1:
        path.append(node)
        node = prev[node]
    path.reverse()
    
    return dist[end], path

# 测试路径输出
if __name__ == "__main__":
    n = 5
    edges = [[0,1,2], [0,2,5], [1,2,1], [1,3,4], [2,3,2], [3,4,3]]
    start, end = 0, 4
    distance, path = dijkstra_with_path(n, edges, start, end)
    print(f"起点{start}到终点{end}的最短距离：{distance}")  # 8
    print(f"最短路径：{path}")  # [0,1,2,3,4]
```

总结

1. **核心要点**：
   - Dijkstra 算法依赖**最小堆**优化，时间复杂度为 $O(E \log V)$（E为边数，V为节点数）；
   - 仅适用于**边权重非负**的场景，若有负权边需用 Bellman-Ford 或 SPFA 算法。
2. **使用注意**：
   - 无向图需在邻接表中双向添加边；
   - 剪枝操作（`current_dist > dist[u]`）可避免处理堆中过时的无效数据，提升效率；
   - 扩展版通过**前驱数组**回溯路径，是最短路径问题的常见需求。
3. **适用场景**：
   - 单源最短路径（如地图导航、网络路由）、边权重非负的有向/无向图。
