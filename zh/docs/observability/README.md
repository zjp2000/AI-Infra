---
status: Active
maintainer: pacoxu
last_updated: 2025-11-04
tags: observability, monitoring, metrics, gpu, llm, inference
canonical_path: zh/docs/observability/README.md
---

# AI 工作负载的可观测性

可观测性对于大规模运维 AI 工作负载至关重要。本指南涵盖 AI 基础设施栈中的监控、指标和可观测性工具：从 GPU 硬件到推理性能再到调度器效率。

## 概述

AI 工作负载可观测性涵盖多个层次：

1. **基础设施层**：GPU 利用率、内存、温度、功耗
2. **推理层**：请求延迟、吞吐量、token 指标
3. **调度器层**：队列深度、调度延迟、资源分配
4. **应用层**：LLM 请求追踪、提示性能、模型质量

## 基础设施侧可观测性

### GPU 监控

跟踪 GPU 健康和利用率的硬件级指标：

**关键指标：**

- **GPU 利用率**：GPU 活跃计算的时间百分比
- **GPU 内存**：每 GPU 已分配 vs 可用内存
- **温度**：工作温度和热节流
- **功耗**：每 GPU 和整个集群的瓦特使用量
- **SM 占用率**：流式多处理器利用率
- **NVLink/PCIe 带宽**：GPU 间和 CPU-GPU 通信速度

**工具和项目：**

- <a href="https://github.com/NVIDIA/dcgm-exporter">`NVIDIA DCGM Exporter`</a>：面向 NVIDIA Data Center GPU Manager 指标的 Prometheus exporter。Kubernetes 中 GPU 监控的行业标准。
- <a href="https://github.com/NVIDIA/gpu-operator">`NVIDIA GPU Operator`</a>：自动部署 DCGM 和监控栈的 Kubernetes Operator。详见 [NVIDIA GPU Operator](../kubernetes/nvidia-gpu-operator.md)。
- <a href="https://github.com/kubernetes-sigs/metrics-server">`Metrics Server`</a>：Kubernetes SIG 项目，用于资源指标（CPU、内存、GPU）。
- <a href="https://github.com/prometheus/prometheus">`Prometheus`</a>：CNCF Graduated。用于指标收集和告警的时间序列数据库。
- <a href="https://github.com/grafana/grafana">`Grafana`</a>：CNCF Incubating。GPU 和基础设施指标的可视化和仪表板。

**示例查询：**

```promql
# 跨集群的 GPU 利用率
avg(DCGM_FI_DEV_GPU_UTIL) by (gpu, kubernetes_node)

# GPU 内存使用百分比
(DCGM_FI_DEV_FB_USED / (DCGM_FI_DEV_FB_USED + DCGM_FI_DEV_FB_FREE)) * 100

# 温度过高的 GPU
DCGM_FI_DEV_GPU_TEMP > 80
```

### 节点和集群指标

除 GPU 外，还需监控计算、网络和存储资源：

- **CPU/内存**：节点级资源利用率
- **网络**：带宽、丢包率、RDMA 指标
- **存储**：模型加载和检查点的 I/O 吞吐量、延迟
- **NUMA**：影响 GPU 性能的非统一内存访问模式

## 推理侧可观测性

### 关键推理指标

LLM 推理性能由几个关键指标衡量：

**延迟指标：**

- **TTFT（首 Token 延迟）**：从请求提交到生成第一个 token 的时间。对交互式应用的体验至关重要。目标：实时聊天 < 200ms。
- **TPOT（每输出 Token 时间）**：生成第一个 token 后每个后续 token 的生成时间。影响流式传输速度。目标：流畅流式传输 < 50ms。
- **ITL（Token 间延迟）**：连续 token 之间的时间变化。低 ITL 提供一致的流式体验。
- **E2E 延迟**：从提交到完成的请求总延迟。

**吞吐量指标：**

- **每秒 Token 数（TPS）**：所有请求每秒生成的 token 总数。
- **每秒请求数（RPS）**：每秒完成的推理请求数。
- **批次大小**：一起处理的请求数量（越高 = 越好的 GPU 利用率）。

