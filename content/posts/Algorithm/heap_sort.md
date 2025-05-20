---
title: 'Heap_sort'
date: 2025-05-20T17:25:47+08:00
lastmod: 2025-05-20T17:25:47+08:00
author: ["Bigodf"]

tags:
- 排序
- 堆排序
- 大根堆
- 小根堆

description: "heap_sort 说明文档"
summary: "" # 文章简单描述，会展示在主页
weight: # 输入1可以顶置文章，用来给文章展示排序，不填就默认按时间排序
slug: ""
draft: false # 是否为草稿
comments: true
showToc: true # 显示目录
TocOpen: true # 自动展开目录
autonumbering: true # 目录自动编号
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
searchHidden: false # 该页面可以被搜索到
showbreadcrumbs: true # 顶部显示当前路径
mermaid: true
cover:
    image: ""
    caption: ""
    alt: ""
    relative: false
---


<!-- ---
date: '2025-05-20T17:25:47+08:00'
draft: true
title: 'Heap_sort'
--- -->

## 大根堆
大根堆：父节点比左、右子节点都大。

父、子节点在数组中的对应关系，$ i=0 $代表$ root $。左子结点为$ 2i+1 $、右子节点为$ 2i+2 $。

对应关系推理过程，假设父节点对应的二叉树位置为第$ n $层、第$ m $个（都从1开始计数），父节点下标为$ p $，左、右子节点下标分别为$ l $、$ r $，下标都从0开始计数。根据等比数据列求和公式：

$ S_n = a_i \frac{1-q^n}{1-q} $

父节点、子节点的下标为：

$ \begin{aligned}
p &= 1\frac{1-2^{(n-1)}}{1-2} + m - 1 \\
  &= 2^{(n-1)} + m - 2 \\
l &= 1\frac{1-2^n}{1-2} + 2(m-1) + 1 - 1 \\
  &= 2^n + 2m - 3 \\
r &= l + 1 \\
  &= 2^n + 2m - 2
\end{aligned} $ 

由此可以推出，$ p $、$ l $、$ r $之间的对应关系为：

$ \begin{align}
l &= 2p + 1 \\
r &= 2p + 2 \\
\end{align} $

## 堆排序
首先构建大根堆，从底层开始线上构建，不断调整子树结构，相当于不断添加父节点。具体步骤如下：

1. 构建大根堆：
    1. 数组从右往左开始扫描。
    2. 检查父节点和左右节点的大小关系，保证父节点大于子节点，如果有交换，也要保证子树是正确的。
2. 此时$ nums[0] $是最大的数，把它交换到$ nums[n-1] $,然后继续开始从$ nums[0] $开始构建子树。



小根堆的情况和大根堆类似，只需要修改判断条件。

### 时间复杂度
1. 构建堆：$ O(nlog(n)) $
2. 排序：$ O(nlog(n)) $

### 空间复杂度
不考虑递归调用的情况下，空间复杂度为$ O(1) $，不需要额外的空间。

### 代码
```python
def max_heapify(self, heap, root, heap_len):
    p = root
    hl = heap_len
    while True:
        l, r = 2 * p + 1, 2 * p + 2
        # 找到子节点中较大的
        n = l
        if r < hl and heap[l] < heap[r]:
            n = r
        # 判断是否需要交换
        if n < hl and heap[p] < heap[n]:
            heap[n], heap[p] = heap[p], heap[n]
            p = n
        else:
            break

def build_heap(self, heap, heap_len):
    for i in range(heap_len-1, -1, -1):
        self.max_heapify(heap, i, heap_len)

def heap_sort(self, heap):
    heap_len = len(heap)
    self.build_heap(heap, heap_len)
    for i in range(heap_len-1, -1, -1):
        # 把最大的放到末尾，并且堆大小缩减1，从根节点更新堆
        heap[0], heap[i] = heap[i], heap[0]
        self.max_heapify(heap, 0, i)
```



