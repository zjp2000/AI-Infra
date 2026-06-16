---
status: Active
maintainer: pacoxu
last_updated: 2026-01-13
tags: mlops, machine-learning, model-lifecycle, kubernetes
canonical_path: zh/docs/training/mlops.md
---

# MLOps：机器学习运维

## 概述

MLOps（Machine Learning Operations）将模型生命周期转变为可重复、可审计、可回滚的工程化闭环。它将机器学习开发实践与 DevOps 原则相结合，以简化从数据准备到生产部署和监控的端到端 ML 工作流。

**核心目标：**

- **可重复性**：一致且可复现的 ML 工作流
- **可审计性**：模型血统的全面可见性和跟踪
- **可回滚**：安全部署，能够回退到以前的版本
- **自动化**：减少人工干预和人为错误
- **协作**：实现数据科学家和工程师之间的无缝团队协作

## MLOps 架构

MLOps 生态系统可以组织为七个关键功能领域：

### 1) 端到端平台

为整个 ML 生命周期提供集成工具链的完整平台，旨在作为"参考平台"，具有强大的 Kubernetes 原生集成。

**关键平台：**

- <a href="https://github.com/kubeflow/kubeflow">`Kubeflow`</a>：面向 Kubernetes 的 AI 平台工具包，涵盖组合性、模块化和可移植性，从实验到生产的多个阶段。CNCF 项目。

**使用场景：**

- 希望标准化 Kubernetes 原生工作流的组织
- 需要端到端可见性和组件互操作性的团队

### 2) 流水线与编排

用于构建、管理和调度具有依赖关系和并行执行的复杂 ML 管道的工作流编排系统。

**关键项目：**

- <a href="https://github.com/kubeflow/pipelines">`Kubeflow Pipelines (KFP)`</a>：面向在 Kubernetes 上构建和部署可移植、可扩展 ML 工作流的容器原生工作流编排。Kubeflow 的一部分 +1
- <a href="https://github.com/argoproj/argo-workflows">`Argo Workflows`</a>：用于在 CRD 为基础的 DAG 或步骤格式中定义工作流的 Kubernetes 原生工作流引擎。CNCF Graduated +1
- <a href="https://github.com/flyteorg/flyte">`Flyte`</a>：可扩展的数据/ML/分析工作流编排平台，具有强类型安全和可复现性。CNCF Incubating +1

**使用场景：**

- Kubernetes 原生环境：Argo / KFP
- 具有强类型要求和可复现性的复杂工作流：Flyte

### 3) 实验跟踪与模型注册

集中式跟踪实验、超参数、指标和模型工件，带有版本控制和血统跟踪。

**关键项目：**

- <a href="https://github.com/mlflow/mlflow">`MLflow`</a>：涵盖实验跟踪和模型生命周期管理的开源平台，Model Registry 提供集中式版本控制和元数据管理。GitHub +1

**使用场景：**

- 需要跟踪实验并为工程团队发布模型的团队，具有标准化的注册中心/血统视图。

### 4) 数据与模型版本管理

用于管理数据和模型工件的 Git 式版本控制系统，确保可复现性并支持回滚。

**关键项目：**

- <a href="https://github.com/iterative/dvc">`DVC`</a>：数据和 ML 实验管理工具，为数据和模型提供类似 Git 的版本控制，使"代码版本"和"数据/模型版本"可以同步跟踪。DVC +1

**使用场景：**

- 大型数据集的训练数据版本管理
- 跨迭代的模型工件版本管理
- 需要严格可复现性和回滚能力的团队

### 5) 数据质量与契约

用于验证数据模式、质量指标和预期的框架，防止数据漂移并确保模型可靠性。

**关键项目：**

- <a href="https://github.com/great-expectations/great_expectations">`Great Expectations`</a>：用于数据质量单元测试的数据验证和文档框架（"期望"作为数据质量断言）。greatexpectations.io +1

**使用场景：**

- 经历过"数据漂移/垃圾数据"问题并希望将数据验证集成到 ML 管道中的团队。

