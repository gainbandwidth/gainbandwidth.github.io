---
title: "Hot 100 刷题笔记（五）：链表"
date: 2025-01-05
draft: false
categories: [LeetCode]
tags: [LeetCode, 链表, 反转链表, LRU, 算法, Python]
summary: "LeetCode Hot 100 链表专题：反转链表、环形链表、合并链表、K 个一组翻转、LRU/LFU 缓存。"
ShowToc: true
---

链表是数据结构中最基础也最灵活的线性结构之一。本专题涵盖相交链表、反转链表、回文链表、环形链表、合并链表、K 个一组翻转，以及经典的 LRU/LFU 缓存设计。

## 链表

相交链表 简单（善于利用链表长度）

```python
def getIntersectionNode(self, headA: ListNode, headB: ListNode) -> Optional[ListNode]:
    p, q = headA, headB
    while p != q:
        p = p.next if p else headB
        q = q.next if q else headA
    return p
```

反转链表 简单（迭代、递归两种方法）

```python
def reverseList(self, head: Optional[ListNode]) -> Optional[ListNode]:
    pre = None
    cur = head
    while cur:
        nxt = cur.next
        cur.next = pre  # 把 cur 插在 pre 链表的前面（头插法）
        pre = cur
        cur = nxt
    return pre
```

```python
# 92. 反转链表 II
def reverseBetween(self, head, left, right):
    p0 = dummy = ListNode(next=head)
    for _ in range(left - 1):
        p0 = p0.next

    pre = None
    cur = p0.next
    for _ in range(right - left + 1):
        nxt = cur.next
        cur.next = pre
        pre = cur
        cur = nxt

    p0.next.next = cur
    p0.next = pre
    return dummy.next
```

回文链表 简单（超级三合一：链表中点、反转链表、判断回文）

```python
def isPalindrome(self, head: Optional[ListNode]) -> bool:
    # 找到前半部分的尾节点，反转后半部分链表，判断是否回文
    def middleNode(head):
        slow = fast = head
        while fast and fast.next:
            slow = slow.next
            fast = fast.next.next
        return slow

    def reverseList(head):
        pre = None
        cur = head
        while cur:
            nxt = cur.next
            cur.next = pre
            pre = cur
            cur = nxt
        return pre

    mid = middleNode(head)
    head2 = reverseList(mid)
    while head2:
        if head.val != head2.val:
            return False
        head = head.next
        head2 = head2.next
    return True
```

环形链表 简单

环形链表 II 中等（快慢指针、再走a步）

```python
def detectCycle(self, head: Optional[ListNode]) -> Optional[ListNode]:
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
        if fast is slow: # 相遇
            while slow is not head: # 再走 a 步
                slow = slow.next
                head = head.next
            return slow
    return None
```

合并两个有序链表 简单（分情况讨论，创建新链表：dummy、cur = cur.next）（用于后续链表排序、合并K个升序链表）

```python
def mergeTwoLists(self, list1: Optional[ListNode], list2: Optional[ListNode]) -> Optional[ListNode]:
    cur = dummy = ListNode()
    while list1 and list2:
        if list1.val < list2.val:
            cur.next = list1
            list1 = list1.next # 记得这边指针也要不断向前
        else:
            cur.next = list2
            list2 = list2.next
        cur = cur.next # 记得 p 指针不断向前
    cur.next = list1 or list2
    return dummy.next
```

两数相加 中等（分情况讨论，创建新链表：dummy、cur = cur.next）

```python
def addTwoNumbers(self, l1: Optional[ListNode], l2: Optional[ListNode]) -> Optional[ListNode]:
    carry = 0
    cur = dummy = ListNode()
    while l1 or l2 or carry:
        if l1:
            carry += l1.val
            l1 = l1.next
        if l2:
            carry += l2.val
            l2 = l2.next
        cur.next = ListNode(carry % 10)
        carry //= 10
        cur = cur.next
    return dummy.next
```

删除链表的倒数第 N 个结点 中等（注意循环条件 `while right.next`）

```python
def removeNthFromEnd(self, head, n):
    left = right = dummy = ListNode(next=head)
    for i in range(n):
        right = right.next
    # 如果要遍历到倒数第二个节点，需要写 while node.next
    while right.next:
        left = left.next
        right = right.next
    # 左指针的下一个节点就是倒数第 n 个节点
    left.next = left.next.next
    return dummy.next
```

两两交换链表中的节点 中等（迭代、递归两种方法）

```python
def swapPairs(self, head: Optional[ListNode]) -> Optional[ListNode]:
    # 递归
    if head is None or head.next is None:
        return head
    node1 = head
    node2 = node1.next
    node3 = node2.next

    node1.next = self.swapPairs(node3)
    node2.next = node1
    return node2

    # 迭代
    node0 = dummy = ListNode(next=head)
    node1 = head
    while node1 and node1.next:
        node2 = node1.next
        node3 = node2.next

        node0.next = node2 # 0 -> 2
        node2.next = node1 # 2 -> 1
        node1.next = node3 # 1 -> 3

        node0 = node1
        node1 = node3
    return dummy.next
```

