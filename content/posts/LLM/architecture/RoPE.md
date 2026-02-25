---
title: 旋转位置编码RoPE
date: '2026-01-13T20:01:41'
lastmod: '2026-01-30T14:46:11'
author:
- Bigodf
tags:
- LLM
- 旋转位置编码
- RoPE
description: 旋转位置编码RoPE
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

Transformer中的位置编码主要有两种：
1. 绝对位置编码。两两位置间的embedding的内积和该两位置之间的距离没有关系，例如Bert中的位置编码。
2. 相对位置编码。两两位置间的embedding的内积和该两位置之间的距离有关系，例如本文要介绍的RoPE。
Transformer中的embedding主要是做内积运算，两个位置embedding的内积运算能否包含两者之间的相对距离关系，是这两种编码的主要区别。

自然语言中，相邻位置的token之间的相关性很高，对应attention score应该也较高，这是朴素的直觉。与之对应的解决方案，是如何把token之间的相对位置关系也编码到位置embedding中，让其在计算内积的时候能够捕获相对位置信息，这是相对位置编码想要解决的问题。
## Bert中的位置编码

Bert中的绝对位置编码，是直接加在输入$token\ embedding$上：
$$f_{q, k, v}(x_i, i)=W_{q, k, v}(x_i+p_i) \tag {1.1} $$
如上式所示，$x_i$是$token\ embedding$，$p_i$ 是位置$embedding$，采用正余弦交替的形式组成：
$$
\begin{cases}
	P_{i, 2t} &= sin(\frac{i}{10000^{\frac{2t}{d}}}) \\
	P_{i, 2t+1} &= cos(\frac{i}{10000^{\frac{2t}{d}}}) \\
\end{cases} \tag{1.2}
$$
其中$P_{i, 2t}$ 代表位置$i$的$embedding$中$idx=2t$的取值，也就是小标为偶数。$P_{i, 2t+1}$ 代表位置$i$的$embedding$中$idx=2t+1$的取值，也就是下标为奇数。$d$代表位置$embedding$的维度。

生成代码如下：
```python fold
# position 就对应 token 序列中的位置索引 i
# hidden_dim 就对应词嵌入维度大小 d
# seq_len 表示 token 序列长度
def get_position_angle_vec(position):
    return [position / np.power(10000, 2 * (hid_j // 2) / hidden_dim) for hid_j in range(hidden_dim)]

# position_angle_vecs.shape = [seq_len, hidden_dim]
position_angle_vecs = np.array([get_position_angle_vec(pos_i) for pos_i in range(seq_len)])

# 分别计算奇偶索引位置对应的 sin 和 cos 值
position_angle_vecs[:, 0::2] = np.sin(position_angle_vecs[:, 0::2])  # dim 2t
position_angle_vecs[:, 1::2] = np.cos(position_angle_vecs[:, 1::2])  # dim 2t+1

# positional_embeddings.shape = [1, seq_len, hidden_dim]
positional_embeddings = torch.FloatTensor(position_angle_vecs).unsqueeze(0)
```

## RoPE
RoPE的目标是构造一个位置编码使得下式成立：
$$
< f_q(x_m, m), f_k (x_n, n) > = g(x_m, x_n, n-m)  \tag{2.1}
$$
其中$x_m、x_n$ 分别是位置$m、n$的$token\ embedding$，$<>$是内积运算。该式的目的是构造一个位置编码方式，使得$q、k$的内积运算能够包含相对位置距离$n-m$的信息。

有两种方式可以解释这种关系：
- 旋转操作。
- 复数运算。

旋转操作比较简单，先来说着这个。
### 旋转操作
把位置编码定义为一种旋转操作，例如对位置$m$的编码为：
$$\begin{align}
p_m &= R(m) \\
\end{align} \tag {2.1.1}
$$
那么$attention\ score$，即$<f_q(x_m, m), f_k (x_n, n) >$计算如下：
$$\begin{align}
< f_q(x_m, m), f_k (x_n, n) > &= (R(m) W_q x_m)^T (R(n) W_k x_n) \\
                             &= (W_q x_m)^T R(m)^T R(n) W_k x_n \\
                             &= (W_q x_m)^T R(-m) R(n) W_k x_n  \\
                             &= (W_q x_m)^T R(n-m) W_k x_n \\
                             &= g(x_m, x_n, n-m)
\end{align} \tag {2.1.2}
$$
 上式的运算用到了旋转矩阵的性质。假设$R(a)$表示角度为$a$的旋转矩阵，那么$R$具有如下性质：  
$$ \begin{align}
R(a)^T &= R(-a)  \\
R(a) R(b) &= R(a+b) \\
\end{align} \tag {2.1.3}
$$
以上可以很容易的证明，旋转操作可以满足式$2.1$的成立。
### 复数运算

