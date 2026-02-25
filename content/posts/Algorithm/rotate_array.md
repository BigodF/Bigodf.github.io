---
title: 旋转数组
date: '2026-02-10T19:38:39'
lastmod: '2026-02-25T15:14:39'
author:
- Bigodf
tags:
- 旋转数组
- leetcode
description: leetcode旋转数组
summary: ''
weight: null
slug: ''
draft: false
comments: true
showToc: true
TocOpen: true
autonumbering: true
hidemeta: false
disableShare: true
searchHidden: false
showbreadcrumbs: true
mermaid: true
cover:
  image: ''
  caption: ''
  alt: ''
  relative: false


---

url：[旋转数组](https://leetcode.cn/problems/rotate-array/solutions/551039/xuan-zhuan-shu-zu-by-leetcode-solution-nipk/)

题目描述：给定一个长为 $l$ 数组、一个整数 $k$。要求数组向右循环移动 $k$ 位（超出的位置补在最左边，类似汇编的循环位移）。

```python fold
nums = [1, 2, 3, 4, 5]
k = 2
# after rotate
nums = [4, 5, 1, 2, 3]
```

分析：
由于需要原地替换，空间复杂度为 $O(1)$ ，所以不能申请额外的空间，定义一个变量tmp，用于保存每次替换的值。

对于位置 $i$ 的元素，移动后的位置 $j=(i+k)\mod l$ ，接下来是 $j=(i+2k)\mod l$ ，一直到 $j=(i+nk)\mod l$ 。

需要注意的是这些位置之间有什么关系。有两个问题需要注意：
- 这些位置会不会重复？
- 能覆盖多少位置？
重复覆盖对应的数学关系如下，此时对应的下标都是 $c$ 。
$$
\begin{align}
i + xk = al + c \\
i+ yk = bl + c \\
\end{align} \tag{1}
$$
两式相减可得
$$
\begin{align}
(y-x)k = (b-a)l
\end{align} \tag{2}
$$
由此可见，位置序列会重复，具有周期性，周期是 $k、l$ 的最小公倍数 $lcm(k, l)$。

也就是说，每次移动的序列长度为 $lcm(k,l)/k$。需要移动的序列个数为
$$
\begin{align}
\frac{n}{\frac{lcm(k,l)}{k}}
\end{align} \tag{3}
$$

```python fold
def rotate(self, nums: List[int], k: int) -> None:
	"""
	Do not return anything, modify nums in-place instead.
	"""
	l = len(nums)
	k = k % l
	cnt = 0
	st = 0
	# 已完全替换作为结束条件
	while cnt < l:
		tmp = nums[st]
		i = (st + k) % l
		# 已重复作为结束条件
		while i != st:
			nums[i], tmp = tmp, nums[i]
			i = (i + k) % l
			cnt += 1
		nums[st] = tmp
		cnt += 1
		st += 1
	return
```


