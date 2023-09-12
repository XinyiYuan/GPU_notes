# Kubernetes Notes

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