假设向量的维度$d=2$，需要证明的函数形式如下：
$$
\begin{align}
f_q(x_m, m) &= (W_q x_m)e^{im\theta} \\
f_k(x_n, n) &= (w_k x_n)e^{in\theta} \\
g(x_m, x_n, m-n) &= Re[(W_q x_m)(W_k x_n)^*e^{i(m-n)\theta}]
\end{align} \tag {2.2.1}
$$
$(W_k x_n)^*$表示复数$W_k x_n$的共轭，共轭定义$(a + ib)^* = a - ib$。

首先引用复数的欧拉公式：
$$
e^{ix} = cos(x) + i \cdot sin(x) \tag {2.2.2}
$$
带入$f_q$函数可得
$$
\begin{align}
f_q(x_m, m) &= (W_q x_m)e^{i m \theta} \\
			&= \begin{pmatrix}
				w_q^{0,0} & w_q^{0,1} \\
				w_q^{1,0} & w_q^{1,1} \\
				\end{pmatrix} 
				\times 
				\begin{pmatrix}
				x_m^0 \\
				x_m^1 \\
				\end{pmatrix} 
				\times
				e^{im\theta} \\
			&= \begin{pmatrix}
				q_m^0 \\
				q_m^1 \\
				\end{pmatrix} 
				\times
				e^{im\theta} \\
\end{align}  \tag{2.2.3}
$$
这里把$q_m$也当成一个复数，记为
$$
\begin{align}
q_m &= q_m^0 + i \times q_m^1 \\
    &= \begin{pmatrix}
    q_m^0 \\
    q_m^1
    \end{pmatrix}
\end{align} \tag{2.2.4}
$$

则
$$
\begin{align}
f_q(x_m, m) &= \begin{pmatrix}
				q_m^0 \\
				q_m^1 \\
				\end{pmatrix} 
				\times
				e^{im\theta} \\
			&= q_m \times e^{im\theta} \\
			&= (q_m^0 + i \times q_m^1) \times (cos(m\theta) + i \times sin(m\theta)) \\
			&= q_m^0 cos(m\theta) - q_m^1 sin(m\theta) + i \times (q_m^0 sin(m\theta) + q_m^1 cos(m\theta)) \\
			&= \begin{pmatrix}
				cos(m\theta) & -sin(m\theta) \\
				sin(m\theta) & cos(m\theta) \\
				\end{pmatrix} 
				\times 
				\begin{pmatrix}
				p_m^0 \\
				p_m^1 \\
				\end{pmatrix} 
\end{align}  \tag{2.2.5}
$$
同理
$$
\begin{align}
f_k(x_n, n)= \begin{pmatrix}
				cos(n\theta) & -sin(n\theta) \\
				sin(n\theta) & cos(n\theta) \\
				\end{pmatrix} 
				\times 
				\begin{pmatrix}
				k_n^0 \\
				k_n^1 \\
				\end{pmatrix} 
\end{align}  \tag{2.2.6}
$$
由此可以看出$f$函数是对$q、v$向量的旋转操作。
$$
\begin{align}
< f_q(x_m, m), f_k(x_n, n) > &= \left( 
					\begin{pmatrix}
					cos(m\theta) & -sin(m\theta) \\
					sin(m\theta) & cos(m\theta) \\
					\end{pmatrix} 
					\times 
					\begin{pmatrix}
					p_m^0 \\
					p_m^1 \\
					\end{pmatrix}
				\right)^T
				\\
				& \quad \times 
				\left(
					\begin{pmatrix}
					cos(n\theta) & -sin(n\theta) \\
					sin(n\theta) & cos(n\theta) \\
					\end{pmatrix} 
					\times 
					\begin{pmatrix}
					k_n^0 \\
					k_n^1 \\
					\end{pmatrix} 
				\right) \\
				&= \begin{pmatrix}
					p_m^0 \\
					p_m^1 \\
					\end{pmatrix}^T
					\times
					\begin{pmatrix}
					cos(m\theta) & sin(m\theta) \\
					-sin(m\theta) & cos(m\theta) \\
					\end{pmatrix} \\
				& \quad \times
					\begin{pmatrix}
					cos(n\theta) & -sin(n\theta) \\
					sin(n\theta) & cos(n\theta) \\
					\end{pmatrix} 
					\times 
					\begin{pmatrix}
					k_n^0 \\
					k_n^1 \\
					\end{pmatrix} \\
			    &= \begin{pmatrix}
					p_m^0 \\
					p_m^1 \\
					\end{pmatrix}^T
					\times
					\begin{pmatrix}
					cos((m-n)\theta) & sin((m-n)\theta) \\
					-sin((m-n)\theta) & cos((m-n)\theta) \\
					\end{pmatrix} 
					\times 
					\begin{pmatrix}
					k_n^0 \\
					k_n^1 \\
					\end{pmatrix} \\
				&= \begin{pmatrix}
					p_m^0 cos((m-n)\theta) - p_m^1 sin((m-n)\theta) \\
					p_m^0 sin((m-n)\theta) + p_m^1 sin((m-n)\theta) 
					\end{pmatrix} ^ T
					\times 
					\begin{pmatrix}
					k_n^0 \\
					k_n^1 \\
					\end{pmatrix} \\
				&= (p_m^0 k_n^0 + p_m^1 k_n^1) cos((m-n)\theta) + (p_m^0 k_n^1 - p_m^1 k_n^0) sin((m-n)\theta)
