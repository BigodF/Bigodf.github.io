---
title: RoPE的外推
date: 2026-01-27T16:44:30+0800
lastmod: 2026-02-02T15:09:45+0800
author:
- Bigodf
tags:
- LLM
- 旋转位置编码
- RoPE
- RoPE外推
description: RoPE的外推
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

RoPE在计算上可以外推到无限长度，但是由于外推的位置，训练时没有见过，效果较差，这种方式也叫**直接外推**。此外还有线性插值、NTK-Aware Scaled RoPE、ReRoPE等外推方式。

为了简化表达，定义一下变量：
- $N$ 代表模型的训练最大长度。
- $L$ 代表要外推的长度。
- $k=\frac N L$ 。

注：这里所指的放缩是针对每组配对的旋转角来说的，即 $\theta_i$ 。

### 线性插值
和图片scale的方式类似，对位置 $n$ 进行放缩，即
$$
score(q_m, k_n) = (R(\frac m k) \cdot q_m)^T \cdot (R(\frac n k) \cdot k_n) \tag{1.1}
$$
也相当于对 $\theta$ 进行了放缩。
$$
\theta _i = \frac {10000^{-2i/d}} {k} \tag{1.2}
$$
有论文通过实现证实，在没有额外训练的情况下， 线性内插的效果比直接外推差；如果额外训练 1000 步，线性内插的效果和原本效果是接近的。

### NTK-aware Scaled RoPE
由于旋转操作是两两分组的，每组对应的 $\theta_i = 10000^{-\frac {2i}{d}}$ ，也就是说随着 $i$ 的增大，三角函数对应的周期增大、频率减小。在训练过程中，高频的分组会被覆盖多次，训练是充分的，低频分组覆盖较低，训练是不充分的。

所以从直觉上来说，充分训练的高频可以使用外推的方式，缺少训练的低频分组使用线性插值。NRK 给出的方式如下：
$$
\begin{align}
\theta _i &= (10000k)^{-2i/d} \\
          &= (10000)^{-2i/d} \cdot (k)^{-2i/d}

\end{align} \tag{2.1}
$$
当 $2i=0$ 时，$(k)^{-2i/d}=1$ ，相当于直接外推；当 $2i=d$ 时，$(k)^{-2i/d}=(k)^{-1}$ ，相当于线性插值。$(k)^{-2i/d}$ 是一个光滑的指数函数，所以在 $2i |_0^d$ 的过程中，相当于对每个分组做了直接外推 -> 线性插值的平滑。

这里有一个小问题，我们取了 $2i=d$ ，实际上 $2i$ 只能取到 $d-2$。所以我们只需要在最后一组，实现对 $\theta_i$ 的 $k$ 倍放缩即可。推理过程如下：
$$
\begin{align}
\theta _i &= (10000\lambda)^{-2i/d} =  (10000)^{-2i/d} \cdot k^{-1} \\
\end{align} \tag{2.2}
$$
带入 $2i=d-2$ ，解得 $\lambda=k^{d/(d-2)}$，故外推后
$$
\begin{align}
\theta _i &= (10000k^{d/(d-2)})^{-2i/d}

\end{align} \tag{2.3}
$$
从实验效果看，NTK的外推效果和原始效果相差5%左右，已经很接近了。

#### 另一种解释：$\beta$ 进制
神经网络模型擅长处理的是“**不大不小**”的输入数据，太大、太小都会影响效果。如果要输入一个大整数特征，有一种方式：
* 正则。例如处理成0均值，单位方差。
- 离散化。分段映射。
- 进制表示。例如分成四个特征：个、十、百、千。这样的好处是每位都是0-9，适合神经网络处理。并且鲁棒性好。

对于训练设置了4个特征位置，但是推理需要外推 $k$ 倍，应该怎么做呢？

一种方式是训练是就增加到5维特征，但是这一种没有经过训练，效果肯定不好；另一种方式是进制转化，把10进制转换为另一种进制，这种进制可以外推后的范围。神经网络主要学习是偏序关系（大小），所以第二种方式不经过训练，也能保持较好的效果。

转换的目标进制选择过程如下，其中 $\alpha、\beta、n、k$ 分别表示旧进制、新进制、特征维度、放大倍数。
$$
\begin{align}
k a\alpha^n &= \beta^n \\
\beta &= \alpha k^{1/n}

\end{align} \tag{2.4}
$$
如何进制转换应用到RoPE的外推上呢？

对于 $n$ 的 $\beta$ 进制表达，$m$ 位取值为

$$
\begin{align}
\left \lfloor \frac{n}{\beta^{m-1}} \right\rfloor \mod \beta
\end{align} \tag{2.5}
$$
观察位置 $n$ 的旋转RoPE，可以看为Sinusoidal位置编码
$$
\begin{align}
\left(
cos(\frac{n}{\beta^0}),sin(\frac{n}{\beta^0}),cos(\frac{n}{\beta^1}),sin(\frac{n}{\beta^1}), \cdots , cos(\frac{n}{\beta^{d/2-1}}),sin(\frac{n}{\beta^{d/2-1}})
\right)
\end{align} \tag{2.6}
$$
其中 $\beta=10000^{2/d}$ 。这种形式和进制表示很相似，可以看成是 $n$ 的 $\beta$ 进制 $d/2-1$ 位表示。至于模运算，它的最重要特性是周期性，cos、sin也刚好是周期函数。

