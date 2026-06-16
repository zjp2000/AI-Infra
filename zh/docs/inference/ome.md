---
status: Active
maintainer: pacoxu
last_updated: 2025-10-29
tags: inference, ome, kubernetes-operator, llm-management
canonical_path: zh/docs/inference/ome.md
---

# OME：用于 LLM 管理的 Kubernetes Operator

OME 是一个全面的 Kubernetes Operator，专为企业级大规模语言模型（LLM）的管理和服务而设计。由 SGL 项目团队开发，OME 通过自动模型管理、智能运行时选择和复杂的部署模式来优化 LLM 的部署和运维。

## 概述

[`OME`](https://github.com/sgl-project/ome) 在 LLM 推理领域中的独特之处在于将模型视为 Kubernetes 一等公民自定义资源，支持复杂的模型解析、自动运行时选择和高级部署模式，包括 prefill-decode 分离。

**注意**：[贡献者页面](https://github.com/sgl-project/ome/graphs/contributors)显示，截至目前这是一个单人项目（104/168 次提交）。

## 关键特性

### 模型管理

- **一等公民自定义资源**：模型作为原生 Kubernetes 资源进行管理，具有全面的元数据提取
- **架构检测**：自动解析模型文件以提取架构、参数数量和能力
- **多格式支持**：支持 SafeTensors、PyTorch、TensorRT 和 ONNX 模型格式
- **分布式存储**：自动修复、双重加密和命名空间范围限制，确保模型存储安全

### 智能运行时选择

- **自动匹配**：加权评分系统将模型与最优运行时配置匹配
- **多因素分析**：考虑架构、格式、量化、参数大小和框架兼容性
- **性能优化**：确保每个模型在最合适的推理引擎上运行

### 高级部署模式

- **Prefill-Decode 分离**：支持复杂的 PD 分离以提高资源利用率
- **多节点推理**：将大型模型分布到多个节点
- **传统部署**：具有高级缩放控制的标准 Kubernetes 部署

### 资源优化

- **GPU Bin-packing**：专用调度算法实现最优 GPU 利用率
- **动态重新优化**：持续集群效率优化，同时保持高可用性
- **智能资源分配**：跨异构环境最大化硬件利用率

## 运行时集成

### SGLang（一级支持）

OME 提供与 [`SGLang`](https://github.com/sgl-project/sglang) 的深度集成，具有以下特性：

- 缓存感知负载均衡
- 多节点部署能力
- Prefill-decode 分离服务
- 多 LoRA 适配器服务
- 高级推理优化

### vLLM 支持

OME 支持 [`vLLM`](https://github.com/vllm-project/vllm) 作为高性能运行时选项，并提供[内置配置](https://github.com/sgl-project/ome/tree/main/config/runtimes/vllm)用于优化的 LLM 服务。

## Kubernetes 生态系统集成

OME 与现代 Kubernetes 组件深度集成：

- **[Kueue](https://kueue.sigs.k8s.io/)**：多 Pod 工作负载的 Gang 调度
- **[LeaderWorkerSet](https://github.com/kubernetes-sigs/lws)**：弹性的多节点部署
- **[KEDA](https://keda.sh/)**：基于高级自定义指标的自动缩放
- **[K8s Gateway API](https://gateway-api.sigs.k8s.io/)**：复杂的流量路由
- **[Gateway API Inference Extension](https://gateway-api-inference-extension.sigs.k8s.io/)**：标准化的推理端点

## 核心架构

OME 使用基于 Kubernetes 自定义资源的组件化架构：

### 自定义资源

- **BaseModel/ClusterBaseModel**：定义模型来源和元数据
- **ServingRuntime/ClusterServingRuntime**：定义模型的提供方式
- **InferenceService**：将模型连接到运行时进行部署
- **BenchmarkJob**：测量模型在不同工作负载下的性能

### 自动化工作流

1. **模型发现**：下载并解析模型以了解其特征
2. **运行时选择**：为每个模型选择最优运行时配置
3. **资源生成**：生成用于高效部署的 Kubernetes 资源
4. **持续优化**：优化集群中的资源利用率

## 生产就绪

OME 已为生产环境做好准备：

- ✅ API 版本：v1beta1
- ✅ 全面的文档
- ✅ 单元和集成测试覆盖
- ✅ 大规模生产部署
- ✅ 标准 Kubernetes 监控和事件
- ✅ 基于 RBAC 的安全和模型加密
- ✅ 具有冗余存储的高可用性

## 安装

### 要求

需要 Kubernetes 1.28 或更新版本

### 快速开始（OCI 注册表）

```bash
# 安装 OME CRDs
helm upgrade --install ome-crd oci://ghcr.io/moirai-internal/charts/ome-crd \
  --namespace ome --create-namespace

# 安装 OME 资源
helm upgrade --install ome oci://ghcr.io/moirai-internal/charts/ome-resources \
  --namespace ome
```

### Helm 仓库

```bash
# 添加仓库
helm repo add ome https://sgl-project.github.io/ome
helm repo update

# 安装组件
helm upgrade --install ome-crd ome/ome-crd \
  --namespace ome --create-namespace
helm upgrade --install ome ome/ome-resources --namespace ome
```

## 学习主题

- Kubernetes 自定义资源开发和模式
- LLM 推理优化策略
- 多节点分布式服务架构
- 资源调度和 Bin-packing 算法
- 模型管理和元数据提取
- Kubernetes 中的生产 MLOps 模式

## 与其他平台的比较

OME 通过以下方面脱颖而出：

- **原生 Kubernetes 集成**：基于 CRD 的深度架构 vs. 外部编排
- **智能运行时选择**：自动匹配 vs. 手动配置
- **高级部署模式**：内置 PD 分离和多节点支持
- **模型优先设计**：模型作为一等资源 vs. 以部署为中心的方法

## 路线图

- 增强模型解析能力，支持更多模型族和架构
- 支持模型量化和优化工作流
- 跨多个 Kubernetes 集群的联邦
- 扩展运行时生态系统集成

## 参考资料

- [OME GitHub 仓库](https://github.com/sgl-project/ome)
- [OME 文档](https://sgl-project.github.io/ome/docs/)
- [SGLang 项目](https://github.com/sgl-project/sglang)
- [安装指南](https://sgl-project.github.io/ome/docs/installation/)
- [API 参考](https://sgl-project.github.io/ome/docs/reference/ome.v1beta1/)

---

**注意**：本文档提供 OME 功能的概述。有关详细的实现指南和高级配置，请参考官方 OME 文档。