### 6) 特征平台

用于定义、管理、发现和服务特征的开源特征工程平台，在训练和在线推理之间保持一致，确保训练-服务一致性。

**关键项目：**

- <a href="https://github.com/feast-dev/feast">`Feast`</a>：用于管理、发现和服务 ML 特征的特征存储，支持训练-服务一致性。Feast +1

**使用场景：**

- 传统 ML（推荐系统、广告、风险建模）
- RAG/LLM 场景，需要大规模结构化特征在线服务

### 7) 部署与推理服务

模型部署和服务平台，提供具有自动缩放、监控、金丝雀部署和健康检查的推理服务。

**关键项目：**

- <a href="https://github.com/kserve/kserve">`KServe`</a>：Kubernetes 原生模型推理平台（CRD），提供预训练模型服务的打包、自动缩放、网络、健康检查和配置管理。KServe Documentation +1
- <a href="https://github.com/SeldonIO/seldon-core">`Seldon Core`</a>：基于 Kubernetes 的 MLOps/LLMOps 框架，用于打包、部署、监控和管理数千个生产 ML 模型。GitHub +1
- <a href="https://github.com/bentoml/BentoML">`BentoML`</a>：基于 Python 的模型/AI 应用服务框架，具有快速模型打包、容器化和边缘服务能力。GitHub +1
- <a href="https://github.com/ray-project/ray/tree/master/python/ray/serve">`Ray Serve`</a>：可扩展、可编程的服务框架（框架无关），支持动态批处理、多模型/多 GPU 服务等。docs.ray.io

**推理引擎集成：**

有关 LLM 推理服务的深入介绍，请参阅 [Inference Guide](../inference/README.md)：

- <a href="https://github.com/nvidia/TensorRT-LLM">`NVIDIA Triton Inference Server`</a>：GPU 优化的多框架服务，支持多个后端（TensorRT、ONNX、PyTorch）。GitHub +1

**使用场景：**

- KServe：基于 CRD 管理的 Kubernetes 原生模型服务
- Seldon Core：面向大规模生产模型部署的 MLOps 框架
- BentoML：快速原型设计和边缘部署
- Ray Serve：灵活、可编程的服务，具有高级批处理能力
- Triton：多框架 GPU 优化服务

## 综合 MLOps 工作流项目

跨越多个 MLOps 类别的端到端平台：

- <a href="https://github.com/kubeflow/kubeflow">`Kubeflow`</a>：面向 Kubernetes 的机器学习工具包，涵盖管道、训练、服务等。参见 [Kubeflow Training Guide](./kubeflow.md)。
- <a href="https://github.com/flyteorg/flyte">`Flyte`</a>：可扩展且灵活的工作流编排平台，无缝统一数据、ML 和分析栈。CNCF Incubating。
- <a href="https://github.com/Netflix/metaflow">`Metaflow`</a>：构建、部署和管理 AI/ML 系统。最初由 Netflix 开发。
- <a href="https://github.com/mlflow/mlflow">`MLflow`</a>：涵盖跟踪、项目、模型和模型注册的机器学习生命周期开源平台。
- <a href="https://github.com/polyaxon/polyaxon">`Polyaxon`</a>：用于管理和编排机器学习生命周期的 MLOps 工具。
- <a href="https://github.com/ray-project/ray">`Ray`</a>：AI 计算引擎。包含核心分布式运行时和一组用于加速 ML 工作负载的 AI 库。
- <a href="https://github.com/zenml-io/zenml">`ZenML`</a>：连接 ML 和 Ops 的桥梁。可扩展的 MLOps 框架，专注于可复现性。

## 学习主题

### 基础

- **ML 生命周期基础：**
  - 模型开发和实验
  - 模型训练和超参数调优
  - 模型评估和验证
  - 模型部署和服务
  - 模型监控和重新训练

- **版本控制：**
  - 代码版本控制（Git）
  - 数据版本控制（DVC、Git LFS）
  - 模型版本控制（MLflow、DVC）
  - 实验跟踪和可复现性

