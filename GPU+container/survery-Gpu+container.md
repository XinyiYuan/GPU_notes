> 2023.8.29
>
> k8s封装GPU，Pod资源调度管理
>
> cuda 国产硬件库函数 编译 cuda工作原理 层次
>
> arm生态 -> riscv生态，自动化迁移

[TOC]

## CUDA工作原理

<img src="./pic/032zh3rohx.jpeg" alt="032zh3rohx" style="zoom:50%;" />

Nvidia GPU的软件调用栈：

ML程序 -> TensorFlow/PyTorch/paddle等框架 -> Cuda Runtime及周边生态（cudart、cublas、cudnn、cufft、cusparse等） -> Cuda driver (User Mode Driver) -> Nvidia driver (Kernel Mode Driver) -> Nvidia GPU

CUDA开发者使用的，通常是CUDA Runtime API（high-level），而CUDA Driver API是low-level的，对程序和GPU硬件有更精细的控制。Runtime API是对Driver API的封装。

CUDA Driver即为UMD（User Mode Driver，GPU用户态驱动程序），他直接和KMD（Kernel Mode Driver，GPU内核态驱动程序）打交道。两者都属于Nvidia Driver Package。英伟达软件生态封闭：无论是 nvidia.ko，还是 libcuda.so，还是 libcudart，都是被剥离了符号表的。大多数函数名是加密替换了的。其它的反调试、反逆向手段也基本不可用。（nvidia实习生实现了cuda driver逆向工程。只你想了一小部分umd和kmd之间的接口，已不维护）

## GPU共享

为了实现GPU共享，我们需要解决的问题：

- 算力隔离：限制任务占据算力（线程/SM）及显存的比例，进一步地，可以限制总线带宽

- 故障隔离：限制一个任务产生故障后的影响范围

- 并行模式：时间片模式和MPS模式

GPU池化：使用远程访问的形式使用GPU资源，任务是用本机的CPU和另一台机器的GPU，两者通过网络进行通信。

## 容器GPU虚拟化

### NVidia（均未开源）

- Nvidia GRID (vGPU)共享模块在Nvidia driver中（未开源、有加密、无法逆向）。只能用于虚拟机平台，不能用于docker容器。无法动态调整资源分配比例（只能所有用户平分）。
- Nvidia MPS，部分文档公开。在算力隔离方面表现良好（将多个进程的cuda context合并到一个context中，在context内部实现算力隔离，省去context switch的开销）。但会导致额外的故障传播（某一个server或client出错，会导致其他无关client异常退出），因此在工业界很少使用。

### kubernetes

- 阿里 cGPU，未开源无论文。支持容器级GPU虚拟化，多个容器共享GPU。但只能在阿里云使用。

  其共享模块在Nvidia driver层之上，也就是内核态。由于是在公有云使用，相对于用户态的共享会更加安全。它也是通过劫持对driver的调用完成资源隔离的，通过设置任务占用时间片长度来分配任务占用算力，但不清楚使用何种方式精准地控制上下文切换的时间。值得一提的是，由于Nvidia driver是不开源的，因此需要一些逆向工程才可以获得driver的相关method的name和ioctl参数的结构。该方案在使用上对用户可以做到无感知，当然JCT是有影响的。

- **腾讯TKE等 GaiaGPU/vCUDA (ISPA'18)，开源**。它通过劫持对Cuda driver API的调用来做到资源隔离。劫持的调用如图二所示。具体实现方式也较为直接，在调用相应API时检查：（1）对于显存，一旦该任务申请显存后占用的显存大小大于config中的设置，就报错。（2）对于计算资源，存在硬隔离和软隔离两种方式，共同点是当任务使用的GPU SM利用率超出资源上限，则暂缓下发API调用。不同点是如果有资源空闲，软隔离允许任务超过设置，动态计算资源上限。而硬隔离则不允许超出设置量。

- **KubeShare (HPDC'20)，开源**。也是通过拦截转发的方式。

  > GaiaGPU ... based on the LD_PRELOAD API interception technique, like our work.

- **Nvidia Docker**

  可以在docker内使用GPU，可以装一部分用户态的驱动程序。但只能给容器分配一整个GPU，无法share。

- ConvGPU

  在容器内share GPU memory，但不分享算力资源（线程）。

  > only supports sharing of memory resources and only virtualizes a single GPU

- rCUDA (HPCS'10)，未开源

  http://www.rcuda.net

  和Gaia一样，在Cuda driver API之上，通过劫持调用来做资源隔离。不同的是，rCuda除了资源隔离，最主要的目标是支持池化。池化简单来讲就是使用远程访问的形式使用GPU资源，任务使用本机的CPU和另一台机器的GPU，两者通过网络进行通信。也是因为这个原因，共享模块需要将CPU和GPU的调用分开。然而正常情况下混合编译的程序会插入一些没有开源的Cuda API，因此需要使用作者提供的cuda，分别编译程序的CPU和GPU部分。如果使用该产品，用户需要重新编译，对用户有一定的影响。

- Gandiva (OSDI ’18)

  https://zhuanlan.zhihu.com/p/347789271

### 其他

- SIREN (INFOCOM'19) 是基于serverless function而不是k8s。但感觉思路可以借鉴。

  一个基于无服务器架构设计的分布式机器学习框架。由两部分组成：本地客户端和无服务器云平台（AWS Lambda）。

  使用深度强化学习agent进行资源调度决策，云平台根据决策加载无状态函数。

  特点：每个epoch都会用agent进行一次决策。所以在不同epoch中可以启动不同数量、不同内存配置的函数。解决了ML工作流中不同任务异构性导致的资源不平衡问题。ML用户往往会根据整个工作流中需要最大算力的时刻进行资源分配，导致GPU利用率降低。

> A survey of GPU sharing for DL: https://zhuanlan.zhihu.com/p/285994980
>
> GPU虚拟化，算力隔离，和qGPU: https://cloud.tencent.com/developer/article/1831090

## GPU migration

