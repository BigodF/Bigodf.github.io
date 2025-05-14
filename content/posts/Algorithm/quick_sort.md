---
title: 'Quick_sort'
date: 2025-05-14T17:21:52+08:00
lastmod: 2025-05-14T17:21:52+08:00
author: ["Bigodf"]

tags:
- 排序
- 快速排序

description: "quick_sort 说明文档"
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
date: '2025-05-14T17:21:52+08:00'
draft: true
title: 'Quick_sort'
--- -->


快速排序使用递归的方式，每次找一个基准$ p $，把待排序列分为两部分$ A $、$ B $（准确来说是三个部分$ A $、$ p $、$ B $），使得$ E(x\in A), x <= p $，$ E(x\in B), x >= p $

## 复杂度分析
### 空间复杂度
不考虑递归调用的情况下，空间复杂度为$ O(1) $。

### 时间复杂度
理想情况下，每次排序都能够把待排序列平均分为两部分。递归的次数为$ log(n) $，每层递归需要遍历一遍整个数组，所以时间复杂度为$ O(nlog(n)) $。

其它情况，会有退化。全相等、已排序的数组，递归的次数会变化$ log(n) $-> $ n $。

## 实现
有两种实现方式：

1. 单向扫描。从左端开始，把小于基准的都交换到左边。
2. 双向扫描。从左端找一个大于基准的，从右端找一个小于基准的，开始交换。

注意点：

1. 需要预留基准的位置，左端、或者右端。
2. 扫描结束，需要把基准放到**正确的位置**，必须放到正确的位置，因为之后这个位置不会再参与交换。

### 单向扫描
```python
def quicksort_inplace(arr, low, high):
    def partition(arr, low, high):
        pivot = arr[high]  # 使用最后一个元素作为 pivot
        i = low - 1  # 小于 pivot 区域的最后一个索引
        for j in range(low, high):
            if arr[j] <= pivot:
                i += 1
                arr[i], arr[j] = arr[j], arr[i]  # 交换
        arr[i + 1], arr[high] = arr[high], arr[i + 1]  # 把 pivot 放到中间
        return i + 1

    if low < high:
        pivot_index = partition(arr, low, high)
        Solution.quicksort_inplace(arr, low, pivot_index - 1)
        Solution.quicksort_inplace(arr, pivot_index + 1, high)
```

### 双向扫描
```python
def partition(l, r):
    # i = random.randint(l, r)
    p_idx = l
    p = nums[p_idx]
    nums[l], nums[p_idx] = nums[p_idx], nums[l]
    p_idx = l
    l += 1
    
    while l < r:
        while l < r and nums[l] < p:
            l += 1
        while l < r and nums[r] >= p:
            r -= 1
        
        if l < r and nums[l] >= p and nums[r] < p:
            nums[l], nums[r] = nums[r], nums[l]
            l += 1
            r -= 1
            
    if nums[l] > p:
        l -= 1
    nums[l], nums[p_idx] = nums[p_idx], nums[l]
    return l

def quick_sort(nums: List[int], l, r) -> List[int]:
    if l >= r:
        return
    p = partition(l, r)
    quick_sort(nums, l, p-1)
    quick_sort(nums, p+1, r)
```

