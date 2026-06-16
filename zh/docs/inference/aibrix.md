---
status: Active
maintainer: pacoxu
last_updated: 2026-04-06
tags: inference, aibrix, vllm, llm-serving, lora
canonical_path: zh/docs/inference/aibrix.md
---

# AIBrix：云原生 LLM 推理基础设施

[`AIBrix`](https://github.com/vllm-project/aibrix) 是一个开源计划，旨在提供必要的构建模块来构建可扩展的 GenAI 推理基础设施。AIBrix 提供云原生解决方案，专为部署、管理和扩展大型语言模型（LLM）推理而优化，特别针对企业需求定制。

## 概述

AIBrix 是 vLLM 项目生态系统的一部分，专注于为企业级大规模 LLM 推理提供基础设施。它通过高级路由、自动缩放和分布式推理能力等功能，解决了在生产环境中部署和管理 LLM 工作负载的挑战。

## 关键特性

- **高密度 LoRA 管理**：简化对模型轻量级低秩适配的支持
- **LLM 网关和路由**：高效管理和跨多个模型及副本引导流量
- **LLM 应用定制自动缩放器**：基于实时需求动态缩放推理资源
- **统一 AI 运行时**：实现指标标准化、模型下载和管理的多功能 Sidecar
- **分布式推理**：跨多个节点处理大型工作负载的可扩展架构
- **分布式 KV 缓存**：实现大容量、跨引擎的 KV 重用
- **高性价比异构服务**（实验性）：支持混合 GPU 推理以降低成本，同时提供 SLO 保证
- **GPU 硬件故障检测**：主动检测 GPU 硬件问题

## 架构

AIBrix 采用为 Kubernetes 环境设计的云原生架构。系统由多个协同工作的组件组成，提供全面的 LLM 推理平台：

- **控制面**：管理推理工作负载的生命周期
- **网关**：处理请求路由和负载均衡
- **运行时**：管理模型服务和资源分配
- **自动缩放器**：基于 LLM 特定指标提供智能缩放

## 学习主题

- LLM 感知负载均衡和路由
- 动态 LoRA 切换和管理
- 分布式推理架构
- GPU 资源优化和故障检测
- 高性价比异构服务策略
- 企业级 LLM 部署模式

## 项目状态

- **当前版本**：v0.6.0（截至 2026 年 3 月）
- **许可证**：Apache 2.0
- **成熟度**：活跃开发中，定期发布
- **社区**：vLLM 项目的一部分，拥有专属 Slack 频道

## Prefill/Decode 分离支持

AIBrix v0.4.0 引入了全面的 Prefill/Decode（P/D）分离支持，包含 StormService 和 RoleSet CRD，以实现 P/D 角色的细粒度编排以及路由功能，解锁大规模分离式推理。这些能力已成为当前 v0.6.0 版本发布线的一部分。

关键特性包括：

- StormService 和 RoleSet 自定义资源定义（CRD）
- Prefill 和 Decode 角色的细粒度编排
- 用于分离式推理的高级路由能力
- 大规模部署的可扩展架构

此功能通过 v0.4.0 版本中的多个 PR 交付，包括对控制面、网关路由和分布式推理能力的增强。

更多详情请参见 [PD 分离](./pd-disaggregation.md)。

## 快速开始

### 快速安装

```bash
# 安装组件依赖
AIBRIX_VERSION="v0.6.0"
GITHUB_BASE="https://github.com/vllm-project/aibrix/releases/download"
kubectl apply -f \
  "${GITHUB_BASE}/${AIBRIX_VERSION}/aibrix-dependency-${AIBRIX_VERSION}.yaml" \
  --server-side

# 安装 AIBrix 组件
kubectl apply -f \
  "${GITHUB_BASE}/${AIBRIX_VERSION}/aibrix-core-${AIBRIX_VERSION}.yaml"
```

### 开发设置

```bash
# 克隆仓库
git clone https://github.com/vllm-project/aibrix.git
cd aibrix

# 安装夜间依赖
kubectl apply -k config/dependency --server-side

# 安装夜间组件
kubectl apply -k config/default
```

## 资源

- **文档**：[aibrix.readthedocs.io](https://aibrix.readthedocs.io/latest/)
- **博客**：[aibrix.github.io](https://aibrix.github.io/)
- **白皮书**：[arxiv.org/abs/2504.03648](https://arxiv.org/abs/2504.03648)
- **Slack**：[#aibrix](https://vllm-dev.slack.com/archives/C08EQ883CSV)
- **GitHub**：[vllm-project/aibrix](https://github.com/vllm-project/aibrix)

## 重要演讲

- **KubeCon China 2025**：[AIBrix: Cost-Effective and Scalable Kubernetes Control Plane for vLLM](https://kccncchn2025.sched.com/event/1x5im/)
- **KubeCon EU 2025**：[LLM-Aware Load Balancing in Kubernetes: A New Era of Efficiency](https://kccnceu2025.sched.com/event/1txC7/)（与 Google 联合主题演讲）
- **ASPLOS'25 Workshop**：[AIBrix: An Open-Source, Large-Scale LLM Inference Infrastructure for System Research](https://docs.google.com/presentation/d/1YDVsPFTIgGXnROGaJ1VKuDDAB4T5fzpE/edit)
- **FORCE Conference**：[AIBrix 基于 vLLM 的高性价比 LLM 推理加速方案—FORCE 大会直播回放](https://www.bilibili.com/video/BV1q6MezaEtP/)（中文）

---

**注意**：部分内容可能由 AI 工具生成。在生产环境使用前请核实信息。
