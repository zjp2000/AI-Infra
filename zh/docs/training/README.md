---
status: Active
maintainer: pacoxu
last_updated: 2025-10-31
tags: training, kubernetes, fault-tolerance, distributed-training
canonical_path: zh/docs/training/README.md
---

# 在 Kubernetes 上进行训练

## 概述

在 Kubernetes 上训练大规模 AI 模型会带来独特的挑战，特别是在容错、诊断和保持高有效训练时间比（ETTR）方面。随着训练集群的扩展，故障的频率和影响显著增加，因此强大的故障处理机制至关重要。

具体主题的更多详情：

- [Transformers：标准化跨 PyTorch 生态系统的模型定义](./transformers.md)
- [PyTorch 生态系统与加速器集成（DeepSpeed、vLLM、NPU/HPU/XPU）](./pytorch-ecosystem.md)
- [预训练大语言模型（MoE、DeepseekV3、Llama4）](./pre-training.md)
- [并行策略（数据并行、分片数据并行、上下文并行）](./parallelism.md)
- [Kubeflow Training Operator 与 Trainer V2](./kubeflow.md)
- [ArgoCD 用于 GitOps 工作流](./argocd.md)
- [MLOps：机器学习运维生命周期](./mlops.md)

### 大规模场景的关键挑战

资源规模的扩展带来广泛的故障，对训练稳定性构成重大挑战：

- **故障类型：**
  - CUDA 错误（GPU 硬件/驱动故障）
  - NaN 值（数值不稳定）
  - 任务挂起（通信超时、死锁）
  - 节点故障（硬件问题）

- **故障特征：**
  - **Fail-stop 特性：** 传统方法在出现故障时立即停止训练
  - **高诊断开销：** 确定根本原因需要大量时间和专业知识
  - **频率随集群规模增长：** 更大的集群会遇到更频繁的故障

### 关键指标

- **ETTR（有效训练时间比）：**
  有效训练时间与总任务运行时间的比率。该指标同时反映了故障频率和恢复效率。
  - 公式：`ETTR = 有效训练时间 / 总运行时间`
  - 更高的 ETTR 表示更好的训练效率

- **MFU（模型 FLOPs 利用率）：**
  衡量模型有效利用可用计算资源的能力，与理论峰值性能进行比较。
  - 反映了硬件利用率和软件优化水平

## 行业最佳实践

### 字节跳动训练优化框架

