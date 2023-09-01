# CUDA和k8s基本原理

## CUDA原理

<img src="/Users/smx1228/Desktop/GPU_notes/GPU+container/pic/032zh3rohx.jpeg" alt="032zh3rohx" style="zoom:50%;" />

Nvidia GPU的软件调用栈：

ML程序 -> TensorFlow/PyTorch/paddle等框架 -> Cuda Runtime及周边生态（cudart、cublas、cudnn、cufft、cusparse等） -> Cuda driver (User Mode Driver) -> Nvidia driver (Kernel Mode Driver) -> Nvidia GPU

CUDA开发者使用的，通常是CUDA Runtime API（high-level），而CUDA Driver API是low-level的，对程序和GPU硬件有更精细的控制。Runtime API是对Driver API的封装。

CUDA Driver即为UMD（User Mode Driver，GPU用户态驱动程序），他直接和KMD（Kernel Mode Driver，GPU内核态驱动程序）打交道。两者都属于Nvidia Driver Package。英伟达软件生态封闭：无论是 nvidia.ko，还是 libcuda.so，还是 libcudart，都是被剥离了符号表的。大多数函数名是加密替换了的。其它的反调试、反逆向手段也基本不可用。（nvidia实习生实现了cuda driver逆向工程。只你想了一小部分umd和kmd之间的接口，已不维护）

## k8s原理

docker镜像，容器，仓库

docker镜像：一个特殊的文件系统，提供容器运行时需要的程序、库、资源、配置等文件，还包括一些为运行时准备的配置参数（如环境变量）。不包含任何动态数据。

docker registry：负责管理docker镜像。官方docker hub中有大量高质量官方镜像。

docker用于具体业务的困难：编排、管理、调度困难



kubernetes集群：包括一个master（主结点，负责管理和控制）和很多node（计算结点，执行具体任务）

master结点：