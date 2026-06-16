---
status: Active
maintainer: pacoxu
last_updated: 2025-10-29
tags: training, kubeflow, pytorch, tensorflow, ml-operators
canonical_path: zh/docs/training/kubeflow.md
---

# Kubeflow：AI 训练管理

## 概述

<a href="https://github.com/kubeflow/kubeflow">`Kubeflow`</a> 是一个 CNCF Incubating 项目，为 Kubernetes 提供全面的机器学习平台。它简化了 ML 工作流在 Kubernetes 上的部署和管理，使数据科学家和 ML 工程师能够大规模地构建、训练和部署模型。

## Kubeflow 生态系统

Kubeflow 生态系统由几个关键组件组成，旨在覆盖完整的 ML 生命周期：

### 核心组件

- <a href="https://github.com/kubeflow/kubeflow">`Kubeflow`</a>：编排 ML 生命周期的核心平台（CNCF Incubating）
- <a href="https://github.com/kubeflow/pipelines">`Kubeflow Pipelines`</a>：ML 工作流编排和实验管理
- <a href="https://github.com/kubeflow/katib">`Katib`</a>：超参数调优和神经架构搜索（AutoML）
- <a href="https://github.com/kubeflow/notebooks">`Kubeflow Notebooks`</a>：交互式开发环境（Jupyter notebooks）
- <a href="https://github.com/kserve/kserve">`KServe`</a>：模型服务平台（原 KFServing，CNCF Incubating）
- <a href="https://github.com/kubeflow/metadata">`Kubeflow Metadata`</a>：ML 元数据和工件跟踪
- <a href="https://github.com/kubeflow/mpi-operator">`MPI Operator`</a>：MPI 风格的分布式训练支持
- <a href="https://github.com/kubeflow/training-operator">`Training Operator`</a>：用于多种框架的统一训练 Operator

### 集成组件

- <a href="https://argoproj.github.io/">`Argo Workflows`</a>：工作流引擎（Kubeflow Pipelines 的替代方案）
- <a href="https://min.io/">`MinIO`</a>：用于工件和模型的对象存储
- <a href="https://github.com/istio/istio">`Istio`</a>：用于网络和安全的服务网格
- <a href="https://knative.dev/">`Knative`</a>：Serverless 工作负载支持
- <a href="https://www.kubeflow.org/docs/components/central-dash/">`Central Dashboard`</a>：Kubeflow 组件的统一 Web UI
- <a href="https://github.com/dexidp/dex">`Dex`</a>：用于认证的身份服务
- <a href="https://cert-manager.io/">`cert-manager`</a>：用于 TLS 的证书管理

## Kubeflow Training Operator

<a href="https://github.com/kubeflow/training-operator">`Training Operator`</a>（原 TF Operator、PyTorch Operator 等）为跨多种 ML 框架的分布式训练提供 Kubernetes 原生支持。

### 支持的框架

- **PyTorch**（`PyTorchJob`）：分布式 PyTorch 训练，原生支持 DDP、FSDP 和弹性训练
- **TensorFlow**（`TFJob`）：分布式 TensorFlow 训练，支持参数服务器和 all-reduce 策略
- **XGBoost**（`XGBoostJob`）：分布式 XGBoost 训练，用于梯度提升
- **MPI**（`MPIJob`）：基于 MPI 的分布式训练（Horovod、DeepSpeed）
- **PaddlePaddle**（`PaddleJob`）：分布式 PaddlePaddle 训练
- **JAX**（`JAXJob`）：实验性的 JAX 分布式训练支持

### 关键特性

- **Gang Scheduling**：全或无调度，确保所有副本同时启动
- **容错**：自动重启策略和弹性训练支持
- **资源管理**：GPU 和分布式存储集成
- **多租户**：命名空间隔离和资源配额
- **可观测性**：与 Prometheus 和 Kubernetes 指标集成

## Kubeflow Trainer V2

<a href="https://www.kubeflow.org/docs/components/training/trainer-v2/">`Kubeflow Trainer V2`</a> 是下一代训练接口，通过 Python 优先的 API 简化了 Kubernetes 上的分布式训练。

