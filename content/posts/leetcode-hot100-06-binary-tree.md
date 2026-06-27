---
title: "Hot 100 刷题笔记（六）：二叉树"
date: 2025-01-06
draft: false
categories: [LeetCode]
tags: [二叉树, DFS, BFS, 二叉搜索树, 算法, Python]
summary: "LeetCode Hot 100 二叉树专题：遍历、最大深度、验证 BST、路径总和、最近公共祖先、最大路径和。"
ShowToc: true
---

二叉树是递归思想的最佳载体。本专题从遍历模板出发，涵盖最大深度、翻转/对称、BST 验证、路径总和、最近公共祖先，直至最大路径和等经典题型。

## 二叉树

二叉树的中序遍历 简单（线索二叉树）

```python
def inorderTraversal(self, root: Optional[TreeNode]) -> List[int]:
    # 中序遍历模板
    def traverse(root):
        if not root:
            return
        traverse(root.left)
        ans.append(root.val)
        traverse(root.right)
    ans = []
    traverse(root)
    return ans
		# 线索二叉树
    ans = []
    while root:
        if root.left:
            pre = root.left
            while pre.right and pre.right is not root:
                pre = pre.right
                
            if pre.right is None:
                pre.right = root # 建立线索（回到 root 的路）
                root = root.left # 访问左子树
                continue
            # root 的左子树访问完毕，去掉线索，恢复原样
            pre.right = None

        ans.append(root.val)
        root = root.right
    return ans
```

二叉树的最大深度 简单（递归、DFS、BFS）

```python
# 递归
def maxDepth(self, root):
    if not root:
        return 0
    return 1 + max(self.maxDepth(root.left), self.maxDepth(root.right))
# DFS
def maxDepth(self, root):
    ans = 0
    def dfs(node, depth):
        if not node:
            return
        depth += 1
        nonlocal ans
        ans = max(ans, depth)
        dfs(node.left, depth)
        dfs(node.right, depth)
    dfs(root, 0)
    return ans
# BFS（层序遍历）
def maxDepth(self, root):
    if not root:
        return 0
    ans = []
    q = deque([root])
    while q:
        vals = []
        for _ in range(len(q)):
            node = q.popleft()
            vals.append(node.val)
            if node.left: q.append(node.left)
            if node.right: q.append(node.right)
        ans.append(vals)
    return len(ans)
```

翻转二叉树 简单（递归）

```python
def invertTree(self, root: Optional[TreeNode]) -> Optional[TreeNode]:
    if not root:
        return None
    left = self.invertTree(root.left)
    right = self.invertTree(root.right)
    root.left = right
    root.right = left
    return root
```

对称二叉树 简单

```python
def isSymmetric(self, root: Optional[TreeNode]) -> bool:
    def mirrorSymmetric(root1, root2):
        if root1 is None or root2 is None:
            return root1 is root2
        return root1.val == root2.val and mirrorSymmetric(root1.left, root2.right) and mirrorSymmetric(root1.right, root2.left)

    return mirrorSymmetric(root.left, root.right)
```

**二叉树的直径 简单**

```python
def diameterOfBinaryTree(self, root: TreeNode) -> int:
    ans = 0
    # 返回链的长度，对于叶节点返回 0
    def dfs(node):
        if node is None:
            return -1
        l_len = dfs(node.left) + 1
        r_len = dfs(node.right) + 1
        nonlocal ans
        ans = max(ans, l_len + r_len)
        return max(l_len, r_len)
    dfs(root)
    return ans
```

二叉树的层序遍历 中等

将有序数组转换为二叉搜索树 简单（分治）

```python
def sortedArrayToBST(self, nums: List[int]) -> Optional[TreeNode]:
    if not nums:
        return None
    m = len(nums) // 2
    l = self.sortedArrayToBST(nums[:m])
    r = self.sortedArrayToBST(nums[m + 1:])
    return TreeNode(nums[m], l, r)
```

**验证二叉搜索树 中等**（前中后序方法 / 判断中序是否递增）