**K 个一组翻转链表 困难**

```python
def reverseKGroup(self, head, k):
    # 统计节点个数
    n = 0
    cur = head
    while cur:
        cur = cur.next
        n += 1       
    p0 = dummy = ListNode(next=head)
    pre = None
    cur = head
    while n >= k:
        n -= k
        for _ in range(k):
            nxt = cur.next
            cur.next = pre
            pre = cur
            cur = nxt
        tmp = p0.next
        tmp.next = cur
        p0.next = pre
        p0 = tmp
    return dummy.next
```

随机链表的复制 中等（复制一份1 - 1' - 2 - 2'）

```python
def copyRandomList(self, head: 'Optional[Node]') -> 'Optional[Node]':
    if not head: return
    cur = head
    while cur:
        tmp = Node(cur.val, cur.next)
        cur.next = tmp
        cur = tmp.next
    cur = head
    while cur:
        if cur.random:
            cur.next.random = cur.random.next
        cur = cur.next.next
    pre = head
    cur = res = head.next
    while cur.next:
        pre.next = pre.next.next
        cur.next = cur.next.next
        pre = pre.next
        cur = cur.next
    pre.next = None
    return res
```

排序链表 中等（**归并排序**（分治/迭代）链表的中间节点 + 合并两个有序链表）

```python
def middleNode(self, head):
    slow = fast = head
    while fast and fast.next:
        pre = slow
        slow = slow.next
        fast = fast.next.next
    pre.next = None # 断开 slow 的前一个节点和 slow 的连接
    return slow

def mergeTwoLists(self, list1, list2):
    cur = dummy = ListNode()
    while list1 and list2:
        if list1.val < list2.val:
            cur.next = ListNode(list1.val)
            list1 = list1.next
        else:
            cur.next = ListNode(list2.val)
            list2 = list2.next
        cur = cur.next
    cur.next = list1 if list1 else list2
    return dummy.next

def sortList(self, head):
    if head is None or head.next is None:
        return head

    head2 = self.middleNode(head)
    head = self.sortList(head)
    head2 = self.sortList(head2)
    return self.mergeTwoLists(head, head2)
```

**合并 K 个升序链表 困难**（最小堆/分治/迭代）

```python
def mergeKLists(self, lists: List[Optional[ListNode]]) -> Optional[ListNode]:
    p = dummy = ListNode()
    # 优先队列，最小堆（可以自定义节点比较 ListNode.__lt__ = lambda a, b: a.val < b.val）
    pq = []
    for i, head in enumerate(lists):
        if head: # 可能是空链表
            heappush(pq, (head.val, i, head)) # 加入 i 是因为对于相同的 val 需要比较下一项排序
    while pq:
        val, i, node = heappop(pq)
        p.next = node
        if node.next:
            heappush(pq, (node.next.val, i, node.next))
        p = p.next
    return dummy.next

def mergeKLists(self, lists: List[Optional[ListNode]]) -> Optional[ListNode]:
    m = len(lists)
    if m == 0:
        return None
    if m == 1:
        return lists[0]
    left = self.mergeKLists(lists[:m // 2])
    right = self.mergeKLists(lists[m // 2:])
    # 合并两个有序链表
    return self.mergeTwoLists(left, right)
```

**LRU 缓存 中等**

```python
class Node:
    def __init__(self, key=0, value=0):
        self.key = key
        self.value = value
        self.prev = None
        self.next = None

class LRUCache:
    def __init__(self, capacity):
        self.capacity = capacity
        self.dummy = Node()
        self.dummy.next = self.dummy
        self.dummy.prev = self.dummy
        self.key_to_node = {}
    
    # 获取 key 对应的节点，同时把该节点移到链表头部
    def get_node(self, key):
        if key not in self.key_to_node:
            return None
        node = self.key_to_node[key]
        self.remove(node) # 把这本书抽出来
        self.push_front(node) # 放到最上面
        return node
    
    def get(self, key):
        node = self.get_node(key)
        return node.value if node else -1
    
    def put(self, key, value): # 也可以判断 key 是否在 key_to_value 中
        node = self.get_node(key)
        if node:
            node.value = value # 有这本书 更新 value
            return
        self.key_to_node[key] = node = Node(key, value) # 新书
        self.push_front(node)
        if len(self.key_to_node) > self.capacity: # 书太多了 去掉最后一本书
            back_node = self.dummy.prev
            del self.key_to_node[back_node.key]
            self.remove(back_node)
    
    # 删除一个节点（抽出一本书）
    def remove(self, node):
        node.prev.next = node.next
        node.next.prev = node.prev
    
    # 在链表头添加一个节点（把一本书放到最上面）
    def push_front(self, node):
        node.prev = self.dummy
        node.next = self.dummy.next
        node.prev.next = node
        node.next.prev = node
```

