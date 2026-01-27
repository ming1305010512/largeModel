![image-20260124211718487](C:\Users\16532\AppData\Roaming\Typora\typora-user-images\image-20260124211718487.png)

先给定几个公式：

𝔼[𝑋] = 𝔼[𝔼[𝑋|𝑌 ]] （全期望公式）

*E*[*XY*]=*E*[*XE*[*Y*∣*Z*]]  （条件期望的数学恒等式）

证明：

![image-20260124211959152](C:\Users\16532\AppData\Roaming\Typora\typora-user-images\image-20260124211959152.png)



假设你有两个随机变量：

- $X$
- $Y$

并且满足：
$$
\mathbb E[X \mid s_t,a_t] = \mathbb E[Y \mid s_t,a_t]
$$
定义它们的差：
$$
\varepsilon := X - Y
$$
那么：
$$
\mathbb E[\varepsilon \mid s_t,a_t] = 0
$$
也就是说：

> $\varepsilon$ 是一个 **条件零均值噪声**

###### 把它放进策略梯度里（关键一步）

考虑两种梯度估计：
$$
g_X = \mathbb E[
\nabla_\theta \log \pi(a_t\mid s_t)\; X
]
$$
它们的差是：
$$
g_X - g_Y
=
\mathbb E[
\nabla_\theta \log \pi(a_t\mid s_t)\; (X-Y)
]
=
\mathbb E[
\nabla_\theta \log \pi(a_t\mid s_t)\; \varepsilon
]
$$
现在**分条件期望**：
$$
=
\mathbb E_{s_t,a_t}
\Big[
\nabla_\theta \log \pi(a_t\mid s_t)
\;
\mathbb E[\varepsilon \mid s_t,a_t]
\Big]
$$
但我们刚刚已经知道：
$$
\mathbb E[\varepsilon \mid s_t,a_t] = 0
$$
于是：
$$
g_X - g_Y = 0
$$