\end{align} \tag {2.2.7}
$$
对于$g$函数

$$
\begin{align}
g(x_m, x_n, m-n) &= Re[(W_q x_m)(W_k x_n)^*e^{i(m-n)\theta}] \\
			&= Re[(q_m^0 + i \times q_m^1)(k_n^0 - i \times k_n^1)^*e^{i(m-n)\theta}] \\
			&= Re[((q_m^0 k_n^0 + q_m^1 k_n^1) - i \times(q_m^0 k_n^1 - q_m^1 k_n^0)^*e^{i(m-n)\theta}] \\
			&= Re[((q_m^0 k_n^0 + q_m^1 k_n^1) \\
				& \quad - i \times(q_m^0 k_n^1 - q_m^1 k_n^0)^*(cos((m-n)\theta) \\
				& \quad + i\times sin((m-n)\theta))] \\
			&= (q_m^0 k_n^0 + q_m^1 k_n^1) \times cos((m-n)\theta) \\
				& \quad + (q_m^0 k_n^1 - q_m^1 k_n^0) \times sin((m-n)\theta)) \\
			&= < f_q(x_m, m), f_k (x_n, n) >
\end{align}  \tag{2.2.8}
$$
证毕。

### 实践
以上都是针对$d=2$的情况，对于多维情况，可以两两元素分组进行操作，其中每一组的 $\theta$ 选取不同，目的是不同频率的旋转组合起来。
$$
\theta_j = 10000^{-2j/d} \tag{2.3.1}
$$
具体代码如下：
```python fold

def precompute_freqs_cis(dim: int, seq_len: int, theta: float = 10000.0):
    # 计算词向量元素两两分组之后，每组元素对应的旋转角度\theta_j
    freqs = 1.0 / (theta ** (torch.arange(0, dim, 2)[: (dim // 2)].float() / dim))
    # 生成 token 序列索引 t = [0, 1,..., seq_len-1]
    t = torch.arange(seq_len, device=freqs.device)
    # freqs.shape = [seq_len, dim // 2] 
    freqs = torch.outer(t, freqs).float()  # 计算m * \theta

    # 计算结果是个复数向量
    # 假设 freqs = [x, y]
    # 则 freqs_cis = [cos(x) + sin(x)i, cos(y) + sin(y)i]
    freqs_cis = torch.polar(torch.ones_like(freqs), freqs) 
    return freqs_cis

# 旋转位置编码计算
def apply_rotary_emb(
    xq: torch.Tensor,
    xk: torch.Tensor,
    freqs_cis: torch.Tensor,
) -> Tuple[torch.Tensor, torch.Tensor]:
    # xq.shape = [batch_size, seq_len, dim]
    # xq_.shape = [batch_size, seq_len, dim // 2, 2]
    xq_ = xq.float().reshape(*xq.shape[:-1], -1, 2)
    xk_ = xk.float().reshape(*xk.shape[:-1], -1, 2)
    
    # 转为复数域
    xq_ = torch.view_as_complex(xq_)
    xk_ = torch.view_as_complex(xk_)
    
    # 应用旋转操作，然后将结果转回实数域
    # xq_out.shape = [batch_size, seq_len, dim]
    xq_out = torch.view_as_real(xq_ * freqs_cis).flatten(2)
    xk_out = torch.view_as_real(xk_ * freqs_cis).flatten(2)
    return xq_out.type_as(xq), xk_out.type_as(xk)
```

## 参考
1. [一文看懂 LLaMA 中的旋转式位置编码（Rotary Position Embedding）](https://zhuanlan.zhihu.com/p/642884818)
2. [旋转式位置编码 (RoPE) 知识总结](https://zhuanlan.zhihu.com/p/662790439)
3. [让研究人员绞尽脑汁的Transformer位置编码](https://spaces.ac.cn/archives/8130)
4. [Transformer升级之路：2、博采众长的旋转式位置编码](https://spaces.ac.cn/archives/8265)]
