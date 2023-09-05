# Gandiva - Reading Report

> Gandiva: Introspective Cluster Scheduling for Deep Learning (OSDI ’18)
>
> paper: https://www.usenix.org/system/files/osdi18-xiao.pdf
>
> slide: https://www.usenix.org/sites/default/files/conference/protected-files/osdi18_slides_sivathanu.pdf

[TOC]

## Goal

解决在云环境中运行深度学习任务时，GPU利用率低的问题

使用领域特定知识，提出一个新的云平台资源调度框架，并在k8s实现

## Background

深度学习任务的特点和不良影响：

1. 反馈驱动的搜索Feedback-driven exploration。训练过程中，会将不同configuration的任务一起训练，比较它们的结果，并将效果特别差的任务提前终止。
2. 异构性Heterogeneous。训练任务在运行过程中，例如前向传播、反向求导、梯度更新等阶段对 GPU 资源的需求是不同的，batch之间GPU资源往往是空闲的。

现状：每个任务独占GPU资源，导致

1. 延迟高。队头任务会长时间占用整块GPU资源head-of-line blocking

2. 效率低。一旦将一个任务调度到某个GPU上之后，就不可以再进行迁移migration。但是，不同任务的sensitivity to locality是不同的。最好能进行适当的迁移来提高任务执行效率。

### Domain Knowledge

#### 位置敏感Sensitivity to locality

这一点主要考虑了 GPU 亲和性对训练任务的影响。

两块 GPU 可能分布在 (1) 不同的 CPU Socket; (2) 相同的 CPU socket, 但是不同的 PCI switch; (3) 相同的 PCI switch 这三种情况。对于相同的训练任务，不同的 GPU 亲和性对训练效率会产生较大的影响。

#### 干扰敏感Sensitivity to interference

这一点主要考虑任务之间的相互影响。作者发现，共享于相同 GPU 节点上的模型之间往往会对彼此的任务产生消极的影响，但是受到影响的程度却往往不同。（但是，对于模型受到影响的程度和模型具体有什么样的关系文中没有详细的分析。）

#### 任务间可预测性Intra-job predictability

<img src=".\pic\293a22516a8f39cf836fa5e547a4cd5.png" alt="293a22516a8f39cf836fa5e547a4cd5" style="zoom:30%;" />

从图中可以看出，显存使用量出现周期性变化，且变化周期与minibatch周期相同：在forward pass阶段，显存使用量上升；在backward pass阶段，显存使用量下降。

> epoch：使用训练集中全部数据对模型进行一次完整训练的过程
>
> batch：因为内存无法一次放下所有数据，所以将样本数据分成很多份，每次使用一份对模型权重进行一次反向传播和参数更新，每一份为一个batch
>
> iteration：使用一个batch的数据进行一次参数更新的过程
> $$
> Number \ of \ batches = \frac{Training\ Set \ Size}{Batch \ Size}
> $$
> 梯度下降的几种方式的根本区别就在于上面公式中的 Batch Size不同
>
> <img src=".\pic\e13c34aa57b2e385440e74e23bd8348.png" alt="e13c34aa57b2e385440e74e23bd8348" style="zoom:30%;" />
>
> 注：上表中Mini-Batch的Batch个数为(N/B+1)是针对未整除的情况，CNN会将剩下的样本组一个比较小的batch。整除则是(N/B)。
>
> 另外如果是RNN的话，这些剩下的样本通常就直接被扔掉了。

## Design

### Mechanisms



### Implementation

Gandiva和现有调度器的对比（示意图）：

<img src=".\pic\4119a716ffe00c3d5c270f9b46f4e32.png" alt="4119a716ffe00c3d5c270f9b46f4e32" style="zoom:50%;" />

Gandiva的architecture：

<img src=".\pic\WechatIMG481.jpg" alt="WechatIMG481" style="zoom:30%;" />

论文的改进指出在于图上的绿色方框内：左侧实现了一个资源调度器，右侧实现了一个与调度器进行交互的客户端。
