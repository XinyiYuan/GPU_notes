# Nvidia Docker

NVIDIA Docker将GPU Device和CUDA Driver挂载到容器中

`nvidia-docker2.0` 是一个简单的包，它主要通过修改docker的配置文件 `/etc/docker/daemon.json` 来让docker使用NVIDIA Container runtime。

驱动是在主机上安装的，容器内的是CUDA runtime层

> Docker提供了应用运行需要的类库，就深度学习来说，调用显卡也分为两层：上层是显卡厂商封装好的面向应用各种API接口，下层是面向硬件的显卡驱动。
>
> API接口通常在Userland，在容器中；显卡驱动则是在Kernel，和Host共用。因此，需要在容器中安装显卡驱动以及相关的SDK，而且为了能正确调用内核中的相关驱动模块，最好和Host的驱动版本一致。

但只能给容器分配一整个GPU，无法share。