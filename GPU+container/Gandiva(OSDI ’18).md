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

### 反馈驱动的搜索Feedback-driven exploration

> 实现高精确度的一个前提条件是模型的选择。新模型的发现，如ResNet或Inception，如今大多是一个试错的过程，尽管使之自动化的方法是一个活跃的研究领域。
>
> 除了模型结构外，还有一些参数，称为超参数，也需要作为DLT工作的一部分被指定。超参数包括模型中的层数/权重、最小批量大小、学习率等。这些参数通常由用户根据领域知识和试错来选择，有时甚至会导致早期训练失败。
>
> 因此，DLT工作的早期反馈是至关重要的，特别是在训练的初始阶段。

训练过程中，会将不同configuration的任务一起训练，比较它们的结果，并将效果特别差的任务提前终止。

传统的调度器运行一个工作子集， 完成前其他工作排队。这种模式不适合multi-jobs，因为a multi-job中的所有工作需要同时得到早期反馈。

> multi-job
>
> 超参数空间搜索：用户生成多个（一般是数百个）DLT任务或多任务，每个任务使用一组超参数或配置进行全面训练。这个过程在计算上是很昂贵的。对于超参数搜索算法（如HyperOpt和Hyperband）而言，整个作业集的早期反馈是至关重要的，需要使用这些反馈数据做出有效的训练决定。

### 异构性Heterogeneous

训练任务在运行过程中，例如前向传播、反向求导、梯度更新等阶段对GPU资源的需求是不同的，batch之间GPU资源往往是空闲的。

#### 位置敏感Sensitivity to locality

这一点主要考虑了GPU亲和性对训练任务的影响。

两块GPU可能分布在 （1）不同的 CPU Socket；（2）相同的CPU socket, 但是不同的PCI switch；（3）相同的PCI switch这三种情况。对于相同的训练任务，不同的GPU亲和性对训练效率会产生较大的影响。

实验发现：VGG16受locality影响很大，但ResNet-50基本不受locality影响。原因：VGG16比ResNet-50大，每个mini-batch中的模型同步会给底层PCIe总线带来更高的通信负载。

结论：Thus, a DLT scheduler has to take into account a job’s sensitivity to locality when allocating GPUs.

#### 干扰敏感Sensitivity to interference

这一点主要考虑任务之间由于资源争用导致彼此间相互干扰。

（但是，对于模型受到影响的程度和模型具体有什么样的关系文中没有详细的分析。）

### 现状：每个任务独占GPU资源

导致

1. 延迟高。队头任务会长时间占用整块GPU资源head-of-line blocking

2. 效率低。一旦将一个任务调度到某个GPU上之后，就不可以再进行迁移migration。但是，不同任务的sensitivity to locality是不同的。最好能进行适当的迁移来提高任务执行效率。

### Domain Knowledge

#### 任务内可预测性Intra-job predictability

利用这个特点来缓解上述问题。

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

1. Suspend-Resume and Packing 多任务共享GPU

   在过载期间，允许后来的工作与现有工作共享GPU。有以下两种方式：

   - Suspend-Resume：类似任务调度

     Suspend 的作用是将某一任务暂时挂起。分为四步（1）等待一段时间，直至该任务的显存使用达到最低；（2）将显存上的数据转移到内存中；（3）释放显存资源；（4）将任务挂起。

     Resume的作用是恢复某一任务的执行。分为三步（1）分配显存资源；（2）将内存中数据转移到显存中；（3）启动该任务。

     这个方法的好处是当多个任务需要的显存总和高于GPU的总和时，任务依旧能够执行，而不是像传统的方法中，某一任务一旦获取资源，直至任务运行结束才会释放资源。

   - Packing：类似任务并行

     只有当作业需要的资源不超过GPU的资源、且不会对彼此产生不利影响时，Packing才是有效的。（如果作业相互干扰，Packing就会比Suspend-Resume差很多。）

     > 可能采取类似于Nvidia MPS中上下文共享的方法。因为不同的进程共享了GPU上下文，因此与上一种方法对比则减少了上下文切换的开销。当然，这种方法的局限性在于（1）任务之间不能产生消极的相互影响；（2）任务需要的显存总量不能超过GPU能够提供的最大值。