要把表示范围扩大 $k$ 倍，需要满足：
$$
\begin{align}
\frac{n}{\beta^{d/2-1}} = \frac{nk}{(\lambda \beta)^{d/2-1}}
\end{align} \tag{2.7}
$$

可以解得 $\lambda=k^{2/(d-2)}$ 。进一步代入 $\beta$ 可得
$$
\begin{align}
\theta_i &= (\lambda \beta)^{(-i)} \\
         &= k^{-2i/(d-2)} 10000^{-2i/d} \\
         &= (10000 \cdot k^{d/(d-2)})^{-2i/d} \\
\end{align} \tag{2.8}
$$
这和NTK的形式是一致的。

#### ReRoPE
要想RoPE的外推效果好，需要满足高频直接外推、低频线性插值，NTK-aware的实现方式还能不能再优化呢？RePoPE给出了另一种可能的方式。

$$
D = \begin{pmatrix}
0 \\
1 &0 \\
2 &1 &0 \\
3 &2 &1 &0 \\
\ddots &3 &2 &1 &0 \\
\ddots &\ddots &3 &2 &1 &0 \\
L-2 &\ddots &\ddots &3 &2 &1 &0 \\
L-1 &L-2 &\ddots &\ddots &3 &2 &1 &0 \\
\end{pmatrix} \tag{3.1}
$$
以上是文本序列中，两两位置的相对距离，和直接外推一致。

线性插值是缩小相对位置之间的距离，位置信息更加密集，在没有微调的的情况下，模型还没有适应，效果不好。
$$
D = \begin{pmatrix}
0 \\
1/k &0 \\
2/k &1/k &0 \\
3/k &2/k &1/k &0 \\
\ddots &3/k &2/k &1/k &0 \\
\ddots &\ddots &3/k &2/k &1/k &0 \\
(L-2)/k &\ddots &\ddots &3/k &2/k &1/k &0 \\
(L-1)/k &(L-2)/k &\ddots &\ddots &3/k &2/k &1/k &0 \\
\end{pmatrix} \tag{3.2}
$$
语言模型，明显更依赖相邻位置的token，如何更好保持这种局域性？直接外推保持了局域性，但是引入了超出训练的距离。线性插值虽然没有超出超出训练的距离，但是扰乱的局域性，所以在没有进一步训练时，效果比直接外推要差。

可以假设有一个分段函数，在某个距离阈值内直接外推，超过这个阈值使用线性插值（Leaky ReRoPE）。
$$
D = \begin{pmatrix}
0 \\
1 &0 \\
2 &1 &0 \\
3 &2 &1 &0 \\
\ddots &3 &2 &1 &0 \\
w &\ddots &3 &2 &1 &0 \\
w+1/k & w &\ddots &3 &2 &1 &0 \\
w+2/k &w+1/k &w &\ddots &3 &2 &1 &0 \\
\ddots &\ddots &\ddots &\ddots &\ddots &\ddots &3 &2 &1 &0 \\
w+(L-1-w)/k &\ddots &\ddots &\ddots &\ddots &\ddots &\ddots &3 &2 &1 &0 \\
\end{pmatrix} \tag{3.3}
$$
当 $k=\infty$ ，可以简化为以下形式（ReRoPE）
$$
D = \begin{pmatrix}
0 \\
1 &0 \\
2 &1 &0 \\
3 &2 &1 &0 \\
\ddots &3 &2 &1 &0 \\
w-1 &\ddots &3 &2 &1 &0 \\
w & w-1 &\ddots &3 &2 &1 &0 \\
w &w &w-1 &\ddots &3 &2 &1 &0 \\
\ddots &\ddots &\ddots &\ddots &\ddots &\ddots &3 &2 &1 &0 \\
w &w &w &\ddots &w-1 &\ddots &\ddots &3 &2 &1 &0 \\
\end{pmatrix} \tag{3.4}
$$
这样就得到了ReRoPE，此时 $k=\infty$ ，此时相当于无限外推了。

在计算时，由于使用了分段函数，不可避免的需要计算两次 $k、v$ 的旋转操作。
$$
\begin{cases}
a_{i,j}^1 &= (R^i q_i)^T(R^j k_j) =q_i^T R^{j-i} k_j &\text{(RoPE计算)} \\
a_{i,j}^2 &= (R^{\frac{i-w}{k}+w} q_i)^T(R^{\frac{j}{k}} k_j) =q_i^T R^{\frac{j-i+w}{k} - w} k_j &\text{(Leaky ReRoPE额外计算)} \\
a_{i,j}^2 &= (R^w q_i)^T k_j = q_i^T R^{-w} k_j &\text{(ReRoPE额外计算)} \\
\end{cases} \tag{3.5}
$$
从实验来看 ReRoPE 和 leaky ReRoPE相比，差异不大，精调的 leaky 版本可以超过 ReRoPE ，但是提升很小。实验也显示出 ReRoPE 关于 w 还是很鲁棒的，最优值大致是训练长度的1/4∼1/2左右。

## 参考
1. [旋转式位置编码 (RoPE) 知识总结](https://zhuanlan.zhihu.com/p/662790439)
2. [Transformer升级之路：10、RoPE是一种β进制编码](https://spaces.ac.cn/archives/9675)
3. [Transformer升级之路：11、将β进制位置进行到底](https://spaces.ac.cn/archives/9706) 
4. [Transformer升级之路：12、无限外推的ReRoPE？](https://spaces.ac.cn/archives/9708)
5. [Transformer升级之路：18、RoPE的底数选择原则](https://spaces.ac.cn/archives/10122)