```python
# 前序遍历，判断节点是否在区间内
def isValidBST(self, root, left=-inf, right=inf):
    if root is None:
        return True
    x = root.val
    return left < x < right and \
           self.isValidBST(root.left, left, x) and \
           self.isValidBST(root.right, x, right)

# 中序遍历 
pre = -inf
def isValidBST(self, root: Optional[TreeNode]) -> bool:
    if root is None:
        return True
    if not self.isValidBST(root.left):  # 左
        return False
    if root.val <= self.pre:  # 中
        return False
    self.pre = root.val
    return self.isValidBST(root.right)  # 右

# 后序遍历
def isValidBST(self, root: Optional[TreeNode]) -> bool:
    def dfs(node: Optional[TreeNode]) -> Tuple:
        if node is None:
            return inf, -inf
        l_min, l_max = dfs(node.left)
        r_min, r_max = dfs(node.right)
        x = node.val
        # 也可以在递归完左子树之后立刻判断，如果发现不是二叉搜索树，就不用递归右子树了
        if x <= l_max or x >= r_min:
            return -inf, inf
        return min(l_min, x), max(r_max, x)
    return dfs(root)[1] != inf
```

二叉搜索树中第 K 小的元素 中等（中序遍历）

```python
def kthSmallest(self, root, k):
    ans = 0
    def dfs(root):
        nonlocal k, ans
        if not root or k == 0:
            return
        dfs(root.left)
        k -= 1
        if k == 0:
            ans = root.val
        dfs(root.right)
    dfs(root)
    return ans
```

二叉树的右视图 中等（层序遍历）

```python
def rightSideView(self, root: Optional[TreeNode]) -> List[int]:
    # 层序遍历
    if not root: return []
    ans = []
    q = deque([root])
    while q:
        ans.append(q[-1].val)
        for _ in range(len(q)):
            cur = q.popleft()
            if cur.left: q.append(cur.left)
            if cur.right: q.append(cur.right)
    return ans

    # DFS
    ans = []
    def dfs(node: Optional[TreeNode], depth: int) -> None:
        if node is None:
            return
        if depth == len(ans):  # 这个深度首次遇到
            ans.append(node.val)
        dfs(node.right, depth + 1)  # 先递归右子树，保证首次遇到的一定是最右边的节点
        dfs(node.left, depth + 1)
    dfs(root, 0)
    return ans
```

二叉树展开为链表 中等（类似中序遍历的线索二叉树）

```python
def flatten(self, root: Optional[TreeNode]) -> None:
    while root:
        # 左子树为 null，直接考虑下一个节点
        if not root.left:
            root = root.right
        else:
            # 找左子树最右边的节点
            pre = root.left
            while pre.right:
                pre = pre.right
            pre.right = root.right
            root.right = root.left
            root.left = None
            # 考虑下一个节点
            root = root.right
		
    # 方法二：分治 + 拼接
    if root is None:
        return
    self.flatten(root.left)
    self.flatten(root.right)
    left = root.left
    right = root.right
    root.left = None
    root.right = left
    cur = root
    while cur.right:
        cur = cur.right
    cur.right = right
```

从前序与中序遍历序列构造二叉树 中等（递归）

```python
def buildTree(self, preorder, inorder):
    # 递归
    if not preorder:
        return None
    m = inorder.index(preorder[0])
    l = self.buildTree(preorder[1: 1 + m], inorder[:m])
    r = self.buildTree(preorder[1 + m:], inorder[1 + m:])
    return TreeNode(preorder[0], l, r)
		
    # DFS（记录每一段起始、结束位置）
    index = (x: i for i, x in enumerate(inorder))
    def dfs(pre_l, pre_r, in_l):
        if pre_l == pre_r:
            return None
        leftSize = index(preorder[pre_l]) - in_l
        left = dfs(pre_l + 1, pre_l + 1 + leftSize, in_l)
        right = dfs(pre_l + 1 + leftSize, pre_r, in_l + 1 + leftSize)
        return TreeNode(preorder[pre_l], left, right)
    return dfs(0, len(preorder), 0)
```

