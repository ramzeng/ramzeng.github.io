---
title: "[译] Kubernetes 架构详解"
date: 2023-08-28T21:51:20+08:00
draft: true
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
- kube-apiserver
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

### kube-apiserver
kube-apiserver 是 Kubernetes 集群的中心枢纽，负责暴露 Kubernetes API。

终端用户和其他集群组件通过 API 服务器与集群通信。在极少数情况下，监控系统和第三方服务可能会通过 API 服务器与集群进行交互。

所以，当您使用 kubectl 管理集群时，在后端实际上是通过 HTTP REST API 与 API 服务器进行通信。然而，集群内部组件（如调度器、控制器等）是通过 gRPC 与 API 服务器通信。

API 服务器与集群中其他组件之间基于 TLS 进行通信，以防止未经授权访问集群。

![Kubernetes API Server](/kubernetes-apiserver.png)

work in progress...
