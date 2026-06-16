---
status: Active
maintainer: pacoxu
last_updated: 2026-03-30
tags: inference, grove, kubernetes, gang-scheduling, disaggregated-inference, nvidia
canonical_path: zh/docs/inference/grove.md
---

# Grove：适用于任何推理架构的统一 API

[`Grove`](https://github.com/ai-dynamo/grove) 是一个 Kubernetes API，提供单一的声明式接口来编排任何 AI 推理工作负载——从简单的单 Pod 部署到复杂的多节点分离式系统。Grove 让您可以将多节点推理部署从单个副本扩展到数据中心级别，支持数万个 GPU。

Grove 允许您将整个推理服务系统在 Kubernetes 中描述为单个自定义资源（CR）——例如 prefill、decode、路由或任何其他组件。从这一规范出发，平台协调分层 Gang 调度、拓扑感知放置、多级自动缩放和显式启动顺序。

## 概述

现代 AI 推理工作负载需要 Kubernetes 本身不具备的能力：

- **多节点/多 Pod 单元的缩放**——大型模型跨多个节点分片。基本缩放单位不再是单个 Pod，而是共同构成一个模型实例的整个 Pod 组。
- **分层 Gang 调度**——多节点模型实例要求 Pod 一起调度；部分子集会留下空闲资源并可能导致死锁。分离式推理也有类似约束：必须至少调度一个 prefill 和一个 decode 实例才能形成功能流水线。
- **启动顺序**——即使组件必须一起调度，它们通常也需要按特定顺序启动（例如，MPI 工作负载要求所有工作进程就绪后领导者才能启动）。
- **拓扑感知放置**——推理组件间通信密集；网络优化放置（例如在 NVLink 域内）对于最小化通信开销和最大化性能至关重要。

## 核心概念

Grove 引入了四个核心抽象：

| 概念 | 描述 |
| --- | --- |
| **PodClique** | 代表特定角色（例如领导者、工作者、前端）的一组 Pod。每个 Clique 具有独立配置并支持自定义缩放逻辑。 |
| **PodCliqueScalingGroup** | 一组 PodClique，作为 Gang 一起缩放和调度。适用于紧密耦合的角色，如 prefill 领导者和工作者。 |
| **PodCliqueSet** | Grove 顶层对象，定义了一组一起管理和共置的组件。支持具有拓扑感知分布的自动缩放以实现可用性。 |
| **PodGang** | 调度器 API，定义了一个 Gang 调度单元——一组 Pod 组，其中每个组定义了调度保证的最小副本数。 |

## 关键特性

- **分层 Gang 调度**——确保所有需要的 Pod（跨多个角色和缩放组）作为一个单元原子化调度。
- **拓扑感知放置**——将 Pod 放置在 NVLink 域或其他网络拓扑边界内，最小化节点间通信延迟。
- **多级自动缩放**——在 PodCliqueScalingGroup 级别保持 Gang 约束的同时独立缩放 PodClique。
- **启动顺序**——保证跨 Clique 的指定启动序列（例如，工作者在领导者之前启动）。
- **滚动更新**——支持可配置的滚动更新，同时尊重 Gang 调度不变性。
- **单一 CRD 简洁性**——整个推理系统（prefill + decode + router）表达为一个 `PodCliqueSet` CR——无需脚本、多个 YAML 文件或自定义控制器。

## 示例用例

- **多节点分离式推理**适用于跨数万个 GPU 的大型模型（例如 DeepSeek-R1、Llama-4-Maverick）。
- **单节点分离式推理**，具有有序的 prefill 和 decode 启动。
- **Agentic 流水线**，其中一系列专门的模型必须一起放置和缩放。
- **标准聚合推理**——单节点或单 GPU 部署，具有完整生命周期管理。

## 与 NVIDIA Dynamo 的关系

Grove 作为 [NVIDIA Dynamo](https://github.com/ai-dynamo/dynamo)（NVIDIA 的分布式推理框架）的 Kubernetes 编排层。Dynamo 支持两种部署模式：

- **Grove 模式**——NVIDIA 原生，使用 Grove CRD 实现完整的 Gang 调度和拓扑感知。
- **LWS 模式**——Kubernetes 原生，使用 [LeaderWorkerSet](https://github.com/kubernetes-sigs/lws) 实现轻量级多节点部署。

Grove 也可以与任何推理引擎独立于 Dynamo 使用。

## 快速开始

```bash
# 1. 创建本地 kind 集群
cd operator && make kind-up

# 2. 部署 Grove
make deploy

# 3. 部署您的第一个工作负载
kubectl apply -f samples/simple/simple1.yaml

# 4. 检查创建的资源
kubectl get pcs,pclq,pcsg,pg,pod -o wide
```

完整指南请参见[快速入门指南](https://github.com/ai-dynamo/grove/blob/main/docs/quickstart.md)和[安装文档](https://github.com/ai-dynamo/grove/blob/main/docs/installation.md)。

## 路线图

### 已完成（2025）

- 分层 Gang 调度 ✅
- 多级水平自动缩放 ✅
- 启动顺序 ✅
- 滚动更新 ✅
- 拓扑感知调度 ✅

### 计划中（2026）

- 资源优化的滚动更新
- 拓扑分布约束
- 自动拓扑检测

## 学习主题

- Gang 调度概念和 Kubernetes 调度器扩展点
- 基于 CRD 的工作负载编排模式
- NVLink / RDMA 网络的拓扑感知放置
- 分离式推理架构（prefill + decode）
- 推理系统的多级自动缩放策略

## 参考资料

- [GitHub: ai-dynamo/grove](https://github.com/ai-dynamo/grove)
- [核心概念概览](https://github.com/ai-dynamo/grove/blob/main/docs/user-guide/01_core-concepts/01_overview.md)
- [API 参考](https://github.com/ai-dynamo/grove/blob/main/docs/api-reference/operator-api.md)
- [NVIDIA Dynamo](https://github.com/ai-dynamo/dynamo)
- [Grove 邮件列表](https://groups.google.com/g/grove-k8s)
- [NVIDIA Dynamo Discord](https://discord.gg/UxcbxEYqS4)
