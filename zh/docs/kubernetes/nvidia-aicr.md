---
status: Active
maintainer: pacoxu
last_updated: 2026-03-13
tags: kubernetes, nvidia, gpu, cluster-setup, recipes, helm, argocd
canonical_path: zh/docs/kubernetes/nvidia-aicr.md
---

# NVIDIA AI Cluster Runtime（AICR）

<a href="https://github.com/NVIDIA/aicr">**NVIDIA AI Cluster Runtime（AICR）**</a> 是一个 CLI 工具项目，帮助快速搭建 GPU 加速的 Kubernetes 集群。它捕获驱动、Operator、内核和系统配置的已知良好组合，并将其发布为版本锁定的 **recipe（方案）**——用于 Helm、ArgoCD 和其他部署框架的可复现工件。

## 概述

可靠地运行 GPU 加速的 Kubernetes 集群很困难。内核版本、驱动、容器运行时、Operator 和 Kubernetes 版本之间的微小差异可能导致难以诊断且重现代价高昂的故障。AICR 通过将验证过的配置编码为可组合的分层 overlay，使所有人都能使用这些配置。

### 每个 Recipe 的关键特性

- **优化**——针对特定的硬件、云、OS 和工作负载意图组合进行了调优
- **验证**——在发布前通过自动约束和兼容性检查
- **可复现**——相同的输入每次产生相同的部署

## 核心组件

### 1. `aicr` CLI

单一二进制文件，作为主要接口。关键子命令：

| 命令 | 描述 |
| ------- | ----------- |
| `aicr snapshot` | 捕获当前集群状态（GPU 硬件、驱动、OS、Operator）到 YAML 快照 |
| `aicr recipe` | 为目标环境生成验证过的、版本锁定的方案 |
| `aicr bundle` | 将方案实例化为部署就绪的 Helm chart |
| `aicr validate` | 针对实时集群快照验证方案 |

### 2. API Server（`aicrd`）

REST API 服务器，暴露与 CLI 相同的能力。设计用于 CI/CD 流水线或离线环境中的集群内使用。

### 3. 快照 Agent

一个 Kubernetes Job，捕获实时集群状态（GPU 硬件、驱动、OS、Operator）并将其保存到 ConfigMap 中，用于针对方案的验证。

## 快速开始

```bash
# 安装 CLI（Homebrew）
brew tap NVIDIA/aicr
brew install aicr

# 或使用安装脚本
curl -sfL https://raw.githubusercontent.com/NVIDIA/aicr/main/install | bash -s --

# 捕获集群当前状态
aicr snapshot --output snapshot.yaml

# 为您的环境生成验证过的方案
aicr recipe --service eks --accelerator h100 --os ubuntu \
  --intent training --platform kubeflow -o recipe.yaml

# 针对您的集群验证方案
aicr validate --recipe recipe.yaml --snapshot snapshot.yaml

# 渲染为部署就绪的 Helm chart
aicr bundle --recipe recipe.yaml --output ./bundles

# 验证您的集群（部署、性能、一致性）
aicr validate --recipe recipe.yaml --phase all --output report.json
```

`bundles/` 目录包含每个组件的 Helm chart，带有 values 文件、校验和和部署器配置。使用 `helm install` 部署，提交到 GitOps 仓库，或使用内置的 ArgoCD 部署器。

## 支持的环境

| 维度 | 此版本 |
| --------- | ------------ |
| **Kubernetes** | Amazon EKS、Google GKE、自管理（Kind） |
| **GPU** | NVIDIA H100、GB200 |
| **OS** | Ubuntu |
| **工作负载** | 训练（Kubeflow）、推理（Dynamo） |
| **组件** | GPU Operator、Network Operator、cert-manager、Prometheus 栈 |

## 工作原理

**Recipe** 是针对特定环境的版本锁定配置。您描述您的目标（云、GPU、OS、工作负载意图），方案引擎将其与已验证的 **overlay** 库进行匹配——分层配置从基础默认值通过云、加速器、OS 和工作负载特定调优自下而上组合。

**Bundler** 将方案实例化为部署就绪的工件：每个组件一个文件夹，包含 Helm values、校验和和 README。**Validator** 将方案与实时集群快照进行比较，并标记任何超出规格的内容。

这种分离意味着相同的验证配置无论使用 Helm、ArgoCD、Flux 还是自定义流水线部署都能工作。

## 供应链安全

每个 AICR 发布包括：

- SLSA Level 3 来源证明
- 签名的 SBOM
- 镜像 attestation（cosign）
- 每个组件的校验和验证

## 与其他 NVIDIA 工具的关系

| 工具 | 目的 |
| ---- | ------- |
| [GPU Operator](./nvidia-gpu-operator.md) | 管理 Kubernetes 节点上 GPU 驱动、设备插件和监控的生命周期 |
| **AICR** | 更高级别的集群方案工具，协调 GPU Operator 和其他组件到验证的、版本锁定的部署 |
| [DRA 驱动](./dra.md) | 通过动态资源分配提供灵活的 GPU 资源分配 |

AICR 位于 GPU Operator **之上**：它生成 Helm values 和部署配置，以已知良好组合安装和配置 GPU Operator（以及 Network Operator、cert-manager、Prometheus 等）。

## 学习主题

- GPU 工作负载的集群引导和配置管理
- Kubernetes 集群组件的供应链安全
- 集群生命周期管理的 GitOps 模式
- 基于方案的 Infrastructure as Code
- 每节点 GPU 管理（GPU Operator）与集群级配置管理（AICR）的区别

## 参考

- <a href="https://github.com/NVIDIA/aicr">GitHub：NVIDIA/aicr</a>
- [NVIDIA GPU Operator](./nvidia-gpu-operator.md)
- [动态资源分配（DRA）](./dra.md)