**（补充）带 TTL 的 LRU**

```python
import time
class Node:
    def __init__(self, key=0, value=0, ttl=0):
        self.key = key
        self.value = value
        self.ttl = ttl  # 新增：记录TTL时长（秒）
        self.expire_time = 0  # 新增：记录过期时间戳
        self.prev = None
        self.next = None

class TTLLRUCache:
    def __init__(self, capacity):
        self.capacity = capacity
        self.dummy = Node()
        self.dummy.next = self.dummy
        self.dummy.prev = self.dummy
        self.key_to_node = {}
    
    # 新增：检查节点是否过期
    def is_expired(self, node):
        # ttl=0 表示永不过期
        if node.ttl <= 0:
            return False
        # 当前时间戳 > 过期时间戳 则过期
        return time.time() > node.expire_time
    
    # 重写：获取节点时先检查过期
    def get_node(self, key):
        if key not in self.key_to_node:
            return None
        
        node = self.key_to_node[key]
        # 新增：检查节点是否过期，过期则删除并返回None
        if self.is_expired(node):
            self.remove(node)
            del self.key_to_node[key]
            return None
        
        # 原有逻辑：移到链表头部
        self.remove(node)
        self.push_front(node)
        return node
    
    def get(self, key):
        node = self.get_node(key)
        return node.value if node else -1
    
    # 重写：put时新增ttl参数，记录过期时间
    def put(self, key, value, ttl=0):
        node = self.get_node(key)
        if node:
            # 新增：更新value的同时更新过期时间
            node.value = value
            node.ttl = ttl
            node.expire_time = time.time() + ttl if ttl > 0 else 0
            return
        
        # 新增：在创建新节点前，先清理所有过期节点
        self.clean_expired()
        
        # 清理过期节点后，再创建新节点
        self.key_to_node[key] = node = Node(key, value, ttl)
        node.expire_time = time.time() + ttl if ttl > 0 else 0
        self.push_front(node)
        
        # 此时再判断容量：只有清理过期后仍超容，才淘汰尾部有效节点
        if len(self.key_to_node) > self.capacity:
            back_node = self.dummy.prev
            del self.key_to_node[back_node.key]
            self.remove(back_node)
    
    # 原有方法：删除节点（无改动）
    def remove(self, node):
        node.prev.next = node.next
        node.next.prev = node.prev
    
    # 原有方法：添加节点到头部（无改动）
    def push_front(self, node):
        node.prev = self.dummy
        node.next = self.dummy.next
        node.prev.next = node
        node.next.prev = node

    # 新增：主动清理所有过期节点（可选方法）
    def clean_expired(self):
        current = self.dummy.next
        # 遍历链表，删除所有过期节点
        while current != self.dummy:
            next_node = current.next  # 先记录下一个节点，避免删除后断链
            if self.is_expired(current):
                self.remove(current)
                del self.key_to_node[current.key]
            current = next_node
```

**（补充）LFU**

```python
class Node:
    __slots__ = 'prev', 'next', 'key', 'value', 'freq'
    def __init__(self, key=0, value=0):
        self.key = key
        self.value = value
        self.freq = 1 # 新增对每个节点的访问次数

class LFUCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.key_to_node = {}
        def new_list():
            dummy = Node()
            dummy.prev = dummy
            dummy.next = dummy
            return dummy
        self.freq_to_dummy = defaultdict(new_list) # 哈希表实现多摞书，注意这个地方的初始化
    
    def get_node(self, key):
        if key not in self.key_to_node:
            return None
        node = self.key_to_node[key]
        self.remove(node)
        dummy = self.freq_to_dummy[node.freq]
        if dummy.prev == dummy: # 抽出来后，这摞书是空的
            del self.freq_to_dummy[node.freq] # 移除空链表
            if self.min_freq == node.freq: # min_freq 表示最小看过的次数，这摞书是最左边的
                self.min_freq += 1
        node.freq += 1
        self.push_front(self.freq_to_dummy[node.freq], node)
        return node

    def get(self, key: int) -> int:
        node = self.get_node(key)
        return node.value if node else -1       

    def put(self, key: int, value: int) -> None:
        node = self.get_node(key)
        if node:
            node.value = value
            return
        if len(self.key_to_node) == self.capacity: # 书太多了
            dummy = self.freq_to_dummy[self.min_freq]
            back_node = dummy.prev
            del self.key_to_node[back_node.key]
            self.remove(back_node)
            if dummy.prev == dummy: # 这摞书是空的
                del self.freq_to_dummy[self.min_freq]
        self.key_to_node[key] = node = Node(key, value) # 新书
        self.push_front(self.freq_to_dummy[1], node) # 放在「看过 1 次」的最上面
        self.min_freq = 1
    
    def remove(self, node):
        node.prev.next = node.next
        node.next.prev = node.prev
    
    def push_front(self, dummy, node):
        node.prev = dummy
        node.next = dummy.next
        node.prev.next = node
        node.next.prev = node
```