2. Migration 改变分配给任务的GPU集

   何时需要进行迁移：有空闲GPU，或同一个GPU上的作业发生干扰，或作业需要locality。

   2种实现方式：

   （1）利用通用进程迁移机制，如CRIU。但CRIU本身不支持GPU设备的进程迁移，需要一些manual work；且CRIU的检查点大小是GB级别的，迁移开销非常高（~1000s）。

   （2）（本文采用）通过深度学习框架提供的保存与恢复机制，例如`torch.save()`和`torch.load()`。且作者引入了`warm up`的机制（提前预热），从而减小Migration带来的开销（~1s）。

3. Grow-Shrink 在集群未完全利用时，提高任务效率

   在集群空闲（运行的任务较少）时，分配给某一任务的GPU数量则会增多（Grow）；反之，则会减少（Shrink）。

   此机制仅适用于（自我声明）性能与GPU数量相关的应用。当多个DLT作业符合这一标准时，Gandiva会使用profiling information来估算每个作业的进度，然后相应地分配GPU。

4. Profiling 资源利用率评估

   传统的资源调度框架在衡量资源利用率时，往往从GPU memory和GPU core的利用率。

   与他们不同，本模型考虑到DLT任务的规律性（intra-job predictability），通过比对每个迭代过程所花费的时间，来判断该中资源分配是否合理。

   这种评估方法非常简单和直接，否则我们需要对DLT作业的性能、他们之间可能存在的各种干扰方式进行建模，非常困难。

### Concepts

- DLT任务（DLT job）

  封装在容器中。

  包含的参数：需要的GPU数量（2^n），任务的权重（可动态调整），标识符（是否可以Grow-Shrink）。

- 集群（cluster）

  集群由1-n个服务器组成，每个服务器有1-n个GPU。

- 服务器的高度（height of a server）

  定义为⌈M/N]， 其中M是分配的GPU数量，N是总GPU的数量。只有当服务器的高度超过1时，才会使用Suspend-Resume机制。

- 集群的高度（height of a cluster）

  集群中服务器的最大高度。当集群的高度大于1时，就会出现过载：即所有作业的请求/分配的GPU之和大于GPU的总数。

- 服务器的亲和力（the affinity of a server）

  由分配给该服务器的DLT任务决定（例如，最初服务器的亲和力为零，如果一个需要两个GPU的作业被分配到一个服务器上，那么该服务器的亲和力就会变成2）。

  调度器使用这个参数来将具有类似GPU需求的作业分配到同一台服务器上。

### Scheduler

目标：为作业提供早期反馈，提高集群效率。

公平性并**不是**目标（只关注Suspend-Resume对于作业之间的公平性的影响，集群级的公平性留给未来工作）。

为了实现这些目标，Gandiva调度器以两种模式运行（可同时处于两种模式）：响应模式Reactive Mode、内省模式Introspective Mode。

- 响应模式Reactive Mode

  对事件（如工作到达、离开、机器故障等）做出反应

  - 作业到达

    <img src=".\pic\20230906224442.png" alt="20230906224442" style="zoom:30%;" />

    findNodes是一个函数，用于返回满足作业请求的节点候选者。有一个可选参数，可用来约束亲和力。

    最初，Gandiva试图找到与新工作具有相同亲和力的节点，并在这些节点中找到具有最小负载的节点。如果存在这样的节点，并且它们的高度小于1（第5-6行），该节点将被分配。

    否则，Gandiva试图找到并分配未亲和的节点（第7-8行）。

    如果没有这样的空闲服务器，第三个选择是寻找有空闲GPU的节点，同时忽略亲和力（第9-10行）。（这可能会导致多个节点之间的碎片化分配，但Migration可以用于处理碎片化问题）。

    如果上述方法都不奏效，这意味着集群中没有可用的GPU。在这种情况下，如果存在具有相同亲和力的节点，它们将被用于暂停-恢复（第11-12行）；如果没有，作业将被排队（第13-14行）。

  - 作业离开

    传统的调度器用作业离开来触发从等待队列中挑选下一个作业

    另外，Gandiva会检查集群的高度是否可以降低（通过Migration）

    最后，作业离开可以导致migrations for improving locality

  - 机器故障

    没有展开说。we follow the conventional approach for failure handling

