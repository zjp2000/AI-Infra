---
status: Active
maintainer: pacoxu
last_updated: 2025-11-17
tags: serverless, inference, knative, sagemaker, ai-platforms
canonical_path: zh/docs/inference/serverless.md
---

# AI 推理无服务器支持

本文档概述了无服务器 AI 推理平台，重点关注 Kubernetes 原生解决方案和云提供商产品，用于部署 LLM 和 ML 模型，具有自动缩放、按用量付费和最小运维开销的特点。

## 目录

- [概述](#overview)
- [无服务器平台对比](#serverless-platform-comparison)
- [Kubernetes 原生解决方案](#kubernetes-native-solutions)
  - [KNative AI](#knative-ai)
- [云提供商解决方案](#cloud-provider-solutions)
  - [AWS SageMaker Serverless Inference](#aws-sagemaker-serverless-inference)
- [平台对比矩阵](#platform-comparison-matrix)
- [关键考虑因素](#key-considerations)
- [参考资料](#references)

## 概述

无服务器 AI 推理平台使组织可以部署机器学习模型而无需管理底层基础设施。这些平台根据流量自动缩放，仅按实际使用量收费，并高效处理冷启动。关键优势包括：

- **自动缩放**：基于需求自动缩放到零和扩展
- **成本优化**：按请求或按计算付费的定价模式
- **减少运维**：无需基础设施管理
- **快速部署**：快速模型部署和更新
- **多模型支持**：在共享基础设施上托管多个模型

## 无服务器平台对比

基于行业分析和平台能力，以下是领先的无服务器 AI 推理平台的全面对比：

| 平台 | 冷启动 | 最大执行时间 | 定价模式 | 最佳用例 |
|----------|-----------|---------------|---------------|---------------|
| Cyfuture AI | 3-7s | 无限 | 企业按小时 | 企业部署 |
| AWS SageMaker | 1-10s | 15 min | 按请求付费 | 企业 AWS 集成 |
| Google Vertex AI | 2-8s | 60 min | 按小时计算 | AutoML 工作流 |
| Azure ML | 2-8s | 60 min | 按用量计费 | Microsoft 生态系统 |
| Hugging Face | 30-120s | 无限 | GPU 按小时 | NLP 应用 |
| Replicate | 10-30s | 无限 | 按秒计费 | 开源模型 |
| Modal | <5s | 无限 | GPU 按小时 | 高性能 ML |
| Potassium | <1s | 无限 | 基于实例 | 生产推理 |
| RunPod | 15-45s | 无限 | 按秒 GPU | 成本优化 |
| OctoAI | 5-15s | 无限 | 基于性能 | 模型优化 |

**关键指标：**

- **冷启动**：初始化和服务第一个请求的时间
- **最大执行时间**：每次请求的最大推理时间
- **定价模式**：平台如何收费
- **最佳用例**：平台表现出色的主要场景

## Kubernetes 原生解决方案

### KNative AI

<a href="https://github.com/knative/serving">`Knative Serving`</a> 提供了一个 Kubernetes 原生的无服务器平台，可用于 AI 推理工作负载。结合 AI 特定扩展和模型服务框架，Knative 支持：

**关键特性：**

- **Scale-to-zero**：空闲时自动缩放到零副本
- **自动缩放**：基于请求和并发的自动缩放
- **版本管理**：流量分割和金丝雀部署
- **请求缓冲**：在冷启动期间排队请求
- **集成**：与 KServe、Seldon Core 和自定义服务运行时配合使用

**架构：**

Knative AI 实现通常组合：

1. **Knative Serving**：无服务器运行时和自动缩放
2. **Model Server**：vLLM、TGI、Triton 或自定义推理引擎
3. **KServe/自定义 CRD**：模型生命周期和版本管理
4. **Autoscaler**：KPA（Knative Pod Autoscaler）用于智能缩放

**示例用例：**

```yaml
# LLM 推理的 Knative Service 示例
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: llama-inference
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/target: "10"
        autoscaling.knative.dev/metric: "concurrency"
    spec:
      containers:
      - image: vllm/vllm-openai:latest
        env:
        - name: MODEL_NAME
          value: "meta-llama/Llama-3.1-8B"
        resources:
          limits:
            nvidia.com/gpu: "1"
```

**集成示例：**

- <a href="https://github.com/knative/docs/blob/main/docs/blog/articles/ai_functions_llama_stack.md">`Llama Stack with Knative`</a>：Llama 模型的无服务器部署
- <a href="https://github.com/kserve/kserve">`KServe`</a>：具有 Knative 集成的 Kubernetes 原生模型服务
- <a href="https://github.com/SeldonIO/seldon-core">`Seldon Core`</a>：具有 Knative 自动缩放的 ML 部署

**冷启动优化：**

- 具有最小缩放设置的预预热副本
- 模型缓存和镜像优化
- GPU Pod 启动优化（参见 [GPU Pod 冷启动](../kubernetes/gpu-pod-cold-start.md)）
- 快速模型加载策略

**学习资源：**

- <a href="https://knative.dev/docs/">`Knative 文档`</a>
- <a href="https://kserve.github.io/website/">`KServe 文档`</a>
- <a href="https://github.com/knative/community/blob/main/working-groups/WORKING-GROUPS.md">`Knative 工作组`</a>

### KEDA——LLM 推理的事件驱动自动缩放

<a href="https://keda.sh">`KEDA（Kubernetes Event-Driven Autoscaling）`</a> 是一个 CNCF 毕业项目，扩展了 Kubernetes HPA，基于外部事件源和自定义指标进行缩放——这对于 LLM 推理至关重要，因为 CPU/内存指标是差的缩放信号。

**为什么选择 KEDA 用于 LLM 推理：**

标准 HPA 基于 CPU 或内存缩放，但推理工作负载受限于：

- GPU 饱和度和请求队列深度
- 负载下 TTFT（首 token 时间）退化
- vLLM `num_requests_waiting` 指标超过阈值

**推理的关键缩放器：**

| 缩放器 | 用例 |
| --- | --- |
| Prometheus | 基于 vLLM/SGLang 队列深度或 TTFT 指标缩放 |
| Kafka / RabbitMQ | 基于批量推理队列深度缩放 |
| Redis | 基于 Redis 流中请求积压缩放 |
| AWS SQS / GCP PubSub | 云原生基于队列的缩放 |

**示例：基于 vLLM 请求队列深度缩放**

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: vllm-scaledobject
spec:
  scaleTargetRef:
    name: vllm-deployment
  minReplicaCount: 1
  maxReplicaCount: 8
  cooldownPeriod: 120
  triggers:
  - type: prometheus
    metadata:
      serverAddress: http://prometheus:9090
      metricName: vllm_requests_waiting
      query: sum(vllm:num_requests_waiting{namespace="default"})
      threshold: "5"
```

**Scale-to-Zero 考虑因素：**

LLM 推理冷启动（模型加载）可能需要 30–300 秒。缓解策略：

- 设置 `minReplicaCount: 1` 以避免对延迟敏感路径完全缩放到零
- 对延迟关键型路由使用 **warm pool 模式**（预初始化 Pod）
- 在节点本地存储（NVMe）上缓存模型权重以加快冷启动
- 参见 [GPU Pod 冷启动](../kubernetes/gpu-pod-cold-start.md) 了解详细策略

**与 Gateway API Inference Extension 集成：**

<a href="https://github.com/kubernetes-sigs/gateway-api-inference-extension">`Gateway API Inference Extension`</a> 公开了每个模型的队列深度和 TTFT 指标，KEDA 可以直接使用，实现推理感知自动缩放：

```yaml
# 基于每个模型 TTFT 退化缩放
triggers:
- type: prometheus
  metadata:
    query: |
      histogram_quantile(0.95,
        rate(inference_model_request_duration_seconds_bucket[2m]))
    threshold: "0.5"  # P95 TTFT > 500ms 时扩展
```

**参考资料：**

- <a href="https://keda.sh/docs/">`KEDA 文档`</a>
- <a href="https://github.com/kedacore/keda">`KEDA GitHub`</a>（CNCF 毕业）
- <a href="https://github.com/kubernetes-sigs/gateway-api-inference-extension">`Gateway API Inference Extension`</a>

## 云提供商解决方案

### AWS SageMaker Serverless Inference

<a href="https://docs.aws.amazon.com/sagemaker/latest/dg/serverless-endpoints.html">`AWS SageMaker Serverless Inference`</a> 是一个专为部署和扩展 ML 模型而构建的无服务器选项，无需管理基础设施。

**关键特性：**

- **自动缩放**：从零扩展到数千个请求
- **按用量付费**：仅按计算时间和处理的数据收费
- **内存配置**：选择 1GB 到 6GB 的内存
- **并发调用**：处理最多 200 个并发请求
- **托管基础设施**：无需服务器配置或管理
- **集成**：原生 AWS 生态系统集成（Lambda、S3、IAM）

**规格：**

- **冷启动**：1-10 秒，取决于模型大小和内存
- **最大调用时间**：每次请求 15 分钟
- **最大负载**：请求/响应 6MB
- **内存选项**：1GB、2GB、3GB、4GB、5GB、6GB
- **并发**：每个端点最多 200 个并发调用

**定价模式：**

- **计算**：按推理时长付费（每毫秒）
- **内存**：按分配的内存付费（每 GB-秒）
- **数据**：同区域数据传输不收费
- **无最低消费**：无最低费用或预付款

**示例配置：**

```python
# 创建无服务器端点配置
serverless_config = {
    'MemorySizeInMB': 4096,  # 4GB 内存
    'MaxConcurrency': 50,     # 最大并发调用数
}

# 使用无服务器配置创建端点
response = sagemaker_client.create_endpoint(
    EndpointName='llama-serverless',
    EndpointConfigName='llama-config',
    ServerlessEndpointConfig=serverless_config
)
```

**最佳实践：**

1. **模型优化**：
   - 使用模型压缩和量化
   - 优化容器大小和依赖项
   - 启用模型制品缓存

2. **减少冷启动**：
   - 尽可能保持模型在 1GB 以下
   - 使用较小的内存配置
   - 考虑定时调用以保持热状态

3. **并发规划**：
   - 根据预期流量设置 MaxConcurrency
   - 监控冷启动频率
   - 对可预测的工作负载使用预置并发

4. **成本优化**：
   - 合理调整内存分配
   - 对大型作业使用批量推理
   - 监控空闲时间并调整超时设置

**用例：**

- **间歇性流量**：不可预测或零星使用的应用
- **开发/测试**：高性价比的模型测试和验证
- **低延迟要求**：小模型的亚秒级推理
- **多模型托管**：无需过度配置即可部署多个模型

**限制：**

- 最大调用时间 15 分钟（不适合长时间运行的推理）
- 6MB 负载限制（不适合非常大的输入/输出）
- 冷启动可能影响延迟敏感型应用
- 有限的 GPU 支持（主要为基于 CPU 的推理）

**迁移路径：**

对于超出无服务器限制的工作负载：

- <a href="https://docs.aws.amazon.com/sagemaker/latest/dg/realtime-endpoints.html">`SageMaker 实时端点`</a>：适用于一致流量模式
- <a href="https://docs.aws.amazon.com/sagemaker/latest/dg/async-inference.html">`SageMaker 异步推理`</a>：适用于长时间运行或大负载推理
- <a href="https://docs.aws.amazon.com/sagemaker/latest/dg/batch-transform.html">`SageMaker 批量转换`</a>：适用于离线批处理

**学习资源：**

- <a href="https://aws.amazon.com/sagemaker/serverless-inference/">`AWS SageMaker 无服务器概述`</a>
- <a href="https://docs.aws.amazon.com/sagemaker/latest/dg/serverless-endpoints-how-it-works.html">`工作原理文档`</a>
- <a href="https://github.com/aws/amazon-sagemaker-examples/tree/main/serverless-inference">`示例笔记本`</a>

## 平台对比矩阵

### 选择标准

选择无服务器 AI 推理平台时，请考虑：

1. **冷启动要求**：
   - 亚秒级：Potassium、Modal
   - 1-10 秒：AWS SageMaker、Knative、Google Vertex AI、Azure ML
   - 10 秒以上：Hugging Face、Replicate、RunPod、OctoAI

2. **执行时间限制**：
   - 无限：Knative、Hugging Face、Replicate、Modal、Potassium、RunPod、OctoAI
   - 15 分钟：AWS SageMaker
   - 60 分钟：Google Vertex AI、Azure ML

3. **定价模式偏好**：
   - 按请求付费：AWS SageMaker
   - 按秒计费：Replicate、RunPod
   - 按小时 GPU：Hugging Face、Modal
   - 企业：Cyfuture AI
   - 按用量计费：Azure ML、Google Vertex AI

4. **生态系统集成**：
   - AWS 生态系统：SageMaker
   - Kubernetes 原生：Knative、KServe
   - Microsoft 生态系统：Azure ML
   - Google Cloud：Vertex AI
   - 开源模型：Hugging Face、Replicate

5. **GPU 支持**：
   - 强 GPU：Hugging Face、Replicate、Modal、RunPod、OctoAI
   - 有限 GPU：AWS SageMaker Serverless（主要为 CPU）
   - 灵活：Knative（取决于集群配置）

### 决策矩阵

| 需求 | 推荐平台 |
|-------------|---------------------|
| 企业 AWS 工作负载 | AWS SageMaker Serverless |
| Kubernetes 原生部署 | Knative + KServe |
| 最低冷启动（<1s） | Potassium、Modal |
| 成本优化 | RunPod、Replicate |
| NLP/Transformer 模型 | Hugging Face |
| 生产推理 | Potassium、Cyfuture AI |
| AutoML 工作流 | Google Vertex AI |
| Microsoft 生态系统 | Azure ML |
| 高性能 ML | Modal、OctoAI |
| 开源灵活性 | Knative、Replicate |

## 关键考虑因素

### 性能优化

1. **模型大小**：小模型冷启动更快
2. **容器优化**：最小化镜像大小和依赖项
3. **预热策略**：实施保活或定时 ping
4. **缓存**：对 LLM 使用模型制品缓存和 KV 缓存
5. **批处理**：尽可能对请求进行分组以分摊启动成本

### 成本管理

1. **合理调整**：匹配内存/计算与实际模型需求
2. **使用模式**：无服务器最适合可变/不可预测的流量
3. **监控**：跟踪冷启动频率和空闲时间
4. **混合方法**：将无服务器与专用实例结合用于基准负载

### 运维卓越

1. **监控**：跟踪延迟、冷启动、错误和成本
2. **日志记录**：实施全面的请求/响应日志记录
3. **版本管理**：使用版本管理实现安全发布
4. **测试**：进行负载测试以了解缩放行为
5. **SLA**：设置适当的超时和并发限制

### 安全性

1. **访问控制**：实施适当的认证和授权
2. **网络隔离**：使用 VPC 和私有端点（如可用）
3. **数据保护**：加密传输中和静态数据
4. **合规性**：确保平台满足监管要求
5. **密钥管理**：使用平台原生的密钥存储

## 参考资料

### 平台

- <a href="https://knative.dev/">`Knative`</a>：基于 Kubernetes 的无服务器平台
- <a href="https://aws.amazon.com/sagemaker/serverless-inference/">`AWS SageMaker Serverless`</a>：AWS 托管无服务器推理
- <a href="https://cloud.google.com/vertex-ai">`Google Vertex AI`</a>：Google Cloud AI 平台
- <a href="https://azure.microsoft.com/en-us/products/machine-learning">`Azure ML`</a>：Microsoft Azure ML 平台
- <a href="https://huggingface.co/inference-endpoints">`Hugging Face Inference Endpoints`</a>：托管 Transformer 模型服务
- <a href="https://replicate.com/">`Replicate`</a>：在云端运行开源模型
- <a href="https://modal.com/">`Modal`</a>：数据和 ML 的无服务器云
- <a href="https://www.banana.dev/">`Potassium (Banana)`</a>：生产 ML 推理
- <a href="https://www.runpod.io/">`RunPod`</a>：GPU 云计算
- <a href="https://octo.ai/">`OctoAI`</a>：优化模型服务

### 相关文档

- [模型生命周期管理](./model-lifecycle.md)：冷启动优化策略
- [GPU Pod 冷启动](../kubernetes/gpu-pod-cold-start.md)：GPU 初始化优化
- [缓存策略](./caching.md)：KV 缓存和模型缓存技术
- [性能测试](./performance-testing.md)：无服务器平台基准测试

### 社区资源

- <a href="https://github.com/kubernetes/community/blob/master/wg-serving/README.md">`Kubernetes WG Serving`</a>
- <a href="https://kserve.github.io/website/">`KServe 社区`</a>
- <a href="https://github.com/knative/community">`Knative 社区`</a>

---

**注意**：平台规格和定价可能会发生变化。在做出部署决策之前，请始终通过官方平台文档验证当前能力和成本。冷启动时间和性能指标可能因模型大小、配置和流量模式而异。
