# Kubernetes Notes

[TOC]

## Kubernetes

开源平台，用于管理生产中的容器化应用程序。适用于可能需要快速扩展和缩减(scale up and down quickly)的大型应用程序。

## k8s六层抽象

下面是 K8s 的六层概念，从最高级的抽象开始：

Deployment，ReplicaSet，Pod，Node Cluster，Node Processes，Docker Container

管理和运行 Pod 的高级 K8s API 资源有五个：Deployments、StatefulSets、DaemonSets、Jobs 和 CronJobs。

## Deployment

如果你想制作一个持续运行的无状态应用，比如 HTTP 服务器，你需要一个 Deployment。 Deployment允许您在不停机的情况下更新正在运行的应用程序。 Deployment还指定了在 Pod 死亡时重新启动 Pod 的策略。您可以从命令行或配置文件创建Deployment。

## ReplicaSet

Deployment 会创建一个 ReplicaSet，以确保您的应用程序具有所需数量的 Pod。 ReplicaSets 将根据您在 Deployment 中指定的触发器创建和扩展 Pod。

## Pod（荚）

Pod是 Kubernetes 的基本构建块。 一个 Pod 包含一组（一个或多个）容器。 通常，每个 Pod 有一个容器。

Pod 处理容器的卷(Volumes)、密钥(Secrets)和配置。

Pod 是短暂的。 它们旨在在它们死亡时自动重新启动。

当应用程序被 ReplicationSet 水平扩展时，Pod 会被复制。 每个 Pod 将运行相同的容器代码。

Pod 存在于工作节点上（Worker Nodes）。

## Master

Master 组件是 API 服务器（又名 kube-apiserver）、etcd、Scheduler（又名 kube-scheduler）、kube-controller-manager 和cloud-controller manager。

- API 服务器——公开 K8s API。 它是 Kubernetes 控制的前端。 （又名。kube-apiserver）。
- etcd — 集群状态数据的分布式键值存储，想想集群信息。
- Scheduler（调度程序） — 为新 Pod 选择节点。 （又名 kube-scheduler），想想匹配器。
- kube-controller-manager — 运行控制器来处理集群后台任务的进程。 想想集群控制器。
- cloud-controller-manager — 运行与云供应商交互的控制器。 想想云界面。

## Worker

Worker Node 的组件是 kubelet、kube-proxy 和 Container Runtime。

- kubelet — 负责工作节点上的一切。 它与 Master 的 API 服务器通信。 是工作节点的大脑。
- kube-proxy — 将连接路由到正确的 Pod。 还为服务执行跨 Pod 的负载平衡。
- Container Runtime — 下载镜像并运行容器。 例如，Docker 是一个容器运行时。

## DaemonSet

确保全部或者某些节点上必须运行一个Pod的工作负载资源（守护进程），当有节点加入集群时， 也会为他们新增一个 Pod。

通过创建DaemonSet可以确保守护进程pod被调度到每个可用节点上运行。

DaemonSet 是由控制器（controller manager）管理的 Kubernetes 工作资源对象。我们通过声明一个想要的daemonset状态，表明每个节点上都需要有一个特定的 Pod。协调控制回路会比较期望状态和当前观察到的状态。如果观察到的节点没有匹配的 Pod，DaemonSet controller将自动创建一个。

不过DaemonSet 控制器创建的 Pod 会被Kubernetes 调度器忽略，即DaemonSet Pods 由 DaemonSet 控制器创建和调度。



## Deployment

在 Kubernetes 中，一个 Deployment 的最小单元不是容器，而是 Pod。Pod 是一组容器（当然这一组也可以只有一个），它们运行在同一台服务器中，并共享一些资源。但是我们永远无法创建独立的容器：最相近的操作也只能是创建一个仅包含单一容器的一个 Pod。（假如想让 Kubernetes 创建 NGINX，完整的台词是：“我要一个 Pod，其中只包含一个容器，这个容器运行的是 nginx 镜像”。）

Kubernetes 是一个声明式系统（和指令式系统相对），这就意味着我们无法给它发出命令。我们不能说：“运行这个容器”。我们能做的只能是——描述我们需要的东西，然后等 Kubernetes 根据现有内容，同步为预期内容。

ReplicaSet 的对象结构和 Pod 很相似，只不过它还有个副本数量的字段，用于描述我们所需要的符合规范的 Pod 数量。我们可以修改现存 ReplicaSet 的副本数量，以此来完成伸缩。Kubernetes 会根据伸缩指令来创建或删除 Pod，让 Pod 数量符合要求。

Deployment 并不会直接负责 Pod 的创建和删除。它会把这些工作委托给一个或多个 ReplicaSet。在我们创建 Deployment 的时候，它会用自己的 Pod 规范创建一个 ReplicaSet。当更新一个 Deployment 并修改副本数量时，它会把更新内容传递给下游的 ReplicaSet。

需要更新 Pod 规范的时候，Deployment 会用新的 Pod 规范创建新的 ReplicaSet。新的 ReplicaSet 的初始实例数量是 0。接下来 ReplicaSet 的实例数量会逐步提升，同时逐渐减少另一个 ReplicaSet 的尺寸。

整个过程之中，请求被发送给新旧两个 ReplicaSet，用户不会感觉服务中断。全景大致如此，其中还有很多小细节，让整个过程更加健壮。