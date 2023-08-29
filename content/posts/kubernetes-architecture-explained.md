---
title: "[译] Kubernetes 架构详解"
date: 2023-08-28T21:51:20+08:00
draft: false
tags: ["K8s", "翻译"]
---

> 原文：https://devopscube.com/kubernetes-architecture-explained

## 前言

这份关于 Kubernetes 架构的综合指南旨在通过插图详细解释每个 Kubernetes 组件。

所以，如果你想，

- 理解 Kubernetes 架构
- 理解 Kubernetes 核心组件工作流程

那么您一定会喜欢这份指南。

## Kubernetes 架构

下面的 Kubernetes 架构图展示了 Kubernetes 集群的所有组件，以及外部系统如何连接到 Kubernetes 集群。

![Kubernetes Architecture](/kubernetes-architecture.png)

关于 Kubernetes，你首先应该了解的是，它是一个 **分布式** 系统。也就是说，它有多个组件，分布在网络上的不同服务器上。这些服务器可以是虚拟机，也可以是裸金属服务器。我们称之为 Kubernetes 集群。

一个 Kubernetes 集群由控制平面节点和工作节点组成。

## 控制平面

控制平面负责容器协调和维护集群的理想状态。它包括以下组件：

- kube-api-server
- etcd
- kube-scheduler
- kube-controller-manager
- cloud-controller-manager

## 工作节点

工作节点负责运行容器化应用程序。Worker 节点有以下组件：

- kubelet
- kube-proxy
- Container runtime

## Kubernetes 控制平面组件

首先，让我们来看看每个控制平面组件以及每个组件背后的重要概念。

### kube-api-server

kube-api-server 是 Kubernetes 集群的中心枢纽，负责暴露 Kubernetes API。

终端用户和其他集群组件通过 kube-api-server 与集群通信。在极少数情况下，监控系统和第三方服务可能会通过 kube-api-server 与集群交互。

所以，当您使用 kubectl 管理集群时，在后端实际上是通过 HTTP REST API 与 kube-api-server 通信。然而，集群内部组件（如调度器、控制器等）是通过 gRPC 方式。

kube-api-server 与集群中其他组件之间基于 TLS 进行通信，以防止未经授权访问集群。

![Kubernetes API Server](/kubernetes-api-server.png)

kube-api-server 负责以下工作：

- API 管理，暴露集群 API 端点并处理所有 API 请求
- 身份认证（使用客户端证书、令牌和 HTTP 基本身份验证）和授权（ABAC 和 RBAC）
- 处理 API 请求并验证 API 对象（如 Pods，Services 等）的数据
- 唯一与 etcd 通信的组件
- 协调控制平面和工作节点组件之间的所有进程
- 内置了 bastion api-server 代理。它是 API 服务器进程的一部分。它主要用于从集群外部访问 ClusterIP 服务，尽管这些服务通常只能在集群内部访问

> 要减少集群攻击面，确保 API 服务器的安全至关重要。Shadowserver 基金会进行了一项实验，发现了 38 万个可公开访问的 Kubernetes API 服务器。

### etcd

Kubernetes 是一个分布式系统，因此需要像 etcd 这样高效的分布式数据库来支持其分布式特性。它充当后端服务发现和数据库的双重角色。你可以把它称为 Kubernetes 集群的大脑。

etcd 是一款开源的强一致性，分布式 KV 数据库。是什么意思呢？

- 强一致性：如果对某个节点进行了更新，强一致性将确保它能立即更新到集群中的所有其他节点。此外，根据 CAP 定理，利用强一致性和分区容忍度实现 100% 的可用性是不可能的
- 分布式：etcd 被设计为在多个节点上作为集群运行，同时不牺牲一致性
- KV 数据库：以键和值的形式存储数据的非关系型数据库。它还提供键值 API。数据存储建立在 BboltDB 之上，而 BboltDB 是 BoltDB 的分叉

etcd 采用 Raft 共识算法，具有很强的一致性和可用性。它以领导者-成员的方式工作，实现高可用性并容忍部分节点故障。

那么，etcd 如何与 Kubernetes 协同工作？

简单来说，使用 kubectl 获取 Kubernetes 对象详细信息时，是从 etcd 获取的。此外，在部署 Pod 等对象时，也会在 etcd 中创建一个条目。

