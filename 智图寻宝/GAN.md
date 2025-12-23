[TOC]



## 一、GAN 想解决什么问题？

目标：**学习一个数据分布，并且能从中“采样”出新的样本**。
 比如：

- 给你一堆真实人脸照片 → 学会生成“看起来像真的”新脸
- 给你一堆手写数字 → 生成新数字
- 后续也可以扩展到：风格迁移、超分辨率、图像修复等等

传统做法（显式建模）：

- 明确写出一个概率分布 $p_\theta(x)$，比如高斯混合、VAE 等
- 尝试最大化似然 $\log p_\theta(x)$

GAN 做的是 **隐式建模（implicit model）**：

- 不写出明确的密度函数，只给一个“采样器”：
  $$
  z \sim p(z) \quad(\text{高斯/均匀随机噪声}) \\
  x = G_\theta(z) \quad(\text{生成器网络})
  $$

- 希望生成的 $x$ 的分布接近真实数据分布 $p_{data}(x)$。

------

## 二、基本角色：生成器 G 和判别器 D

### 1. 生成器 G（Generator）

- 输入：随机噪声 $z \sim p(z)$（比如标准正态 $N(0,I)$）
- 输出：假的样本 $\hat{x} = G(z)$（比如一张 $64 \times 64$ 的人脸）

目的：

> 骗过判别器，让 D 以为这是“真图”。

------

### 2. 判别器 D（Discriminator）

- 输入：一张图（可能来自真实数据，也可能来自 G）
- 输出：一个标量 $D(x) \in (0,1)$
  - 表示“这张图是真实的概率”或“是真图的置信度”

目的：

> 分得清
>
> - 真图：$D(x) \to 1$
> - 假图：$D(G(z)) \to 0$

------

## 三、原始 GAN 的目标函数：一个二人零和博弈

Goodfellow 2014 的原始形式是一个 min-max 游戏：
$$
\min_G \max_D V(D, G) =
\mathbb{E}_{x \sim p_{data}(x)}[\log D(x)] + 
\mathbb{E}_{z \sim p(z)}[\log(1 - D(G(z)))]
$$
解释一下：

- 对于 **判别器 D**：

  - 真图 $x$：希望 $\log D(x)$ 大 → D(x) 靠近 1
  - 假图 $G(z)$：希望 $\log(1 - D(G(z)))$ 大 → D(G(z)) 靠近 0
     → 所以 **D 要最大化这个式子**。

- 对于 **生成器 G**：

  - 生成样本 $G(z)$ 只出现在第二项：
    $$
    \mathbb{E}_{z \sim p(z)}[\log(1 - D(G(z)))]
    $$

  - G 想让 D 搞不清楚真假，所以希望 D(G(z)) 越大越好 → 让 $\log(1 - D(G(z)))$ 变小
     → 所以 **G 要最小化整个式子**。

> 总结：
>  D 想把真/假分开得越清楚越好；
>  G 想生成的假图让 D 分不清。

------

## 四、训练过程是怎么跑的？

一个标准的训练 loop，大致是：

1. **更新 D（打假的人）** —— 固定 G，训练 D 分清真假

   - 从真实数据采样一批 $x \sim p_{data}$

   - 从噪声分布采样一批 $z \sim p(z)$，生成 $G(z)$

   - 损失（取负号写成最小化形式）：
     $$
     L_D = -\Big(
       \mathbb{E}_{x \sim p_{data}}[\log D(x)] +
       \mathbb{E}_{z \sim p(z)}[\log(1 - D(G(z)))]
     \Big)
     $$

   - 对 D 的参数做一次梯度下降。

2. **更新 G（造假的人）** —— 固定 D，训练 G 去骗 D

   - 从噪声采样 $z \sim p(z)$

   - 只考虑第二项损失（但通常会用“非饱和形式”，下面讲）：
     $$
     L_G = \mathbb{E}_{z \sim p(z)}[\log(1 - D(G(z)))]
     $$

   - 对 G 的参数做一次梯度下降。

3. 如此交替进行：更新 D 一次或几次 → 更新 G 一次 → ……

直观比喻：

- 先让“警察”变强一点（能区分真假）
- 再逼迫“造假者”进阶
- 两边轮流升级

------

## 五、为什么要用“非饱和损失”？

实践中常用的生成器损失不是原始的 $\log(1 - D(G(z)))$，而是：
$$
L_G^{\text{(non-saturating)}} = - \mathbb{E}_{z \sim p(z)}[\log D(G(z))]
$$
原因：

- 初期 D 很强时，D(G(z)) 接近 0
   → 原始损失 $\log(1 - D(G(z))) \approx 0$，梯度几乎没了，G 学不动
- 换成 $-\log D(G(z))$：
  - 当 D(G(z)) 很小 → $-\log D(G(z))$ 大 → 梯度大
  - G 会被强烈推动去增加 D(G(z))

本质上只是一个 **更好训练的替代表达形式**，目标仍然是“让 D(G(z)) 变大”。

------

