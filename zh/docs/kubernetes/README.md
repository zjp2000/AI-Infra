---
status: Active
maintainer: pacoxu
last_updated: 2026-05-09
tags: kubernetes, ai-infrastructure, scheduling, resource-management
canonical_path: zh/docs/kubernetes/README.md
---

# Kubernetes for AI Infrastructure

面向 AI 工作负载的 Kubernetes 综合指南，涵盖容器生命周期、调度优化、资源管理和工作负载隔离。

## 学习路径

### 入门

- **[Kubernetes 学习计划](./learning-plan.md)**：结构化的三阶段学习方法，涵盖 Docker 基础知识、核心 Kubernetes 概念和云原生生态系统工具

### 核心概念

- **[Pod 生命周期](./pod-lifecycle.md)**：了解 Pod 创建、调度、启动和终止流程，包含详细图示
- **[Pod 启动速度优化](./pod-startup-speed.md)**：加速 Pod 启动的策略，包括镜像优化、调度改进和 GPU 工作负载考量

### 高级主题

#### 调度和资源管理

- **[调度优化](./scheduling-optimization.md)**：全面指南，涵盖高吞吐量调度、多调度器模式、Gang 调度、拓扑感知调度、负载均衡和去调度策略
- **[Swap 内存管理](./swap-memory-management.md)**：kubelet swap 行为（`NoSwap`/`LimitedSwap`）、可观测性和 Kubernetes 内存压力调优实用指南
- **[动态资源分配（DRA）](./dra.md)**：灵活的设备分配，支持结构化参数和拓扑感知
- **[DRA 驱动功能矩阵](./dra-driver-feature-matrix.md)**：公开 DRA 实现生态系统，映射到明确的 Beta/GA 功能信号、进行中的工作和当前公开空白
- **[NVIDIA GPU Operator](./nvidia-gpu-operator.md)**：自动 GPU 驱动安装、设备插件部署、DRA 驱动集成和基于 DCGM 的 GPU 监控
- **[NVIDIA AI Cluster Runtime（AICR）](./nvidia-aicr.md)**：用于生成优化、验证和可复现的 GPU 加速 Kubernetes 集群方案的 CLI 工具，支持 EKS、GKE 和自管理集群
- **[GPU 故障检测与自愈](./gpu-fault-detection.md)**：全面的 GPU 故障检测、诊断和自动恢复指南，使用 DCGM、Node Problem Detector 和渐进式修复策略
- **[节点资源接口（NRI）](./nri.md)**：运行时层面的细粒度容器资源管理
- **[Kubernetes v1.36 AI 基础设施亮点](./v1.36-ai-infra-highlights.md)**：精选 v1.36 版本信号，聚焦 AI 基础设施相关性、升级风险和推荐操作

#### 工作负载隔离

- **[隔离指南](./isolation.md)**：工作负载隔离技术的全面覆盖，包括 cgroups、安全上下文、用户命名空间、基于 VM 的隔离（Kata Containers、gVisor）以及 AI 工作负载的 checkpoint/restore

#### 可扩展性和大规模集群

- **[大规模集群](./large-scale-clusters.md)**：运行大规模 Kubernetes 集群（130,000+ 节点）的技术和架构模式，包括 Consistent Reads from Cache（KEP-2340）、Snapshottable API Server Cache（KEP-4988）、DRANET、Spanner 和 Lustre 分布式文件系统

## 快速参考

### 调度工程师

1. 从[调度优化](./scheduling-optimization.md)开始学习生产模式
2. 了解 [DRA](./dra.md) 和 [NVIDIA GPU Operator](./nvidia-gpu-operator.md) 以管理 GPU 资源
3. 探索 [NRI](./nri.md) 实现细粒度容器控制
4. 理解 [Pod 生命周期](./pod-lifecycle.md)用于调试
5. 在调优内存压力下的 kubelet 行为时使用 [Swap 内存管理](./swap-memory-management.md)

### 平台工程师

1. 从 [Kubernetes 学习计划](./learning-plan.md)开始建立基础知识
2. 研究 [Pod 启动速度](./pod-startup-speed.md)进行性能优化
3. 实施[隔离技术](./isolation.md)确保安全性和多租户

### AI/ML 工程师

1. 专注于[调度优化](./scheduling-optimization.md)优化 GPU 工作负载
2. 了解 [NVIDIA GPU Operator](./nvidia-gpu-operator.md) 进行 GPU 设置和监控
3. 使用 [NVIDIA AICR](./nvidia-aicr.md) 在 EKS、GKE 或自管理集群上获得经过验证、可复现的集群方案
4. 学习 [GPU 故障检测](./gpu-fault-detection.md)确保生产可靠性和自动恢复
5. 学习[隔离指南](./isolation.md)了解 checkpoint/restore
6. 理解 [DRA](./dra.md) 处理复杂设备需求

## 相关主题

- **训练**：参见[训练指南](../training/README.md)了解分布式训练、容错和 ML 流水线
- **推理**：参见[推理指南](../inference/README.md)了解 LLM 服务和优化

## 贡献

欢迎对 Kubernetes 相关文档做出贡献！请确保：

- 技术与官方 Kubernetes 文档一致
- 提供实际示例和生产场景
- 包含相关 KEP 和社区提案的链接
- 提供会议演讲和学习资源

---

**注意**：这是一个学习仓库。部分内容可能由 AI 生成或总结。在生产环境使用前请与官方文档核对。