简而言之，以下是您需要了解的有关 etcd 的信息：

- 存储 Kubernetes 对象（pods、secrets、daemonsets、deployments、configmaps、statefulsets 等）的所有配置、状态和元数据
- 允许客户端使用 Watch API 订阅事件。kube-api-server 使用 etcd 的 Watch 功能来跟踪对象状态变化
- 使用 gRPC 暴露键值 API。此外，gRPC 网关是一个 RESTful 代理，可将所有 HTTP API 调用转换为 gRPC 消息。这使它成为 Kubernetes 的理想数据库
- 以键值格式将所有对象存储在 /registry 目录键下。例如，在 default 命名空间中名为 Nginx 的 Pod 的信息可在 /registry/pods/default/nginx 下找到

![Kubernetes etcd](/kubernetes-etcd.png)

此外，etcd 是控制平面中唯一的 Statefulset 组件。

### kube-scheduler

kube-scheduler 负责调度工作节点上的 Pod。

部署 Pod 时，您可以指定 Pod 的需求，例如 CPU、内存、亲和性、污点或容忍度、优先级、持久卷 (PV) 等。kube-scheduler 的主要任务是识别创建请求并为 Pod 选择最佳工作节点。

下图展示了 kube-scheduler 工作方式的高级概览。

![Kubernetes Scheduler](/kubernetes-scheduler.png)

在 Kubernetes 集群中，会存在多个工作节点。那么，kube-scheduler 如何从所有工作节点中选择节点呢？

以下是 kube-scheduler 的工作原理：

- 为了选择最佳节点，需要经过工作节点过滤和评分两个阶段
- 筛选阶段，会找到所有适合调度 Pod 的工作节点。例如，如果有 5 个工作节点的资源可用来运行 Pod，就会选择这 5 个节点。如果没有节点，则 Pod 不可调度，并被移至调度队列。如果是一个大型集群，例如有 100 个工作节点，不会遍历所有节点。有一个名为 percentageOfNodesToScore 的调度器配置参数。默认值通常为 50%。会尝试以循环方式遍历 50% 的节点。如果工作节点分布在多个区域，那么会对不同区域的节点进行迭代。对于超大集群，默认值为 5%
- 评分阶段，给筛选出的工作节点分配分数对节点进行排名。它通过调用多个调度插件进行评分。最后，排名最高的工作节点将被选中，用于调度 Pod。如果所有工作节点的排名相同，则会随机选择一个
- 一旦选择了节点，调度器就会在 API 服务器中创建一个 Pod 与工作节点绑定的事件

关于 kube-scheduler 的注意事项：

- 它是一个控制器，用于监听 kube-api-server 中的 Pod 创建事件
- 调度有两个阶段。调度和绑定。这两个阶段称为调度上下文。调度阶段选择一个工作节点，绑定阶段将这一变化应用到集群中
- 高优先级 Pod 在低优先级 Pod 之前进行调度。此外，在某些情况下，当 Pod 在选定工作节点开始运行后，可能会被驱逐或转移到其他工作节点
- 可以创建自定义调度程序，并在集群中与本地调度程序一起运行。部署 Pod 时，可以在 Pod 元信息中指定自定义调度程序。因此，调度决策将根据自定义调度程序逻辑执行
- 调度程序有一个可插拔的调度框架。也就是说，你可以在调度工作流程中添加自定义插件

### kube-controller-manager

什么是控制器？控制器是运行无限控制循环的程序。也就是说，它持续运行并观察对象的实际状态和期望状态。如果实际状态与期望状态存在差异，它就会确保 Kubernetes 资源/对象处于期望状态。

根据官方文档：

> 在 Kubernetes 中，控制器是一个控制循环，它负责观察集群的状态，然后根据需求做出更改。每个控制器都试图让当前集群状态更接近所期望的状态。

假设您要创建一个 Deployment，您可以在 YAML 文件中指定所期望的状态（声明式方法）。例如，2 个副本、1 个卷挂载、配置等。内置 Deployment 控制器可确保 Deployment 始终处于所期望的状态。如果用户将 Deployment 更新为 5 个副本，Deployment 控制器会识别并确保是 5 个副本。

kube-controller-manager 是一个管理所有 Kubernetes 控制器的组件。Kubernetes 资源/对象（如 Pods、命名空间、作业、副本集）由相应的控制器管理。此外，Kube 调度器也是由 Kube 控制管理器管理的控制器。

