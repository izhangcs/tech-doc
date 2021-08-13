## Pod 概述

Pod 是 k8s 系统中可以创建和管理的最小单元，是资源对象模型中由用户创建或部署的最小资源对象模型，也是 k8s 上运行容器化应用的资源对象，其他的资源对象都是用来支撑和扩展 Pod 对象功能的，比如控制器对象是用来管理 Pod 对象的，Service 或者 Ingress 资源对象是用来暴露 Pod 引用对象的，PersistentVolume 资源对象用来为Pod 提供存储等等，k8s 不会直接处理容器，而是 Pod,  Pod 是由一个或多个 container 组成的。

Pod 是 Kubernetes 的最重要概念，每个 Pod 都有一个特殊的被称为 “根容器” 的 Pause 容器。Pause 容器对应的镜像属于 Kubernetes 平台的一部分，除了 Pause 容器，每个 Pod 还包含一个或多个紧密相关的用户业务容器。

### Pod vs 应用 

每个 Pod 都是应用的一个实例，有专用的IP

### Pod vs 容器

一个 Pod 可以有多个容器，彼此间共享网络和存储资源，每个 Pod 中有一个 Pause 容器保存所有的容器状态，通过管理 pause 容器，达到管理 pod 中所有容器的效果。

### Pod vs 节点

同一个 Pod 中的容器总会被调度到相同的 Node 节点，不同节点间 Pod 的通信基于虚拟二层网络技术实现

### Pod vs Pod 

普通的Pod 和 静态的 Pod

## Pod 特性 

### 资源共享 

一个 Pod 里的多个容器 可以共享存储和网络，可以看作一个逻辑的主机。共享的如 namespace， cgroups 或者其他的隔离资源。

多个容器共享同一个 network namespace, 由此在一个 Pod 里的多个容器共享 Pod 的 IP 和端口 namespace, 所以一个 Pod 内的多个容器之间可以通过 localhost 来进行通信，所需要注意的是不同容器要注意不要端口冲突即可。不同的 Pod 有不同的 IP， 不同的 Pod 内的多个容器之前通信，不可以使用 IPC （如果么有特殊指定的话）通信，通常情况下使用 Pod 的 IP 进行通信。

一个 Pod 里的多个容器可以共享存储卷，这个存储卷会被定义为 Pod 的一部分，并且可以挂载到该 Pod 里的所有容器的文件系统上。

### 生命周期短暂

Pod 属于生命周期比较短暂的组件，比如，当 Pod 所在节点发生故障，那么该节点上的 Pod 会被调度到其他节点，但需要注意的是，被重新调度的 Pod 是一个全新的 Pod，跟之前的 Pod 没有半毛钱关系。

### 平坦的网络

k8s 集群中的所有 Pod 都在同一个共享网络地址空间中，也就是说每个 Pod 都可以通过其他 Pod 的 IP 地址来实现访问。

### Pod 的基本使用方法

在 Kubernetes 中对运行容器的要求为：容器的主程序需要一直在前台运行，而不是后台运行。应用需要改造成前台运行方式。如果我们创建的 Docker 镜像命令是后台执行程序，则在 Kubelet 创建包含这个容器的 Pod 之后运行完该命令，即认为 Pod 已经结束，将立即销毁该 Pod。如果为该 Pod 定义了 RC, 则创建、销毁会陷入一个无限循环的过程中。Pod 可以由 1个或多个容器的组合而成。

### Pod 的 分类

Pod 有两种类型

#### 普通Pod 

普通 Pod 一旦被创建，就会被放入到 etcd 中存储，随后会被 Kubernetes Matser 调度某个具体的 Node 上并进行绑定，随后该 Pod 对应的 Node 上的 Kubelet 进程实例化成一组相关的 Docker 容器并启动起来。在默认情况下，当 Pod 里某个容器停止时，Kubernetes 会自动检测到这个问题并且重新启动这个 Pod 里所有容器，如果 Pod 所在的 Node 宕机，则会将这个 Node 上的所有 Pod 重新调度到其他节点上。

### 静态Pod

 静态 Pod 是由 Kubelet 进行管理的仅存在于特定 Node 上的 Pod，它们不能通过 API Server 进行管理，无法与 ReplicationController、Deployment 或 DaemonSet  进行关联，并且 Kubelet 也无法对它们进行健康检查。

### Pod 生命周期和重启策略

#### Pod 的状态

| 状态值    | 说明                                                         |
| --------- | ------------------------------------------------------------ |
| Pending   | API Server 已经创建了该Pod, 但 Pod 中的一个或多个容器的镜像还没有创建，包括镜像下载过程 |
| Running   | Pod 内所有容器已经创建，且至少一个容器处于运行状态，正在启动状态或正在启动状态 |
| Completed | Pod 内所有容器均成功执行退出，且不会再重启                   |
| Failed    | Pod 内所有容器均已退出，但至少一个容器退出失败               |
| Unknow    | 由于某种原因无法获取 Pod 状态，例如网络通信不畅              |

#### Pod 重启策略

Pod 的重启策略包括 Always、OnFailure 和 Never, 默认值是 Always

| 重启策略  | 说明                                                       |
| --------- | ---------------------------------------------------------- |
| Always    | 当容器失效时，由 Kubelet 自动重启该容器                    |
| OnFailure | 当容器终止运行且退出码不为 0 时，由 kubelet 自动重启该容器 |
| Never     | 不论容器运行状态如何，Kubelet 都不会重启该容器             |

#### 常见状态转换

| Pod 包含的容器数 | Pod 当前的状态 | 发生事件      | Pod 的结果状态       |                         |                     |
| ---------------- | -------------- | ------------- | -------------------- | ----------------------- | ------------------- |
|                  |                |               | RestartPolicy=Always | RestartPolicy=OnFailure | RestartPolicy=Never |
| 包含一个容器     | Running        | 容器成功退出  | Running              | Succeeded               | Succeeded           |
| 包含一个容器     | Running        | 容器失败退出  | Running              | Running                 | Failure             |
| 包含两个容器     | Running        | 1个容器退出   | Running              | Running                 | Running             |
| 包含两个容器     | Running        | 容器被OOM杀掉 | Running              | Running                 | Failure             |

每个 Pod 都可以对其能使用的服务器上的计算资源设置限额，Kubernetes 中可以设置限额的计算资源有CPU与Memory两种，其中CPU的资源单位为CPU数量，是一个绝对值而非相对值。Memory 配额也是一个绝对值，它的单位是内存字节数。

Kubernetes 里，一个计算资源进行配额限定需要设定以下两个参数，Requests 该资源最小申请数量，系统必须满足要求 Limits 该资源最大允许使用的量，不能突破，当容器试图使用超过这个量的资源时，可能会被 Kubernetes Kill 并重启。