- 内省模式Introspective Mode

   一个持续的过程。调度器持续监控并优化作业在GPU上的位置，以提高集群的利用率和作业完成时间

  - Packing

    只在过载时候使用。但可能无效，也可能不如Suspend-Resume。

    采用了贪心算法的思想。首先，所有的任务都采用Suspend-Resume的方法运行一段时间，并进行Profiling（监测资源使用和任务进度）。调度器贪婪地挑选出GPU利用率最低的作业，并试图将其Packing到具有最低GPU利用率的GPU上。如果Packing之后的结果比Packing之前更好，那么重复上述步骤；否则，取消Packing，重新Suspend-Resume，直到找到新的可能适合Packing的任务。

    26%效率提升

  - Migration

    改善位置性：挑选那些不在同一地点的作业，并试图找到一个新的同一地点的位置。

    去碎片化：挑选非空闲、但拥有最多空闲GPU的服务器，尝试将运行在该服务器上的作业迁移到其他空闲GPU较少的服务器上。

  - Grow-Shrink

    使用条件：集群中存在空闲，且DLT作业明确指出自己可以进行Grow-Shrink

    防止震荡：在空闲一段时间后才触发Grow，在新作业可能需要GPU时立即Shrink

    只让作业增长到单台服务器中可用的最大GPU数量

  - Time-Slicing

    在每个服务器中轮回调度，以公平地分享GPU。但作业具有优先级，较高优先级的作业将不会被暂停以适应较低优先级的作业。

## Implementation

DLT作业被封装为Docker容器，其中包含我们定制的DL工具箱和Gandiva客户端的版本。这些工作被提交给Kubernetes系统。Gandiva还实现了一个定制的调度器，然后对这些作业进行调度。

### Scheduler

Gandiva和现有调度器的对比（示意图）：

<img src=".\pic\4119a716ffe00c3d5c270f9b46f4e32.png" alt="4119a716ffe00c3d5c270f9b46f4e32" style="zoom:50%;" />

Gandiva的architecture：

<img src=".\pic\WechatIMG481.jpg" alt="WechatIMG481" style="zoom:30%;" />

论文的改进为图上绿色部分。首先单独实现了一个资源调度器，而且在DLT工作容器中实现了一个与调度器进行交互的客户端。

- 调度器

  一个由Kubernetes管理的容器

  Kubernetes负责整体集群管理，Gandiva调度器从宏观的角度对用户的训练任务进行资源分配。

  Gandiva调度器使用Kubernetes API来获取集群节点和容器信息，每当提交一个新的容器时，调度器会根据调度策略将其分配给集群中的一个或多个GPU。

- 客户端

  与调度器交互，真正负责任务的开始、结束、暂停、重启等操作。

  当一个容器被安排在一个节点上时，最初只有Gandiva客户端开始执行。然后，客户端轮询Gandiva调度器，得到调度器的指令后，使用暂停/恢复和迁移命令控制DLT工作的执行。

### DL Toolkits

- PyTorch time-slicing

  暂停：Gandiva client发出SIGTSTP信号（表示需要立即暂停进程），Toolkits通过跟踪GPU内存使用周期来确定mini-batch的边界。一旦检测到周期最小值，工具包（1）将所有存储的对象从GPU复制到CPU，（2）释放GPU分配，（3）暂停进程。

  恢复：Gandiva client发出SIGCONT信号（表示恢复进程），Toolkits分配GPU内存，将存储的对象从CPU复制到GPU，并恢复进程。（可能发生地址改变，需要注意修补）

- Tensorflow migration

  在每个server上部署一个Migration Helper来协助Migration。

  迁移：dest Helper首先预热TF会话（检查GPU配置并重建元图），src Helper要求TF保存检查点，在跨服务器迁移的情况下将检查点移到目的地，最后恢复训练会话。

  > 为了加快迁移过程，我们采用Ramdisk将检查点保存在内存中。在跨服务器的情况下，修改后的TF通过网络文件系统（NFS）协议将检查点直接保存到远程Ramdisk。

  检查点：Migration Helper要求作业执行checkpoint，修改后的TF会在一个mini-batch结束时调用tf.Saver。

  > 对于数据的并行性，检查点只包括一个GPU中的模型，而不考虑训练中使用的GPU数量。
  >
  > 为了进一步加快TF的迁移，我们在检查点中不包括元图结构，因为它可以根据用户代码进行重建。