![Kubernetes Controller Manager](/kubernetes-controller-manager.png)

以下是重要的内置 Kubernetes 控制器的列表:

- Deployment Controller
- ReplicaSet Controller
- DaemonSet Controller
- Job Controller
- CronJob Controller
- Endpoints Controller
- Namespace Controller
- Service Account Controller
- Node Controller

下面是你需要了解的关于 kube-controller-manager 的信息：

- 它管理所有控制器，而控制器则努力使集群保持在所期望的状态
- 您可以使用与自定义资源定义相关联的自定义控制器来扩展 Kubernetes。

### cloud-controller-manager

在云环境中部署 Kubernetes 时，云控制器管理器充当云平台 API 与 Kubernetes 集群之间的桥梁。

这样，Kubernetes 核心组件就可以独立工作，并允许云提供商使用插件与 Kubernetes 集成。(例如，Kubernetes 集群与 AWS 云 API 之间的接口）

云控制器集成允许 Kubernetes 集群调配云资源，如实例（节点）、负载均衡器（服务）和存储卷（持久卷）。

![Kubernetes Cloud Platform](/kubernetes-cloud-platform.png)

云控制器管理器包含一组特定于云平台的控制器，可确保云特定组件（节点、负载均衡器、存储等）的理想状态。以下是云控制器管理器中的三个主要控制器：

- Node Controller：该控制器通过调用云平台的 API 来更新节点相关信息。例如，节点标签和注释、获取主机名、CPU 和内存可用性、节点健康状况等
- Route Controller：它负责在云平台上配置网络路由。这样，不同节点上的 Pod 就能相互通信
- Service Controller：它负责为 Kubernetes 服务部署负载均衡器、分配 IP 地址等

以下是云控制器管理器的一些经典示例：

- 部署负载均衡器类型的 Kubernetes 服务。在这里，Kubernetes 提供了一个特定于云的负载均衡器，并与 Kubernetes 服务集成
- 为云存储解决方案支持的 Pod 配置存储卷 (PV)

总体云控制器管理器管理 Kubernetes 使用的特定云资源的生命周期。

## Kubernetes 工作节点组件

现在我们来看看每个工作节点组件。

### kubelet

kubelet 是一个代理组件，在集群的每个节点上运行。它不作为容器运行，而是作为由 systemd 管理的守护进程运行。

它负责将工作节点注册到 API 服务器，并主要与 API 服务器上的 podSpec（Pod 规范 - YAML 或 JSON）一起工作。podSpec 定义了应在 Pod 内运行的容器、它们的资源（例如 CPU 和内存限制）以及其他设置，例如环境变量、卷和标签。

然后，它通过创建容器使 podSpec 达到所期望的状态。

简单来说，kubelet 负责以下工作：

- 为 Pod 创建、修改和删除容器
- 负责处理 liveliness、readiness 和 startup 探针
- 负责通过读取 Pod 配置来挂载卷，并在主机上创建相应的卷挂载目录
- 通过调用 API 服务器收集并报告节点和 Pod 状态。

kubelet 也是一个控制器，它可以监控 Pod 的变化，并利用节点的容器运行环境来拉取镜像、运行容器等。

除了来自 API 服务器的 podSpec，kubelet 还能接受来自文件、HTTP 端点和 HTTP 服务器的 podSpec。例如 Kubernetes 静态 Pod，就是 podSpec 来自文件的例子。

静态 Pod 由 kubelet 控制而非 API 服务器。

这意味着您可以通过向 kubelet 组件提供 Pod YAML 文件来创建 Pod。不过，kubelet 创建的静态 Pod 不受 API 服务器管理。

下面是一个静态 Pod 的实际使用案例。

在启动控制平面时，kubelet 会从位于 /etc/kubernetes/manifests 目录中的静态 Pod YAML 文件中读取 podSpec。然后以静态 Pod 的形式启动 api-server、scheduler 和 controller-manager。

以下是有关 kubelet 的一些关键信息：

- kubelet 使用 CRI（容器运行时接口）gRPC 接口与容器运行环境通信
- 它还暴露了一个 HTTP 端点，用于流式传输日志，并为客户端提供执行会话
- 使用 CSI（容器存储接口）gRPC 配置块卷
- 它使用群集中配置的 CNI 插件来分配 Pod IP 地址，并为 Pod 设置必要的网络路由和防火墙规则

![Kubernetes Kubelet](/kubernetes-kubelet.png)

### kube-proxy

要理解 kube-proxy，你需要掌握 Kubernetes Service 和 Endpoint 对象的基本知识。

Kubernetes Service 是一种向内部或外部流量暴露一组 Pod 的方法。创建 Service 对象时，会为其分配一个虚拟 IP。它被称为 clusterIP。它只能在 Kubernetes 集群内访问。

Endpoint 对象包含 Service 对象下 Pod 组的所有 IP 地址和端口。端点控制器负责维护 Pod IP 地址（端点）列表。服务控制器负责为服务配置端点。

您无法 ping 通 ClusterIP，因为它只用于服务发现，不像 Pod IP 可以 ping 通。

现在我们来了解一下 kube-proxy。

kube-proxy 是一个守护进程，运行在每个节点上。它是一个代理组件，为 Pod 实现了 Kubernetes 服务概念。(为一组 Pod 提供单一 DNS 并实现负载均衡）它主要代理 UDP、TCP 和 SCTP，不理解 HTTP。

当您使用 Service（ClusterIP）暴露 Pod 时，kube-proxy 会创建网络规则，将流量路由到 Service 对象下分组的后端 Pod（端点）。这意味着，所有的负载均衡和服务发现都由 kube-proxy 处理。

那么，kube-proxy 是如何工作的呢？

kube-proxy 与 API 服务器进行通信，以获取有关 Service（ClusterIP）和相关 Pod IP 及端口（端点）的详细信息。它还会监控服务和端点的变化。

然后，kube-proxy 会使用以下任何一种模式来创建 / 更新规则，以便将流量路由到服务后面的 Pod：

- IPTables：这是默认模式。在 IPTables 模式下，流量由 IPTable 规则处理。在该模式下，kube-proxy 会随机选择后端 Pod 进行负载均衡。连接建立后，请求会转到同一个 Pod，直到连接终止
- IPVS：对于服务超过 1000 个的集群，IPVS 可提高性能。它支持以下后端负载均衡算法
  - round-robin：默认算法，依次转发
  - least-connection：连接数最少
  - destination-hash：基于客户端 IP 地址的哈希算法
  - source-hash：基于后端 Pod IP 地址的哈希算法
  - shortest-expected-delay：延迟最短
  - never-queue：无需排队
- Userspace：废弃的模式，不推荐使用
- Kernelspace：Windows 集群使用此模式

![Kubernetes Kube Proxy](/kubernetes-kube-proxy.png)

如果您想了解 kube-proxy IPTables 和 IPVS 模式之间的性能差异，[请阅读本文](https://www.tigera.io/blog/comparing-kube-proxy-modes-iptables-or-ipvs)。

此外，你还可以用 Cilium 代替 kube-proxy，在没有 kube-proxy 的情况下运行 Kubernetes 集群。

### Container Runtime

您可能知道 Java Runtime (JRE)。它是在主机上运行 Java 程序所需的软件。同样，Container Runtime 也是运行容器所需的软件组件。

Container Runtime 在 Kubernetes 集群的所有节点上运行。它负责拉取镜像、运行容器、为容器分配和隔离资源，以及管理主机上容器的整个生命周期。

为了更好地理解这一点，让我们看一下两个关键概念：

- Container Runtime Interface（CRI）：这是一组允许 Kubernetes 与不同容器运行时交互的 API。它允许不同的 Container Runtime 与 Kubernetes 交互使用。CRI 定义了用于创建、启动、停止和删除容器以及管理映像和容器网络的 API
- Open Container Initiative（OCI）：这是一套容器格式和运行时标准

Kubernetes 支持符合 Container Runtime Interface（CRI）的多种容器运行时（CRI-O、Docker Engine、containerd 等）。这意味着，所有这些容器运行时都实现了 CRI 接口，并公开了 gRPC CRI API（运行时和镜像服务端点）。

那么，Kubernetes 是如何利用容器运行时的呢？

正如我们在 kubelet 部分学到的，kubelet 代理负责使用 CRI API 与容器运行时交互，以管理容器的生命周期。它还从容器运行时获取所有容器信息，并将其提供给控制平面。

让我们以 CRI-O 容器运行时接口为例。以下是容器运行时如何与 kubernetes 协同工作的高级概述。

![Kubernetes CRI](/kubernetes-cri.png)

- 当 API 服务器发起新的 Pod 请求时，kubelet 会与 CRI-O 守护进程对话，通过 Kubernetes 容器运行时接口启动所需的容器
- CRI-O 使用容器 / 镜像库检查并从配置的 Docker Registry 中拉取所需的容器镜像
- CRI-O 为容器生成 OCI 运行时规范（JSON）
- CRI-O 会启动与 OCI 兼容的运行时（runc），按照运行时规范启动容器进程

## Kubernetes 集群附加组件

除核心组件外，Kubernetes 集群还需要附加组件才能完全运行。选择附加组件取决于项目要求和用途。

以下是集群中可能需要的一些常用附加组件：

- CNI 插件（容器网络接口）
- CoreDNS（用于 DNS 服务器）：CoreDNS 在 Kubernetes 集群中充当 DNS 服务器。启用此插件后，就能启用基于 DNS 的服务发现
- Metrics 服务器（用于资源统计）：此插件可帮助您收集集群中节点和 Pod 的性能数据和资源使用情况
- Web UI（Kubernetes 控制面板）：该插件可启用 Kubernetes 仪表板，以便通过 Web UI 管理对象

### CNI Plugin

首先，您需要了解容器网络接口（CNI）

它是一种基于插件的架构，具有厂商中立的规范和库，可为容器创建网络接口。

它并不局限于 Kubernetes。有了 CNI，容器网络可以在 Kubernetes、Mesos、CloudFoundry、Podman、Docker 等容器编排工具之间实现标准化。

说到容器网络，企业可能会有不同的要求，如网络隔离、安全、加密等。随着容器技术的发展，许多网络提供商为容器创建了基于 CNI 的解决方案，并提供了广泛的网络功能。您可以将其称为 CNI-Plugins。

这样，用户就可以从不同的供应商那里选择最适合自己需求的网络解决方案。

CNI 插件如何与 Kubernetes 配合使用？

- kube-controller-manager 负责为每个节点分配 Pod CIDR。每个 Pod 都能从 Pod CIDR 获取一个唯一的 IP 地址
- kubelet 与容器运行时交互，以启动被调度的 Pod。作为容器运行时一部分的 CRI 插件与 CNI 插件交互，以配置 Pod 网络
- CNI Plugin 可让分布在相同或不同节点上的 Pod 之间通过覆盖网络进行联网

![Kubernetes CNI](/kubernetes-cni.png)

以下是 CNI 插件提供的高级功能：

- Pod 网络
- 使用网络策略控制 Pod 之间和命名空间之间的流量，实现 Pod 网络安全和隔离

一些流行的 CNI 插件包括：

- Calico
- Flannel
- Weave Net
- Cilium
- Amazon VPC CNI
- Azure CNI

## Kubernetes 架构常见问题解答

### Kubernetes 控制平面的主要用途是什么？

控制平面负责维护集群及其上运行的应用程序的理想状态。它由 API 服务器、etcd、调度程序和控制器管理器等组件组成。

### Kubernetes 集群中工作节点的用途是什么？

工作节点是在集群中运行容器的服务器（裸金属服务器或虚拟机）。它们由控制平面管理，并接收控制平面的指令。

### Kubernetes 中控制平面和工作节点之间的通信如何保证安全？

控制平面和工作节点之间的通信使用 PKI 证书进行保护，不同组件之间通过 TLS 进行通信。这样，只有受信任的组件才能相互通信。

### Kubernetes 中 etcd 键值存储的用途是什么？

etcd 主要存储 kubernetes 对象、集群信息、节点信息以及集群的配置数据，例如集群上运行的应用程序的期望状态。

### 如果 etcd 宕机，Kubernetes 应用程序会发生什么？

如果 etcd 发生中断，正在运行的应用程序不会受到影响，但无法创建或更新任何对象。

## 结论

理解 Kubernetes 架构有助于您进行日常 Kubernetes 实施和操作。

在实施生产级集群设置时，正确了解 Kubernetes 组件将有助于您运行应用程序并对其进行故障排除。

接下来，您可以从分步 kubernetes 教程开始，获得 Kubernetes 对象和资源的实践经验。