## 六、理论一丢丢：和 JS 散度的关系（不用硬记）

在理想情况下，如果 D 训练到最优，可以证明：

- 对固定的 G，最优判别器：
  $$
  D^*(x) = \frac{p_{data}(x)}{p_{data}(x) + p_g(x)}
  $$
  其中 $p_g$ 是生成器引出的分布（即 $x = G(z)$ 的分布）

- 把这个 D* 代回原目标，可以得到：
  $$
  \min_G V(D^*, G) = \text{常数} - 2 \cdot \text{JS}(p_{data} \| p_g)
  $$

也就是说：

> 在理想状态下，GAN 实际上是在让 $p_g$ **逼近真实分布** $p_{data}$，
>  以 Jensen-Shannon 散度作为距离度量。

理解成一句话就行：**GAN 在优化“真实分布和生成分布的差距”**。

------

## 七、GAN 常见的问题：模式崩塌 & 不稳定

1. **模式崩塌（Mode Collapse）**
   - 模型只会生成“有限几种模式”，比如只会生成某几种风格的人脸。
   - 现象：
     - 输入不同噪声 $z$，输出非常相似的图片；
     - 模型“投机取巧”：只要能骗过 D 的那几种图，它就不愿意扩展多样性。
2. **训练不稳定**
   - D 太强 → G 梯度接近 0
   - D 太弱 → 无法提供有效的“判别信号”，G 乱学
   - 损失震荡、发散、梯度消失/爆炸都可能出现。

这也是为什么后来有了一堆“改进版”：WGAN、LSGAN、R1/R2 正则、谱归一化等。

------

## 八、GAN 的常见变种（你以后肯定会遇到）

### 1、DCGAN（Deep Convolutional GAN）

- 用全卷积网络替换原文中的 MLP
- 一些经验规则（如去掉 pool，用 stride conv，上采样用转置卷积等）
- 非常经典，做图像生成的基础结构模板。

### 2、Conditional GAN（cGAN）

> 给 GAN 加条件，比如类别标签、语义图、文本 embedding。

- G 输入：噪声 + 条件（例如类别 one-hot）
- D 输入：图像 + 条件 → 判断“这张图在这个条件下是否真实”

用途：

- 给定类别生成对应图片（Condition on class）
- 图像到图像翻译（如 pix2pix：输入轮廓输出真实图）

### 3、WGAN（Wasserstein GAN）

- 换了目标函数，用 Earth Mover 距离（EM 距离）
- 理论和实践上都更稳定
- 判别器改名叫“critic”（不输出概率而是打分）
- 要求 Lipschitz 条件：最初用 weight clipping，后面改成 gradient penalty（WGAN-GP）

这是 GAN 发展中非常重要的一支。

### 4、还有很多…

- LSGAN：把交叉熵换成 L2 回归形式，缓解梯度饱和
- StyleGAN：风格控制、人脸生成的 S 级选手
- BigGAN：大模型 + 大 batch，用在 ImageNet 生成

不用全记，知道它们都是围绕“稳定性、质量、多样性”做文章就行。

------

## 九、简单 PyTorch 训练流程概念（伪代码）

让你对训练 GAN 的节奏有个感觉（只看逻辑即可）：

```
for real_imgs in dataloader:
    # ========= 1. 更新 D =========
    # 真图
    real_imgs = real_imgs.to(device)
    batch_size = real_imgs.size(0)

    # 假图
    z = torch.randn(batch_size, z_dim, 1, 1, device=device)
    fake_imgs = G(z).detach()  # .detach() 避免更新 G

    # D 对真图的预测
    d_real = D(real_imgs)
    # D 对假图的预测
    d_fake = D(fake_imgs)

    # 真图标签为 1，假图标签为 0
    loss_D = -(torch.log(d_real + 1e-8).mean() +
               torch.log(1 - d_fake + 1e-8).mean())

    optimizer_D.zero_grad()
    loss_D.backward()
    optimizer_D.step()

    # ========= 2. 更新 G =========
    z = torch.randn(batch_size, z_dim, 1, 1, device=device)
    fake_imgs = G(z)
    d_fake = D(fake_imgs)

    # 非饱和损失：希望 D(fake) 越大越好
    loss_G = -torch.log(d_fake + 1e-8).mean()

    optimizer_G.zero_grad()
    loss_G.backward()
    optimizer_G.step()
```

整体节奏：

- 每个 iteration：
  - 先训练 D，一般可以训练 1–5 步
  - 再训练 G 一步

------

## 十、小总结（方便记忆）

你可以这样用几句短话概括 GAN：

1. **结构**：

   - G：把随机噪声映射成图像
   - D：判别一张图是真还是假
   - 二人对抗训练：$\min_G \max_D V(D,G)$

2. **目标**：

   > 让生成分布 $p_g$ 逼近真实分布 $p_{data}$

3. **直观理解**：

   - D 像一个“鉴定专家”
   - G 像一个“造假专家”
   - 两者互相博弈，最后得到一个“能造出非常像真的东西”的生成器