**资源指标：**

- **KV 缓存利用率**：transformer 模型中用于键值缓存的内存。
- **解码效率**：计算时间与 token 生成时间的比率。
- **GPU 计算利用率**：GPU 活跃执行推理的百分比。

**质量指标：**

- **Token 准确性**：用于推测解码和草稿模型。
- **缓存命中率**：用于前缀缓存和提示缓存策略。

### 推理可观测性工具

**OpenTelemetry 原生方案：**

- <a href="https://github.com/openlit/openlit">`OpenLit`</a>：面向 AI 工程的开源平台，提供 OpenTelemetry 原生的 LLM 可观测性。为 GenAI 应用提供指标、追踪和日志。
- <a href="https://github.com/traceloop/openllmetry">`OpenLLMetry`</a>：基于 OpenTelemetry 的 GenAI 和 LLM 应用开源可观测性方案。一行集成主流 LLM 框架。

**LLM 特定平台：**

- <a href="https://github.com/langfuse/langfuse">`Langfuse`</a>：开源 LLM 工程平台，提供可观测性、指标、评估和提示管理。支持追踪复杂的 LLM 链和 agent 工作流。
- <a href="https://github.com/truera/trulens">`TruLens`</a>：LLM 实验和 AI agent 的评估和跟踪工具。提供用于质量评估的反馈函数。

**模型验证和测试：**

- <a href="https://github.com/deepchecks/deepchecks">`Deepchecks`</a>：用于持续验证 ML 模型和数据的测试工具。包括漂移检测和模型性能监控。

**网络和分布式追踪：**

- <a href="https://github.com/deepflowio/deepflow">`DeepFlow`</a>：具有深度网络可见性的可观测平台。支持跨分布式 AI 工作负载和微服务的追踪。
- <a href="https://github.com/okahu">`Okahu`</a>：专注于 AI 基础设施可观测性和优化的组织。

### 推理平台集成

大多数 LLM 推理平台提供内置可观测性：

**vLLM 指标：**

```python
# vLLM 在 /metrics 端点暴露 Prometheus 指标
- vllm:prompt_tokens_total
- vllm:generation_tokens_total
- vllm:time_to_first_token_seconds
- vllm:time_per_output_token_seconds
- vllm:num_requests_running
- vllm:gpu_cache_usage_perc
```

**Ray Serve（用于分布式推理）：**

```python
# Ray 仪表板提供详细的指标
- ray_serve_replica_processing_latency_ms
- ray_serve_num_router_requests
- ray_serve_deployment_queued_queries
```

**Triton Inference Server：**

```bash
# Triton 指标端点
curl http://localhost:8002/metrics

# 关键指标
nv_inference_request_success
nv_inference_request_duration_us
nv_inference_queue_duration_us
nv_gpu_utilization
```

## 调度器侧可观测性

### Kubernetes 调度器指标

监控调度效率和资源分配：

**关键指标：**

- **调度延迟**：从 pod 创建到调度决策的时间
- **调度尝试次数**：成功调度前的尝试次数
- **待处理 Pod**：按原因分类等待调度的 pod
- **抢占事件**：为高优先级 pod 驱逐的低优先级 pod
- **队列深度**：批处理调度队列中等待的任务

**Kubernetes 原生指标：**

```promql
# 按操作的调度延迟
histogram_quantile(0.99,
  sum(rate(scheduler_scheduling_duration_seconds_bucket[5m]))
  by (le, operation))

# 按原因的待处理 pod
sum(kube_pod_status_phase{phase="Pending"}) by (reason)

# 失败的调度尝试
rate(scheduler_schedule_attempts_total{result="error"}[5m])
```

**批处理调度器可观测性：**

- <a href="https://github.com/kubernetes-sigs/kueue">`Kueue`</a>：Kubernetes 的任务排队系统，提供队列深度指标、准入延迟和资源配额利用率。
- <a href="https://github.com/volcano-sh/volcano">`Volcano`</a>：批处理调度器，提供 gang scheduling 指标、任务生命周期跟踪和队列管理。
- <a href="https://github.com/koordinator-sh/koordinator">`Koordinator`</a>：基于 QoS 的调度，具有细粒度资源指标和混部效率。

