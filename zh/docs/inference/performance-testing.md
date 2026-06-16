---
status: Active
maintainer: pacoxu
last_updated: 2025-11-04
tags: performance, testing, benchmark, inference, llm
canonical_path: zh/docs/inference/performance-testing.md
---

# AI/LLM 推理的性能测试与基准工具

性能测试和基准测试对于验证和优化大规模 AI/LLM 推理系统至关重要。本指南介绍在 Kubernetes 和云原生环境中测试推理性能、测量吞吐量、延迟和资源利用率的关键工具和框架。

## 概述

AI 推理的性能测试涉及测量：

- **吞吐量**：每秒请求数（RPS）或每秒 token 数（TPS）
- **延迟**：首 token 时间（TTFT）、每输出 token 时间（TPOT）、token 间延迟（ITL）
- **资源利用率**：GPU/CPU 使用率、内存消耗、网络带宽
- **可扩展性**：不同负载和集群规模下的性能
- **成本效率**：每美元性能、资源优化

## Kubernetes 原生性能测试工具

### kubernetes-sigs/inference-perf

[`inference-perf`](https://github.com/kubernetes-sigs/inference-perf) 是由 Kubernetes SIG Serving 开发的 Kubernetes 原生性能测试框架。它为在 Kubernetes 环境中基准测试推理工作负载提供了标准化方法。

**关键特性：**

- Kubernetes 原生设计，使用 CRD 进行测试配置
- 支持多个推理框架（vLLM、TensorRT-LLM、TGI 等）
- 标准化指标收集和报告
- 与 Kubernetes 调度和资源管理集成
- 专为云原生推理平台设计

**用例：**

- 在 Kubernetes 上基准测试 LLM 服务平台
- 在生产部署前验证推理性能
- 比较不同的推理引擎和配置
- 测试负载下的自动缩放行为

### Azure/kperf

[`kperf`](https://github.com/Azure/kperf) 是由 Microsoft Azure 开发的 Kubernetes 性能基准测试工具。虽然并非专为 AI 工作负载设计，但它提供了全面的集群性能测试能力。

**关键特性：**

- 集群级性能测试
- Pod 调度和启动延迟测量
- 资源争用测试
- 网络和存储性能基准测试
- 支持大规模集群测试

### NVIDIA/knavigator

[`knavigator`](https://github.com/NVIDIA/knavigator) 是 NVIDIA 的 Kubernetes 集群性能测试工具，专为验证 GPU 工作负载性能和集群行为而设计。

**关键特性：**

- GPU 特定性能测试
- 集群混沌和压力测试
- 作业调度性能验证
- GPU 资源分配测试
- 大规模集群模拟

## 推理引擎特定基准工具

### Triton Inference Server 性能工具

NVIDIA Triton Inference Server 提供全面的性能测试工具：

#### triton-inference-server/perf_analyzer

[`perf_analyzer`](https://github.com/triton-inference-server/perf_analyzer) 是 Triton Inference Server 的标准性能测试工具。

**关键特性：**

- 全面的推理性能指标
- 支持多种模型格式（TensorRT、ONNX、PyTorch 等）
- 并发请求测试
- 延迟和吞吐量分析
- 自定义请求负载生成

#### genai-perf

[`genai-perf`](https://github.com/triton-inference-server/perf_analyzer/blob/main/genai-perf/README.md) 是专为基准测试大型语言模型（LLM）和生成式 AI 推理而设计的工具。

**关键特性：**

- LLM 特定指标（TTFT、TPOT、ITL）
- 支持流式和批量推理
- Token 级性能分析
- 多 LLM 框架支持（vLLM、TGI、TensorRT-LLM）
- 真实工作负载生成

### vLLM 基准工具

#### vllm-project/vllm benchmark_serving.py

[`benchmark_serving.py`](https://github.com/vllm-project/vllm/blob/main/benchmarks/benchmark_serving.py) 是 vLLM 的内置基准测试脚本，用于测试服务性能。

**关键特性：**

- 原生 vLLM 集成
- 支持 OpenAI 兼容 API 测试
- 可配置的请求模式和负载配置文件
- Token 生成指标
- 吞吐量和延迟分析

### fmperf-project/fmperf

[`fmperf`](https://github.com/fmperf-project/fmperf) 是一个针对基础模型（FM）和大型语言模型的全面基准测试框架。

**关键特性：**

- 多框架支持（vLLM、TGI、TensorRT-LLM 等）
- 标准化基准测试套件
- 真实负载模式
- 全面指标收集
- 性能比较报告

## 云平台特定工具

### GCP latency-profile-generator

[`latency-profile-generator`](https://github.com/GoogleCloudPlatform/ai-on-gke/tree/main/benchmarks/benchmark/tools/profile-generator) 是 Google Cloud 的工具，用于为 AI 推理工作负载生成真实延迟配置文件。

## 负载测试和压力测试工具

### openshift-psap/llm-load-test

[`llm-load-test`](https://github.com/openshift-psap/llm-load-test) 是由 Red Hat 的 OpenShift PSAP 团队开发的、专门为 LLM 推理服务设计的负载测试框架。

## 行业标准和基准

### MLCommons Inference

[`MLCommons Inference`](https://docs.mlcommons.org/inference/) 提供跨不同硬件和软件平台的 AI 推理标准化基准。

### sgl-project/genai-bench

[`genai-bench`](https://github.com/sgl-project/genai-bench) 是由 SGLang 项目开发的生成式 AI 工作负载的全面基准测试套件。

## 性能测试最佳实践

### 测试环境设置

- **一致硬件**：使用专用 GPU 节点或一致的实例类型
- **网络隔离**：测试期间最小化网络干扰
- **预热期**：在收集指标前允许模型预热
- **多次运行**：进行多次测试运行以获得统计显著性

### 指标收集

- **延迟百分位数**：跟踪 P50、P90、P95、P99 延迟
- **负载下吞吐量**：在不同负载水平下测量 RPS/TPS
- **资源利用率**：监控 GPU、CPU、内存和网络使用情况
- **错误率**：跟踪请求失败和超时率

### 工作负载设计

- **真实模式**：使用类似生产的请求模式
- **可变输入长度**：测试不同输入 token 长度
- **批量 vs. 流式**：测试批量和流式推理模式
- **并发请求**：测量并发负载下的性能

## 学习主题

- 理解推理性能指标（TTFT、TPOT、ITL）
- Kubernetes 原生性能测试策略
- GPU 性能分析和优化
- 基于推理指标的自动缩放
- 比较推理引擎和部署策略
- 生产负载测试和容量规划

## KubeCon 参考资料

- [KubeCon EU 2025：AI/ML 工作负载的性能测试](https://kccnceu2025.sched.com/event/1tx7Q)

## 其他资源

- [LLM 性能测试中文文章](https://mp.weixin.qq.com/s/6vfE_VGF66vDmh2R4RTC1g)
- Kubernetes SIG Serving 推理性能文档
- NVIDIA 推理工作负载优化指南
- 云提供商 AI 推理最佳实践

## 相关文档

- [推理指南](./README.md)：推理引擎和平台概述
- [模型生命周期](./model-lifecycle.md)：冷启动优化和模型管理
- [缓存](./caching.md)：KV 缓存性能优化
- [AIBrix](./aibrix.md)：具有内置性能优化的云原生 LLM 推理

---

**注意：** 本指南部分内容由 AI 辅助生成。在生产环境中使用前请验证工具版本、特性和链接。