字节跳动开发了一套大规模训练稳定性的综合方法，记录在其技术文章中：
[字节跳动训练优化实践](https://mp.weixin.qq.com/s/-KDhk_R4nj3C-uPG11VoYQ)

其框架的关键组件：

#### 1. 轻量级实时监控

- 对 GPU、网络和训练指标进行持续健康检查
- 在故障发生前早期检测异常
- 低开销监控，最小化对训练性能的影响

#### 2. 停止-诊断模式

- 将故障检测与诊断分离
- 保留系统状态用于事后分析
- 自动化诊断工具以确定根本原因

#### 3. 回放能力

- 记录训练状态和配置
- 支持复现故障以进行调试
- 支持回滚到稳定的检查点

#### 4. 原地热更新

- 在不销毁运行时环境的情况下更新训练代码
- 保留 GPU 内存和已加载数据
- 最小化软件修复的恢复时间

#### 5. 使用热备快速恢复

- 维护备用资源以实现快速故障转移
- 预热环境减少冷启动开销
- 战略资源分配平衡成本和可用性

#### 6. 减少远程文件系统依赖

- 检查点和数据的本地缓存
- 在恢复期间最小化网络 I/O
- 无需等待远程数据加载即可快速重启

#### 7. MFU 优化

- 持续性能分析和优化
- 内核融合和内存优化
- 流水线并行调优

## 项目和工具

### 训练 Operator

- [`Training Operator`](https://github.com/kubeflow/training-operator)：
  面向分布式训练（PyTorch、TensorFlow、XGBoost 等）的 Kubernetes 原生 Operator。Kubeflow 项目的一部分。详见关于 Kubeflow Training Operator 和 Trainer V2 的 [Kubeflow 文档](./kubeflow.md)。
- [`Volcano`](https://github.com/volcano-sh/volcano)：
  CNCF Incubating 项目，提供针对批处理工作负载的 gang scheduling 和队列管理。
- [`Kueue`](https://github.com/kubernetes-sigs/kueue)：
  Kubernetes SIG 项目，提供作业排队和资源配额管理。
- [`KubeRay`](https://github.com/ray-project/kuberay)：
  Ray 集群的 Kubernetes Operator；支持在 Kubernetes 上进行基于 Ray 的分布式训练和服务（Ray Train、Ray Tune）。

### 核心训练框架

- [`PyTorch`](https://github.com/pytorch/pytorch)：分布式训练的主要框架，也是 AI/ML 生态系统的核心。
- [`JAX`](https://github.com/google/jax)：Google 的高性能数值计算库；常用于 TPU 场景和研究。

### 分布式训练库

- [`Megatron-LM`](https://github.com/NVIDIA/Megatron-LM)：
  NVIDIA 的用于训练大语言模型的框架（TP/PP 并行）。
- [`DeepSpeed`](https://github.com/microsoft/DeepSpeed)：
  Microsoft 的深度学习优化库（ZeRO、offload、checkpoint）。
- [`PyTorch FSDP`](https://pytorch.org/docs/stable/fsdp.html)：全分片数据并行，PyTorch 原生参数分片分布式训练。

### 微调

- [`Transformers`](https://github.com/huggingface/transformers)：Hugging Face 库；微调生态系统的核心。
- [`Accelerate`](https://github.com/huggingface/accelerate)：轻量级 Hugging Face 库，以最少的代码修改实现分布式/混合精度训练。
- [`PEFT`](https://github.com/huggingface/peft)：参数高效微调 — LoRA、Adapter 及相关技术。
- [`Axolotl`](https://github.com/axolotl-ai-cloud/axolotl)：面向 SFT/LoRA 微调工作流的意见化工程管道。
- [`LLaMA-Factory`](https://github.com/hiyouga/LLaMA-Factory)：支持 LLaMA 及其他模型的 SFT、LoRA、DPO 等微调方案的工具箱。

### 后训练 / RLHF

- [`verl`](https://github.com/volcengine/verl)：专为 LLM 设计的 RL 训练系统框架（字节跳动/火山引擎）。
- [`SLIME`](https://github.com/TIGER-AI-Lab/SLIME)：面向 LLM 后训练的 RL 扩展框架。
- [`OpenRLHF`](https://github.com/OpenRLHF/OpenRLHF)：面向大语言模型的开源 RLHF 训练框架。
- [`TRL`](https://github.com/huggingface/trl)：Hugging Face 库，支持 PPO、DPO 及其他基于 RL 的训练方案。

### 容错与检查点

- [`PyTorch Distributed Checkpoint`](https://github.com/pytorch/pytorch/tree/main/torch/distributed/checkpoint)：
  PyTorch 中的原生检查点支持，用于容错。
- [`Megatron-LM`](https://github.com/NVIDIA/Megatron-LM)：
  NVIDIA 的用于训练大语言模型的框架，内置容错能力。
- [`DeepSpeed`](https://github.com/microsoft/DeepSpeed)：
  Microsoft 的深度学习优化库，具备检查点和容错功能。

### 监控与可观测性

- [`DCGM Exporter`](https://github.com/NVIDIA/dcgm-exporter)：
  NVIDIA Data Center GPU Manager 的 Prometheus exporter。
- [`Kineto`](https://github.com/pytorch/kineto)：
  PyTorch 性能分析库，用于 GPU 利用率分析。
- [`Metrics Server`](https://github.com/kubernetes-sigs/metrics-server)：
  Kubernetes SIG 项目，用于资源指标。

### 存储与缓存

- [`3FS`](https://github.com/deepseek-ai/3FS)：DeepSeek 的高性能分布式文件系统，针对 AI 训练优化（检查点 I/O 和小文件/元数据密集工作负载）。
- [`safetensors`](https://github.com/huggingface/safetensors)：Hugging Face 的安全快速张量序列化格式，用于模型权重。
- [`TensorStore`](https://github.com/google/tensorstore)：Google 的用于高效读写大型分片数组和检查点的库。
- [`Fluid`](https://github.com/fluid-cloudnative/fluid)：
  CNCF 项目，用于 Kubernetes 中的数据集编排和缓存。
- [`Ceph CSI`](https://github.com/ceph/ceph-csi)：
  Ceph 的 Container Storage Interface 驱动。
- [`JuiceFS CSI`](https://github.com/juicedata/juicefs-csi-driver)：
  用于检查点存储的高性能分布式文件系统。

### GitOps 与持续部署

- [`ArgoCD`](https://github.com/argoproj/argo-cd)：
  CNCF Graduated 项目，用于声明式 GitOps 持续交付。详见关于通过 GitOps 工作流管理训练任务的 [ArgoCD 文档](./argocd.md)。
- [`Argo Workflows`](https://github.com/argoproj/argo-workflows)：
  Kubernetes 原生工作流引擎，用于 ML 管道（CNCF Graduated）。
- [`Flux CD`](https://github.com/fluxcd/flux2)：
  面向 Kubernetes 部署的 GitOps 工具包（CNCF Graduated）。

## 学习主题

### 基础

- **分布式训练基础：**
  - 数据并行 vs 模型并行
  - 流水线并行和张量并行
  - 梯度同步策略（AllReduce、Ring-AllReduce）
  - 通信库：NCCL、Gloo、MPI
  - **详见 [并行策略指南](./parallelism.md) 了解数据并行（DP）、分片数据并行（FSDP）和上下文并行（CP）策略的全面介绍**

- **PyTorch 分布式：**
  - [DistributedDataParallel (DDP)](https://pytorch.org/tutorials/beginner/dist_overview.html)
  - [Fully Sharded Data Parallel (FSDP)](https://pytorch.org/docs/stable/fsdp.html)
  - [RPC 框架](https://pytorch.org/docs/stable/rpc.html)

### Kubernetes 集成

- **Gang Scheduling：**
  - 分布式作业的全或无调度
  - PodGroup 和调度插件
  - 与训练 Operator 的集成

- **资源管理：**
  - GPU 共享和时间分片
  - RDMA 网络配置
  - 动态资源分配（DRA）

- **存储：**
  - 检查点的持久卷
  - CSI 驱动和卷快照
  - 数据局部性优化

### 容错

- **检查点策略：**
  - 完整检查点 vs 增量检查点
  - 检查点频率优化
  - 分布式检查点存储

- **故障检测：**
  - 健康检查和存活探针
  - 死锁检测
  - GPU 错误监控

- **恢复机制：**
  - 自动重启策略
  - 基于检查点的恢复
  - 弹性训练（动态扩缩容）

### 性能优化

- **性能分析：**
  - PyTorch Profiler
  - NVIDIA Nsight Systems
  - 通信瓶颈分析

- **内存优化：**
  - 梯度检查点
  - 混合精度训练（FP16、BF16）
  - 激活重计算

- **通信优化：**
  - 梯度压缩
  - 计算与通信重叠
  - 网络拓扑感知

## 案例研究与论文

### 学术论文

- **容错：**
  - ["Training Large-Scale AI Models with Fault Tolerance"](https://arxiv.org/abs/2509.16293)
    （arXiv:2509.16293v4）

- **分布式训练：**
  - ["PyTorch Distributed: Experiences on Accelerating Data Parallel Training"](https://arxiv.org/abs/1910.02054)

### 行业博文

- **字节跳动：**
  - [大规模训练优化实践](https://mp.weixin.qq.com/s/-KDhk_R4nj3C-uPG11VoYQ)
    （中文）

- **Meta：**
  - [Training Large Language Models at Scale](https://engineering.fb.com/2021/09/13/ml-applications/llama/)

- **Microsoft：**
  - [DeepSpeed: Extreme-scale model training for everyone](https://www.microsoft.com/en-us/research/blog/deepspeed-extreme-scale-model-training-for-everyone/)

## 路线图（进行中的提案）

### Kubernetes 增强

- **弹性训练 KEP：**
  基于资源可用性动态扩缩容训练任务
- **检查点优化：**
  Kubernetes 原生支持分布式检查点
- **GPU 健康监控：**
  内置 GPU 错误检测和节点隔离

### Training Operator 增强

- [Training Operator Issues](https://github.com/kubeflow/training-operator/issues)：
  功能请求和增强
- **自动故障恢复：**
  具有智能重试逻辑的自愈训练任务
- **跨集群训练：**
  跨多个 Kubernetes 集群的联邦训练

## 快速开始

### 先决条件

- 具有 GPU 节点的 Kubernetes 集群
- 已安装 Training Operator
- 已配置持久化存储（用于检查点）
- 监控栈（Prometheus + Grafana）

### 快速入门示例

```yaml
# PyTorch 分布式训练任务
apiVersion: kubeflow.org/v1
kind: PyTorchJob
metadata:
  name: pytorch-distributed-training
spec:
  pytorchReplicaSpecs:
    Master:
      replicas: 1
      restartPolicy: OnFailure
      template:
        spec:
          containers:
          - name: pytorch
            image: pytorch/pytorch:latest
            resources:
              limits:
                nvidia.com/gpu: 1
    Worker:
      replicas: 3
      restartPolicy: OnFailure
      template:
        spec:
          containers:
          - name: pytorch
            image: pytorch/pytorch:latest
            resources:
              limits:
                nvidia.com/gpu: 1
```

## 最佳实践

### 检查点管理

- 频繁保存检查点（例如，每 N 步或每 M 分钟）
- 将检查点存储在可靠的持久化存储上
- 使用异步检查点以最小化训练中断
- 实现检查点版本控制和清理策略

### 监控与告警

- 监控 GPU 利用率、温度和错误
- 跟踪训练指标（loss、吞吐量、MFU）
- 设置异常和故障告警
- 维护训练日志用于事后分析

### 资源规划

- 预留 10-20% 的额外容量用于容错
- 考虑使用竞价/可抢占实例并配合适当的检查点策略
- 规划检查点写入期间的峰值资源使用
- 使用资源配额防止资源耗尽

### 测试

- 在预发布环境中测试故障场景
- 定期验证检查点/恢复流程
- 基准测试恢复时间目标（RTO）
- 验证恢复后的训练正确性

## 贡献

欢迎对训练相关文档和最佳实践做出贡献！请通过提交 issue 或 pull request 分享您的经验、工具和优化技术。

---

**注意：** 部分内容可能根据引用来源生成或总结。在生产环境中使用前，请以官方文档为准验证技术细节。