### Trainer V2 的关键特性

基于 <a href="https://mp.weixin.qq.com/s/-PFG2ZHVKZ75vrUrwBZ4PQ">Kubeflow Trainer V2 介绍</a>（中文资源）和 <a href="https://www.kubeflow.org/docs/components/training/trainer-v2/">官方 Trainer V2 文档</a>，主要改进包括：

1. **简化的大规模 AI/ML 任务管理**
   - 抽象了 Kubernetes 的复杂性
   - 聚焦于 ML 工程而非基础设施

2. **Python 原生接口**
   - 用于定义训练任务的 Pythonic API
   - 无需手动编写 YAML 清单
   - 与 PyTorch 和 TensorFlow 代码库的无缝集成

3. **最简单且可扩展的 PyTorch 分布式训练**
   - 内置对 PyTorch DDP、FSDP 和 Torchrun 的支持
   - 自动配置分布式环境变量
   - 支持多节点 GPU 训练

4. **内置 LLM 微调支持**
   - 流行 LLM 的预配置模板
   - 与 HuggingFace Transformers 的集成
   - 支持 PEFT（LoRA、QLoRA）和完整微调

5. **Kubernetes 抽象**
   - 对 ML 从业者隐藏基础设施复杂性
   - 专注于模型训练而非集群管理
   - 自动资源分配和调度

6. **社区集成**
   - 结合了 Kubernetes Batch Working Group 的努力
   - 与 Kubeflow 社区生态系统集成
   - 与现有 Kubeflow 组件兼容

### Trainer V2 能力

下图展示了 Kubeflow Trainer V2 的关键能力：