**（补充）路径总和 II**（根节点到叶节点的路径总和）

```python
def pathSum(self, root: Optional[TreeNode], targetSum: int) -> List[List[int]]:
    ans = []
    path = []
    def dfs(node, targetSum):
        if node is None:
            return
        path.append(node.val)
        targetSum -= node.val
        if node.left is None and node.right is None and targetSum == 0:
            ans.append(path.copy())
        else:
            dfs(node.left, targetSum)
            dfs(node.right, targetSum)
        path.pop()

    dfs(root, targetSum)
    return ans
```

**路径总和 III 中等**（方法同 和为 K 的子数组，枚举右端点，统计左端点）

```python
def pathSum(self, root: Optional[TreeNode], targetSum: int) -> int:
    # 枚举路径的终点，统计有多少个起点
    # 一边遍历二叉树，一边用哈希表 cnt 统计前缀和（从根节点开始的路径和）的出现次数。
    # 设从根到终点 node 的路径和为 s，那么起点的个数就是 cnt[s-targetSum]
    ans = 0
    cnt = defaultdict(int)
    cnt[0] = 1

    # s 表示从根到 node 的父节点的节点值之和（node 的节点值尚未计入）
    def dfs(node, s):
        if node is None:
            return
        nonlocal ans
        s += node.val
        # 把 node 当作路径的终点，统计有多少个起点
        ans += cnt[s - targetSum]
        cnt[s] += 1
        dfs(node.left, s)
        dfs(node.right, s)
        cnt[s] -= 1 # 恢复现场（撤销 cnt[s] += 1）

    dfs(root, 0)
    return ans
```

**二叉树的最近公共祖先 中等**

```python
def lowestCommonAncestor(self, root: 'TreeNode', p: 'TreeNode', q: 'TreeNode') -> 'TreeNode':
    if root is None or root is p or root is q:
        return root
    left = self.lowestCommonAncestor(root.left, p, q)
    right = self.lowestCommonAncestor(root.right, p, q)
    if left and right: # 同时出现在左右，返回根节点
        return root
    if left:
        return left
    return right
```

**二叉树中的最大路径和 困难**（和二叉树的直径类似，将直径长度的运算换成路径和）

```python
def maxPathSum(self, root: Optional[TreeNode]) -> int:
    ans = -inf
    def dfs(node):
        if node is None:
            return 0
        L = dfs(node.left)
        R = dfs(node.right)
        nonlocal ans
        ans = max(ans, L + R + node.val)
        return max(max(L, R) + node.val, 0)

    dfs(root)
    return ans
```

（补充）二叉树的锯齿形层序遍历

```python
def zigzagLevelOrder(self, root: Optional[TreeNode]) -> List[List[int]]:
    if not root: return []
    q = deque([root])
    ans = []
    order = True
    while q:
        sz = len(q)
        tmp = []
        for _ in range(sz):
            cur = q.popleft()
            tmp.append(cur.val)
            if cur.left: q.append(cur.left)
            if cur.right: q.append(cur.right)
        if order:
            ans.append(tmp)
        else:
            ans.append(list(reversed(tmp)))
        order = not order
    return ans
```

（补充）二叉树最大宽度

```python
def widthOfBinaryTree(self, root: Optional[TreeNode]) -> int:
    # DFS
    levelMin = {}
    def dfs(node, depth, index): # 也可以没有返回值，用nonlocal ans更新
        if node is None:
            return 0
        if depth not in levelMin:
            levelMin[depth] = index
        return max(index - levelMin[depth] + 1, 
                   dfs(node.left, depth + 1, index * 2), 
                   dfs(node.right, depth + 1, index * 2 + 1))
    return dfs(root, 1, 1)
    # BFS
    ans = 0
    arr = [[root, 1]]
    while arr:
        tmp = []
        for node, index in arr:
            if node.left:
                tmp.append([node.left, index * 2])
            if node.right:
                tmp.append([node.right, index * 2 + 1])
        ans = max(ans, arr[-1][1] - arr[0][1] + 1)
        arr = tmp
    return ans
```
