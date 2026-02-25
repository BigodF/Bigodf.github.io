---
title: 最大公约数
date: '2026-02-11T15:08:50'
lastmod: '2026-02-11T16:14:41'
author:
- Bigodf
tags:
- 辗转相除法
- 最大公约数
- gcd
description: 辗转相除法求最大公约数
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

求最大公约数最常用的方式是辗转相除法，辗转相除法怎么证明呢？

首先回顾一下算法过程，$a_0、a_1$ 分别为较大值、较小值，gcd过程如下：
$$
\begin{align}
\operatorname{divmod}(a_0, a_1) &= (b_0, a_2) \\
\operatorname{divmod}(a_1, a_2) &= (b_1, a_3) \\
&\vdots \\
\operatorname{divmod}(a_{n-3}, a_{n-2}) &= (b_{n-3}, a_{n-1}) \\
\operatorname{divmod}(a_{n-2}, a_{n-1}) &= (b_{n-2}, a_n) \\
\end{align} \tag{1}
$$
证明需要考虑两点：
- $gcd(a_{n-1}, a_n) = gcd(a_{n-2}, a_{n-1})$
- $a_n=0$ 

我们可以假设 $gcd(a_0, a_1)=g$ 。所以
$$
\begin{align}
a_0 &= c_0 g \\
a_1 &= c_1 g \\
\end{align} \tag{2}
$$
其中 $gcd(c_0, c_1)=1$，带入第一个除式
$$
\begin{align}
c_0 g &= b_0 c_1 g + a_2 \\
a_2 &= (c_0 - b_0 c_1) g \\
\text{令}c_2 &= c_0 - b_0 c_1 \\
a_2 &= c_2 g \\
\end{align} \tag{3}
$$
此时也有 $gcd(c_1, c_2)=1$ 。可以用如下反证法证明
$$
\begin{align}
c_0 &= b_0 c_1 + c_2 \\
    &= b_0 x \times gcd(c_1, c_2) + y \times gcd(c_1, c_2) \\
    &= (b_0 x + y)\times gcd(c_1, c_2) \\
\end{align} \tag{4}
$$
又因为 $gcd(c_0, c_1)=1$ ，所以 $gcd(c_1, c_2)=1$ 。

带入第二个除式
$$
\begin{align}
c_1 g &= b_1 (c_2 g)  + a_3 \\
a_3 &= (c_1 - b_1 c_2) g \\
\text{令}c_3 &= c_1 - b_1 c_2 \\
a_3 &= c_3 g \\
\end{align} \tag{4}
$$
可见 $g$ 也是 $a_3$ 的因数，也是数列 $a_n$ 的因数，第一点得证。

由于 $b_n>0$ ，所以 $c_n$ 是严格递减的，每次最少递减1，最后 $c_n$ 一定可以到0，此时 $a_n=0$ 第二点得证。

代码如下：
```python fold
def gcd(a, b):
	if b > 0:
		return gcd(b, a%b)
	return a
```