![Kubeflow Trainer V2 Capabilities](https://github.com/user-attachments/assets/1ccba329-86e3-421b-8c18-62d0d10cff21)

支持的训练模式：

- 多节点训练
- Gang Scheduling
- 弹性训练
- LLM 蓝图
- LLM 微调
- MPI 风格训练

支持的框架和平台：

- PyTorch、JAX、TensorFlow、MLX
- HuggingFace、DeepSpeed、XGBoost
- 多种硬件加速器（NVIDIA、Intel、AMD）
- 云平台（AWS、Azure、Google Cloud、Red Hat OpenShift）
- 本地和自托管部署

### 示例：使用 Trainer V2 进行 PyTorch 分布式训练

```python
from kubeflow.training import TrainingClient
from kubeflow.training import constants

# 初始化训练客户端
client = TrainingClient()

# 定义训练任务
client.create_job(
    name="pytorch-distributed-training",
    namespace="kubeflow",
    base_image="pytorch/pytorch:2.0.0-cuda11.7-cudnn8-runtime",
    num_workers=4,
    num_gpus_per_worker=1,
    command=[
        "python",
        "train.py",
        "--epochs", "100",
        "--batch-size", "32"
    ]
)

# 监控训练任务
client.wait_for_job_completion("pytorch-distributed-training")

# 获取训练日志
logs = client.get_job_logs("pytorch-distributed-training")
print(logs)
```

### 示例：使用 Trainer V2 进行 LLM 微调

```python
from kubeflow.training import TrainingClient

client = TrainingClient()

# 使用 LoRA 微调 LLaMA-2
client.create_job(
    name="llama2-lora-finetuning",
    namespace="kubeflow",
    base_image="huggingface/transformers:latest",
    num_workers=2,
    num_gpus_per_worker=4,
    command=[
        "python",
        "finetune_llama.py",
        "--model_name", "meta-llama/Llama-2-7b-hf",
        "--dataset", "alpaca",
        "--lora_r", "8",
        "--lora_alpha", "16",
        "--epochs", "3"
    ]
)
```

## 架构和工作流

### 训练任务生命周期

1. **任务创建**：用户通过 Python API 或 YAML 提交训练任务
2. **调度**：Training Operator 使用 gang scheduling 创建 pod
3. **初始化**：Pod 初始化分布式训练环境
4. **训练**：在所有 worker 上执行分布式训练
5. **检查点**：定期保存检查点到持久化存储
6. **完成**：任务完成，资源被清理
7. **监控**：收集指标和日志以进行可观测性

### 与 Kubernetes 资源的集成

- **Pods**：每个训练副本在单独的 pod 中运行
- **Services**：主节点和工作节点 pod 之间的通信
- **ConfigMaps**：训练配置和超参数
- **PersistentVolumeClaims**：检查点和数据集存储
- **Secrets**：API 密钥和凭据

## 与 ArgoCD 集成

Kubeflow 训练任务可以使用 ArgoCD 进行部署和管理，实现 GitOps 工作流。详见 [ArgoCD for AI Workloads](./argocd.md)。

## 最佳实践

### 任务配置

- 使用资源请求和限制确保正确调度
- 为分布式任务配置 gang scheduling
- 设置适当的重启策略（OnFailure、Never）
- 使用 init 容器进行数据准备

### 存储管理

- 使用 PersistentVolumes 存储检查点和数据集
- 考虑使用 CSI 驱动实现高性能存储（Fluid、JuiceFS）
- 实施检查点清理策略
- 尽可能本地缓存数据集

### 监控和调试

- 启用 Prometheus 指标进行任务监控
- 使用 Kubeflow UI 查看任务状态和日志
- 配置任务失败告警
- 维护审计日志以符合合规要求

### 安全

- 使用 RBAC 进行访问控制
- 将敏感数据存储在 Kubernetes Secrets 中
- 启用网络策略进行 pod 隔离
- 使用具有最小权限的服务账户

## 学习资源

### 官方文档

- <a href="https://www.kubeflow.org/docs/">Kubeflow Documentation</a>
- <a href="https://www.kubeflow.org/docs/components/training/">Training Operator Documentation</a>
- <a href="https://www.kubeflow.org/docs/components/training/trainer-v2/">Trainer V2 Documentation</a>
- <a href="https://github.com/kubeflow/training-operator/tree/master/examples">Training Operator Examples</a>

### 社区资源

- <a href="https://mp.weixin.qq.com/s/-PFG2ZHVKZ75vrUrwBZ4PQ">Kubeflow Trainer V2 Introduction</a>（中文）
- <a href="https://github.com/kubeflow/community">Kubeflow Community</a>
- <a href="https://www.kubeflow.org/docs/about/community/">Community Meetings and Slack</a>

### 教程和指南

- <a href="https://www.kubeflow.org/docs/started/getting-started/">Getting Started with Kubeflow</a>
- <a href="https://www.kubeflow.org/docs/components/training/pytorch/">PyTorch Training Tutorial</a>
- <a href="https://www.kubeflow.org/docs/components/training/tensorflow/">TensorFlow Training Tutorial</a>

## 路线图和未来发展

### 即将推出的功能

- 增强的弹性训练能力
- 改进与 Ray 的集成以进行分布式计算
- 先进的容错机制
- 对更多 ML 框架的原生支持
- 与云原生存储方案的更好集成

### 社区倡议

- Kubernetes Batch Working Group 协作
- 与 CNCF 项目的集成（Kueue、Volcano）
- 训练任务 API 的标准化
- 大规模训练的性能优化

## 与替代方案的比较

### vs 原生 Kubernetes

- **Kubeflow**：更高层次的抽象、Python API、内置最佳实践
- **原生 K8s**：更灵活，需要更多 YAML 和手动配置

### vs Volcano

- **Kubeflow Training Operator**：框架特定的 Operator，与 Kubeflow 生态系统集成
- **Volcano**：通用批处理调度器，适用于任何工作负载

### vs Ray

- **Kubeflow**：Kubernetes 原生，专注于训练工作流
- **Ray**：分布式计算框架，范围更广（训练、服务、调优）

## 贡献

欢迎对 Kubeflow 和 Training Operator 做出贡献！详见 <a href="https://github.com/kubeflow/training-operator/blob/master/CONTRIBUTING.md">贡献指南</a>。

---

**注意：** 部分内容可能根据引用来源生成或总结。在生产环境中使用前，请以官方文档为准验证技术细节。
