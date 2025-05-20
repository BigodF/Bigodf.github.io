---
title: 'Merge_sort'
date: 2025-05-20T17:25:37+08:00
lastmod: 2025-05-20T17:25:37+08:00
author: ["Bigodf"]

tags:
- 排序
- 归并排序

description: "merge_sort 说明文档"
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
date: '2025-05-20T17:25:37+08:00'
draft: true
title: 'Merge_sort'
--- -->
采用递归的方式，把两个有序的序列合并成一个序列，重复这个过程。

## 复杂度分析
### 空间复杂度
不考虑递归调用的情况下，空间复杂度为$ O(n) $，需要保存一个buffer数组，用于缓存merge数组。

### 时间复杂度
递归的次数为$ log(n) $，每层递归需要遍历一遍整个数组，所以时间复杂度为$ O(nlog(n)) $。

没有特殊情况。

## 实现
```python
def merge_sort(nums, l, r):
    buffer = [0] * len(nums)
    def merge(nums, l, r):
        if l == r:
            return
        m = (l + r) >> 1
        merge(nums, l, m)
        merge(nums, m+1, r)
        idx, i, j = l, l, m+1
        while idx <= r:
            if (j > r) or (i <= m and nums[i] <= nums[j]):
                buffer[idx] = nums[i]
                i += 1
            else:
                buffer[idx] = nums[j]
                j += 1
            idx += 1
        nums[l: r+1] = buffer[l: r+1]
        return
    merge(nums, l, r)
    return nums
```