### Kubernetes 集成

- **Kubernetes 上的模型服务：**
  - KServe 用于标准化模型服务
  - 基于推理负载的自动缩放
  - 金丝雀部署和 A/B 测试
  - GPU/CPU 工作负载的资源管理

- **训练编排：**
  - Kubernetes 上的分布式训练
  - 多节点任务的 Gang Scheduling
  - 与训练 Operator 的集成（Kubeflow、Volcano）
  - 参见 [Training Guide](./README.md) 了解全面介绍。

- **存储和工件管理：**
  - 模型存储的持久卷
  - 对象存储集成（S3、GCS、Azure Blob）
  - 数据集的缓存策略
  - 参见 [Fluid](https://github.com/fluid-cloudnative/fluid) 了解数据编排。

### 自动化和 CI/CD

- **用于 ML 的 GitOps：**
  - ArgoCD 用于声明式部署
  - 自动重新训练管道
  - 模型版本控制和回滚策略
  - 参见 [ArgoCD Guide](./argocd.md)。

- **管道编排：**
  - 基于 DAG 的工作流（Argo Workflows、KFP）
  - 依赖管理和调度
  - 并行执行和资源优化
  - 错误处理和重试逻辑

### 监控和可观测性

- **模型监控：**
  - 预测准确性和漂移检测
  - 特征分布监控
  - 模型性能指标（延迟、吞吐量）
  - 参见 [Observability Guide](../observability/README.md)。

- **数据质量：**
  - 模式验证和约束
  - 数据质量测试（Great Expectations）
  - 数据漂移检测
  - 输入特征的异常检测

## 最佳实践

### 模型生命周期管理

- **对一切进行版本控制：**
  - 一起跟踪代码、数据和模型工件
  - 为模型使用语义化版本
  - 记录模型血统和依赖关系
  - 维护可复现的实验记录

- **自动化测试：**
  - 数据预处理的单元测试
  - 模型管道的集成测试
  - 模型性能基准测试
  - 生产部署前的影子测试

### 部署策略

- **渐进式发布：**
  - 金丝雀部署（1% → 10% → 50% → 100%）
  - 用于模型比较的 A/B 测试
  - 零停机更新的蓝绿部署
  - 用于受控发布的特性标志

- **回滚能力：**
  - 维护以前的模型版本
  - 快速回滚程序
  - 性能下降时的自动回滚
  - 记录回滚条件和程序

### 监控和告警

- **需要监控的关键指标：**
  - 模型推理延迟（p50、p95、p99）
  - 预测准确性和错误率
  - 资源利用率（CPU、GPU、内存）
  - 数据漂移和特征分布

- **告警策略：**
  - 设置模型性能下降告警
  - 监控数据质量问题
  - 跟踪资源耗尽情况
  - 为生产模型配置值班轮换

### 协作和文档

- **团队协作：**
  - 共享实验跟踪（MLflow）
  - 模型变更的代码审查
  - 模型决策的文档
  - 知识共享和事后分析

- **模型文档：**
  - 描述目的和限制的模型卡片
  - 训练数据特征和偏差
  - 性能指标和基准测试
  - 部署要求和依赖项

## 案例研究和参考

### 行业实施

- **Netflix：**
  - [Metaflow: Production ML Infrastructure](https://netflixtechblog.com/open-sourcing-metaflow-a-human-centric-framework-for-data-science-fa72e04a5d9)
  - 关注数据科学家生产力和可复现性

- **Uber：**
  - [Michelangelo: Uber's ML Platform](https://www.uber.com/blog/michelangelo-machine-learning-platform/)
  - 用于训练、服务和监控的端到端 ML 平台

- **Airbnb：**
  - [Bighead: Airbnb's ML Platform](https://databricks.com/session/bighead-airbnbs-end-to-end-machine-learning-platform)
  - 大规模特征工程和模型管理

### 研究论文

- **MLOps 原则：**
  - ["Hidden Technical Debt in Machine Learning Systems"](https://papers.nips.cc/paper/2015/hash/86df7dcfd896fcaf2674f757a2463eba-Abstract.html)（NIPS 2015）
  - ["Machine Learning: The High-Interest Credit Card of Technical Debt"](https://research.google/pubs/pub43146/)（Google Research）

- **模型服务：**
  - ["Clipper: A Low-Latency Online Prediction Serving System"](https://arxiv.org/abs/1612.03079)
  - ["TensorFlow Serving: Flexible, High-Performance ML Serving"](https://arxiv.org/abs/1712.06139)

## 路线图（进行中的倡议）

### Kubernetes 增强

- **模型作为一等公民：**
  - 模型的 Kubernetes 原生资源
  - 内置模型版本控制和血统
  - 集成的模型生命周期管理

- **ModelMesh 集成：**
  - 多模型服务优化
  - 跨模型的资源池化
  - 动态模型加载和卸载

### MLOps 标准化

- **CNCF MLOps 项目：**
  - [Model Spec](https://github.com/modelpack/model-spec)：标准化的模型打包（CNCF Sandbox）
  - [KServe Evolution](https://github.com/kserve/kserve/issues)：推理标准的持续发展

- **行业标准：**
  - ONNX 用于模型互换
  - OpenTelemetry 用于 ML 可观测性
  - OpenMetrics 用于 ML 指标

### 新兴趋势

- **LLMOps：**
  - 针对大语言模型的专门 MLOps 实践
  - 提示工程和版本控制
  - LLM 特定的监控和评估
  - 参见 [Inference Guide](../inference/README.md) 了解 LLM 服务。

- **联邦 MLOps：**
  - 跨集群模型训练
  - 隐私保护的 ML 工作流
  - 边缘部署和联邦学习

## 快速开始

### 先决条件

- Kubernetes 集群（用于部署）
- Python 环境（3.8+）
- 容器注册表访问
- 对象存储（S3、GCS 或 Azure Blob）

### 快速开始：MLflow 跟踪示例

```python
import mlflow

# 启动 MLflow 跟踪
mlflow.set_tracking_uri("http://mlflow-server:5000")
mlflow.set_experiment("my-experiment")

# 记录参数和指标
with mlflow.start_run():
    mlflow.log_param("learning_rate", 0.01)
    mlflow.log_param("batch_size", 32)

    # 训练模型
    model = train_model()

    # 记录指标
    mlflow.log_metric("accuracy", 0.95)
    mlflow.log_metric("loss", 0.05)

    # 记录模型
    mlflow.sklearn.log_model(model, "model")
```

### 快速开始：Kubeflow Pipeline 示例

```python
from kfp import dsl
from kfp import compiler

@dsl.component
def preprocess_data(input_path: str, output_path: str):
    # 数据预处理逻辑
    pass

@dsl.component
def train_model(data_path: str, model_path: str):
    # 模型训练逻辑
    pass

@dsl.pipeline(name='ml-pipeline')
def ml_pipeline(input_data: str):
    preprocess_task = preprocess_data(
        input_path=input_data,
        output_path='/data/preprocessed'
    )

    train_task = train_model(
        data_path=preprocess_task.outputs['output_path'],
        model_path='/models/trained'
    )

# 编译并提交管道
compiler.Compiler().compile(ml_pipeline, 'pipeline.yaml')
```

## 贡献

欢迎对 MLOps 文档和最佳实践做出贡献！请通过提交 issue 或 pull request 分享您的经验、工具和优化技术。

## 相关文档

- [Training on Kubernetes](./README.md)：分布式训练和容错
- [Kubeflow Training](./kubeflow.md)：Kubeflow Training Operator 和 Trainer V2
- [ArgoCD for GitOps](./argocd.md)：用于 ML 部署的 GitOps 工作流
- [Inference Guide](../inference/README.md)：LLM 推理和服务
- [Observability](../observability/README.md)：ML 工作负载的监控和指标

---

**注意：** 部分内容基于社区资源生成和组织。在生产环境中使用前，请以官方文档为准验证技术细节。