### GPU 调度指标

GPU 工作负载调度的专门指标：

- **GPU 分配时间**：查找和分配 GPU 资源的时间
- **GPU 碎片化**：由于分配模式导致的 GPU 容量浪费
- **多 GPU 任务调度**：需要多个 GPU 的任务的延迟
- **调度后 GPU 利用率**：实际 vs 请求的 GPU 使用量

**工具：**

- <a href="https://github.com/Project-HAMi/HAMi">`HAMi`</a>：异构 AI 计算虚拟化中间件，提供 GPU 共享指标
- <a href="https://github.com/NVIDIA/k8s-device-plugin">`NVIDIA Device Plugin`</a>：Kubernetes 中的 GPU 分配跟踪

## 最佳实践

### 1. 仪器化策略

- **从基础设施指标开始**：GPU、CPU、内存、网络
- **添加推理指标**：应用层的 TTFT、TPOT、吞吐量
- **实现分布式追踪**：跨服务跟踪请求
- **启用调试日志**：可配置的详细程度用于故障排除

### 2. 告警和 SLO

为关键指标定义服务级别目标：

```yaml
# 推理延迟的 SLO 示例
- alert: HighTTFT
  expr: histogram_quantile(0.95, ttft_seconds) > 0.5
  annotations:
    summary: "95th percentile TTFT exceeds 500ms"

- alert: LowGPUUtilization
  expr: avg(DCGM_FI_DEV_GPU_UTIL) < 30
  annotations:
    summary: "GPU utilization below 30% - possible inefficiency"
```

### 3. 性能优化工作流

1. **基线**：测量当前性能（TTFT、TPOT、GPU 利用率）
2. **识别瓶颈**：使用指标查找约束（GPU、内存、网络）
3. **优化**：应用技术（批处理、缓存、量化）
4. **验证**：测量改进并确保没有回归
5. **监控**：持续跟踪以检测性能下降

### 4. 成本优化

使用可观测性数据降低基础设施成本：

- **合理调整 GPU 实例**：将工作负载与 GPU 容量匹配
- **识别空闲资源**：缩减利用不足的 GPU
- **优化批次大小**：最大化每 GPU 的吞吐量
- **缓存频繁使用的提示**：减少冗余计算

### 5. 多租户监控

对于共享 AI 基础设施：

- **每租户指标**：资源使用量、请求量、延迟
- **公平份额执行**：监控并告警配额违规
- **嘈杂邻居检测**：识别影响其他工作负载的工作负载
- **成本分摊/展示**：用于成本分配的准确使用归因

## 与云原生生态系统的集成

### 可观测性栈

**推荐架构：**

```text
┌─────────────────────────────────────────────────────────────┐
│                     Grafana（仪表板）                         │
└─────────────────────────────────────────────────────────────┘
                              ▲
                              │
┌─────────────────────────────────────────────────────────────┐
│              Prometheus（指标存储）                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ DCGM Exporter│  │ vLLM Metrics │  │ K8s Metrics  │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
                              ▲
                              │
┌─────────────────────────────────────────────────────────────┐
│              OpenTelemetry Collector                         │
│  （来自 LLM 应用的追踪、指标、日志）                          │
└─────────────────────────────────────────────────────────────┘
                              ▲
                              │
┌─────────────────────────────────────────────────────────────┐
│          AI 工作负载（推理、训练）                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   vLLM       │  │   SGLang     │  │  Training    │      │
│  │   Pods       │  │   Pods       │  │  Jobs        │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
```

### CNCF 可观测性项目

- <a href="https://github.com/prometheus/prometheus">`Prometheus`</a>：CNCF Graduated。指标收集和告警
- <a href="https://github.com/open-telemetry/opentelemetry-collector">`OpenTelemetry`</a>：CNCF Incubating。统一可观测性框架
- <a href="https://github.com/grafana/grafana">`Grafana`</a>：CNCF Incubating。可视化和仪表板
- <a href="https://github.com/grafana/loki">`Loki`</a>：CNCF Incubating。日志聚合
- <a href="https://github.com/jaegertracing/jaeger">`Jaeger`</a>：CNCF Graduated。分布式追踪

## 学习路径

### 第一阶段：基础（第 1-2 周）

- 了解 Prometheus 指标和 PromQL 基础
- 部署 DCGM Exporter 并创建 GPU 监控仪表板
- 学习 OpenTelemetry 概念（追踪、指标、span）

### 第二阶段：推理可观测性（第 3-4 周）

- 使用 OpenLLMetry 或 OpenLit 对 LLM 推理进行仪器化
- 跟踪 TTFT、TPOT 和吞吐量指标
- 创建推理性能仪表板
- 设置延迟 SLO 告警

### 第三阶段：高级主题（第 5-6 周）

- 实现跨 LLM 服务的分布式追踪
- 监控批处理调度器（Kueue、Volcano）
- 构建成本归因和成本分摊仪表板
- 基于可观测性数据优化工作负载

## LLM 能力评测

作为运行时可观测性的补充，模型评估衡量 LLM 输出在知识、推理、代码生成、指令遵循、事实性、安全性和多语言能力等方面的质量。

### 开源评估框架

| 工具 | 描述 |
| --- | --- |
| [lm-eval-harness](https://github.com/EleutherAI/lm-evaluation-harness) | 通用评估框架，包含数百个任务适配器 |
| [OpenCompass](https://github.com/open-compass/opencompass) | 一体化评估系统，任务集丰富，适合内部平台 |
| [HELM](https://github.com/stanford-crfm/helm) | Stanford 的系统评估框架，维度设计全面 |
| [AlpacaEval](https://github.com/tatsu-lab/alpaca_eval) | 指令遵循和对话质量的自动成对评估 |
| [lighteval](https://github.com/huggingface/lighteval) | Hugging Face 的轻量级评估框架，易于集成到 CI |
| [EvalPlus](https://github.com/evalplus/evalplus) | 增强版 HumanEval 代码评估，测试用例更严格 |
| [SWE-bench](https://github.com/princeton-nlp/SWE-bench) | 真实软件工程任务基准测试，用于评估代码 agent |
| [FastChat / MT-Bench](https://github.com/lm-sys/FastChat) | 多轮对话评估脚本和基准测试 |
| [Ragas](https://github.com/explodinggradients/ragas) / [DeepEval](https://github.com/confident-ai/deepeval) | RAG 管道和应用链质量评估 |

### 关键基准数据集

- **知识与语言**：MMLU / MMLU-Pro, ARC, HellaSwag, TruthfulQA
- **推理与数学**：GSM8K, MATH, BBH (Big-Bench Hard), AIME
- **代码**：HumanEval / MBPP, EvalPlus, SWE-bench, LiveCodeBench
- **指令遵循**：MT-Bench, AlpacaEval, IFEval
- **安全**：AdvBench, HarmBench, RealToxicityPrompts
- **多语言**：XNLI, XQuAD / TyDiQA, MIRACL

## 参考和进一步阅读

- [CNCF Observability Landscape](https://landscape.cncf.io/?group=observability-and-analysis)
- [Prometheus Best Practices](https://prometheus.io/docs/practices/)
- [OpenTelemetry for LLMs](https://opentelemetry.io/docs/languages/)
- [NVIDIA DCGM Documentation](https://docs.nvidia.com/datacenter/dcgm/)
- [Kubernetes Monitoring Architecture](https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/)
- 另见：[Kubernetes Guide](../kubernetes/README.md) 了解调度器指标
- 另见：[Inference Guide](../inference/README.md) 了解推理平台指标

## 路线图

正在进行的提案和讨论：

- 通过 DRA（动态资源分配）增强 GPU 指标
- 跨推理引擎的标准化 LLM 指标格式
- 与 Kubernetes WG AI Gateway 集成以实现统一可观测性
- 基于可观测性数据的成本优化建议

---

_本指南整合了 AI 基础设施的可观测性最佳实践。有关具体实施细节，请参考每层链接的文档。_
