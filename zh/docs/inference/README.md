---
status: Active
maintainer: pacoxu
last_updated: 2026-05-26
tags: inference, llm, vllm, serving, optimization
canonical_path: zh/docs/inference/README.md
---

# LLM 推理平台与引擎

面向 Kubernetes 原生部署的 LLM 推理平台、引擎和优化技术的全面概览。

关于特定平台和技术的更多详情：

- [模型架构（Llama、Qwen、DeepSeek、Flux）](./model-architectures.md)
- [NVIDIA Dynamo：全景架构](./dynamo.md)
- [LoRA：高效 LLM 服务的低秩适配](./lora.md)
- [Grove：用于推理编排的 Kubernetes API](./grove.md)
- [kube-agentic-networking：Agentic 网络治理](./kube-agentic-networking.md)
- [OME：用于 LLM 管理的 Kubernetes Operator](./ome.md)
- [无服务器 AI 推理（Knative、AWS SageMaker、平台对比）](./serverless.md)
- [模型切换与动态调度（Aegaeon、vLLM 休眠模式）](./model-switching.md)
- [LLM 推理中的缓存](./caching.md)
- [AI Agent 的内存、上下文与数据库](./memory-context-db.md)
- [大规模专家模型（MoE 模型）](./large-scale-experts.md)
- [模型生命周期管理（冷启动、休眠模式、卸载）](./model-lifecycle.md)
- [模型分发栈一图总览（HF、MatrixHub、Harbor、Dragonfly、ModelPack、ModelExpress）](./model-distribution-stack.md)
- [性能测试与基准工具](./performance-testing.md)

## 推理平台全景

推理平台生态系统可分为三个层级：

### 推理引擎（底座引擎）

执行 LLM 推理的核心服务引擎：

| 引擎 | 组织 | 关键特性 |
| --- | --- | --- |
| [vLLM](https://github.com/vllm-project/vllm) | vLLM Project (LF AI) | PagedAttention、连续批处理、最广泛的模型支持 |
| [SGLang](https://github.com/sgl-project/sglang) | SGLang Team | RadixAttention、快速结构化输出、推测解码 |
| [TensorRT-LLM](https://github.com/NVIDIA/TensorRT-LLM) | NVIDIA | CUDA 优化、NVIDIA 硬件上最高吞吐量 |
| [Triton Inference Server](https://github.com/triton-inference-server/server) | NVIDIA | 多框架、动态批处理、模型集成 |
| [TGI (Text Generation Inference)](https://github.com/huggingface/text-generation-inference) | Hugging Face | 生产就绪、Flash Attention、Paged Attention |
| [MindIE](https://www.hiascend.com/software/mindie) | 华为 | Ascend NPU 优化推理引擎 |
| [lmdeploy](https://github.com/InternLM/lmdeploy) | 上海 AI 实验室 | Turbomind 内核、连续批处理、量化 |
| [llama.cpp](https://github.com/ggml-org/llama.cpp) | ggml-org | CPU/GPU 推理、GGUF 格式、边缘部署 |
| [Ollama](https://github.com/ollama/ollama) | Ollama | 开发者友好、本地部署、模型库 |
| [ONNX Runtime](https://github.com/microsoft/onnxruntime) | Microsoft | 跨平台、多框架、硬件加速 |
| [OpenVINO](https://github.com/openvinotoolkit/openvino) | Intel | Intel 硬件优化、模型优化工具包 |

### 推理平台（推理平台）

用于大规模编排 LLM 推理的 Kubernetes 原生平台：

| 平台 | 组织 | 架构 | 关键特性 |
| --- | --- | --- | --- |
| [AIBrix](https://github.com/vllm-project/aibrix) | vLLM Project | StormService + RoleSet CRDs | 高密度 LoRA、网关、自动缩放、P/D 分离 |
| [Grove](https://github.com/ai-dynamo/grove) | NVIDIA (ai-dynamo) | PodCliqueSet CRDs | 分层 Gang 调度、拓扑感知、多级自动缩放、启动顺序控制 |
| [Kthena](https://github.com/volcano-sh/kthena) | Volcano (CNCF Sandbox) | Serving Group / LWS | Gang 调度、拓扑感知、版本控制 |
| [NVIDIA Dynamo](https://github.com/ai-dynamo/dynamo) | NVIDIA | Grove + Kubernetes 集成（LWS / RBG） | NVIDIA 优化、分离式服务、KV 路由 |
| [OME](https://github.com/sgl-project/ome) | SGLang Project | Operator 模式 | 模型管理、生命周期、多引擎支持 |
| [KServe](https://github.com/kserve/kserve) | CNCF Incubating | InferenceService CRD | 多框架、金丝雀发布、可解释性、无服务器 |
| [llm-d](https://github.com/llm-d/llm-d) | Red Hat / IBM | 双 LWS + KServe | P/D 分离参考实现 |
| [vLLM production-stack](https://github.com/vllm-project/production-stack) | vLLM Project | 基于 Helm | 路由器、多实例、KV 感知路由 |
| [Kaito](https://github.com/kaito-project/kaito) | Microsoft (CNCF Sandbox) | Workspace CRD | GPU 自动预置、预设模型、AKS 集成 |
| [Ray Serve](https://github.com/ray-project/ray) | Anyscale / Ray | 基于 Ray | 在线服务、自动缩放、模型组合、多副本 |
| [BentoML](https://github.com/bentoml/BentoML) | BentoML | BentoService | 模型打包、多框架、云部署 |

### 编排与路由（编排与路由）

处理请求路由、负载均衡和 P/D 分离的组件：

| 组件 | 项目 | 用途 |
| --- | --- | --- |
| Gateway API Inference Extension | Kubernetes SIG | 通过 Gateway API 实现推理感知路由 |
| AIBrix Gateway | AIBrix | LLM 感知路由、负载均衡、前缀缓存 |
| kube-agentic-networking | Kubernetes SIGs | Agent 到工具/API 流量的 Agentic 网络策略和治理 |
| Kthena Router | Kthena | 拓扑感知 P/D 路由 |
| LMCache | [LMCache](https://github.com/LMCache/LMCache) | KV 缓存卸载与重用 |
| Mooncake | [Moonshot AI](https://github.com/kvcache-ai/Mooncake) | 以 KV 缓存为中心的分离式推理 |
| RBG | [SGLang](https://github.com/sgl-project/rbg) | 用于多角色推理编排的 RoleBasedGroup CRD |
| SGLang Router | [SGLang](https://github.com/sgl-project/sglang) | SGLang 的推理路由与负载均衡 |

## 重点项目

### AIBrix

[`AIBrix`](https://github.com/vllm-project/aibrix) 是一个开源、云原生的解决方案，专为在企业环境中部署、管理和扩展大型语言模型（LLM）推理而优化。作为 vLLM 项目生态系统的一部分，AIBrix 为构建可扩展的 GenAI 推理基础设施提供了必要的构建模块。

**核心亮点：**

- 高密度 LoRA 管理与动态切换
- LLM 感知网关与智能路由
- 为 LLM 工作负载量身定制的自动缩放
- 分布式推理与 KV 缓存能力
- 具有 SLO 保证的高性价比异构服务

详细信息请参见 [AIBrix 介绍](./aibrix.md)。

### Grove

[`Grove`](https://github.com/ai-dynamo/grove) 是一个 Kubernetes API，提供单一的声明式接口来编排任何 AI 推理工作负载——从简单的单 Pod 部署到复杂的多节点分离式系统。Grove 将整个推理服务系统（prefill、decode、路由等）描述为一个单独的 `PodCliqueSet` 自定义资源，平台据此协调分层 Gang 调度、拓扑感知放置、多级自动缩放和显式启动顺序。

**核心亮点：**

- **分层 Gang 调度**——原子化调度跨多个角色和缩放组的所有必需 Pod
- **拓扑感知放置**——将 Pod 放置在 NVLink 域或 RDMA 网络内以最小化通信延迟
- **多级自动缩放**——在保持 Gang 约束的同时独立缩放 PodClique
- **启动顺序**——保证正确的初始化序列（例如，工作进程先于领导者启动）
- **单一 CRD**——整个推理系统在一个 `PodCliqueSet` CR 中描述

详细信息请参见 [Grove：Kubernetes 推理编排](./grove.md)。

### kube-agentic-networking

[`kube-agentic-networking`](https://github.com/kubernetes-sigs/kube-agentic-networking) 专注于 Kubernetes 上 Agent 工作负载的网络策略和治理。这帮助平台团队在多租户环境中控制和审计 Agent 如何访问工具、API 和推理端点。

**核心亮点：**

- **Agent 感知网络治理**：Agent 到工具/API 流量的策略边界
- **Kubernetes 原生**：设计适配现有集群策略和平台运维
- **安全与合规**：改善 Agent 系统的最小权限控制和可审计性

详细信息请参见 [kube-agentic-networking：Agentic 网络治理](./kube-agentic-networking.md)。

### Kthena

[`Kthena`](https://github.com/volcano-sh/kthena) 是一个 Kubernetes 原生的 LLM 推理平台，转变了组织在生产环境中部署和管理大型语言模型的方式。Kthena 是 Volcano 生态系统的一部分，为可扩展的 LLM 推理提供全面基础设施。**最新版本：v0.3.0** 引入了 LeaderWorkerSet 支持、网络拓扑感知调度和增强的可观测性。

**核心亮点：**

- **LeaderWorkerSet 集成**：支持具有领导者-工作者拓扑的分布式推理
- **拓扑感知调度**：角色级 Gang 调度和网络拓扑感知，优化 P/D 分离（需要 Volcano v1.14+）
- **ModelServing 版本控制**：ModelServing 的原生版本控制，支持回滚
- **路由器可观测性**：全面的指标、调试端口和 E2E 测试，确保生产可靠性
- **增强的滚动更新**：可配置的 maxUnavailable 以加快部署
- **插件框架**：可扩展的自定义配置逻辑架构
- **vLLM Data Parallel**：支持 vLLM 数据并行部署模式

### llm-d

[`llm-d`](https://github.com/llm-d/llm-d) 是一个生产就绪的 LLM 推理平台，使用双 LeaderWorkSet（LWS）架构实现了 Prefill-Decode 分离。llm-d 展示了在 Kubernetes 上编排分离式推理工作负载的最佳实践。

**核心亮点：**

- 用于 P/D 分离的双 LWS 架构
- **KServe 集成**：基于 KServe 的 InferenceService 抽象实现云原生模型服务
- LMCache 集成实现高效 KV 缓存管理
- 用于智能请求路由的路由 Sidecar
- P/D 分离的生产级参考实现

详细联合 KServe + llm-d 架构的完整介绍，请参见 [云原生 AI 推理的最佳实践：使用 KServe 和 llm-d](https://kserve.github.io/website/blog/cloud-native-ai-inference-kserve-llm-d)。

关于 P/D 分离实现的详细信息，请参见 [Prefill-Decode 分离](./pd-disaggregation.md)。

### NVIDIA Dynamo

[`Dynamo`](https://github.com/ai-dynamo/dynamo) 是 NVIDIA 的开源分布式推理框架，专为 AI 工厂优化。Dynamo 支持分离式 prefill-decode 架构，并与 NVIDIA 硬件和软件栈深度集成。

**核心亮点：**

- **分离式服务**：原生 P/D 分离，支持 KV 缓存路由
- **双部署模式**：Grove（NVIDIA 原生）和 Kubernetes 原生模式（包括 LWS 和新兴 RBG 工作流）
- **智能 KV 路由**：基于 KV 缓存局部性的智能路由
- **NIXL 传输**：高性能 KV 缓存传输协议
- **AIConfigurator**：数据驱动的 P/D 配置优化工具
- **RBG 生态系统进展**：配套示例已合并到 [sgl-project/rbg#255](https://github.com/sgl-project/rbg/pull/255)，Dynamo 侧集成指南跟踪于 [ai-dynamo/dynamo#5457](https://github.com/ai-dynamo/dynamo/pull/5457)

有关 Dynamo 与 Grove、Planner、KVBM、NIXL 以及底层硬件和存储层的整体视图，请参见 [NVIDIA Dynamo：全景架构](./dynamo.md)。

### KServe

[`KServe`](https://github.com/kserve/kserve) 是一个 CNCF Incubating 项目，提供 Kubernetes 原生平台用于大规模部署 ML 模型，并日益增强对 LLM 推理工作负载的支持。

**核心亮点：**

- **InferenceService CRD**：跨框架的统一模型服务 API
- **LLMInferenceService CRD**：专用的 LLM 服务 API，支持 P/D 分离，原生集成 vLLM/LMCache
- **LMCache 集成**：用于 LLM 服务的 KV 缓存卸载
- **金丝雀部署**：通过流量分割实现安全发布
- **无服务器服务**：基于 Knative 的 scale-to-zero 支持
- **llm-d 集成**：通过双 LWS 架构实现分离式 LLM 服务的基础

KServe + llm-d 的组合提供了大规模的云原生 AI 推理。有关完整架构概览，请参见 [云原生 AI 推理的最佳实践：使用 KServe 和 llm-d](https://kserve.github.io/website/blog/cloud-native-ai-inference-kserve-llm-d)。

P/D 分离详情请参见 [Prefill-Decode 分离](./pd-disaggregation.md)。

## 算子与内核优化（算子与内核优化）

加速 LLM 推理操作的低级 GPU 内核库：

### Kernel DSL & 代码生成

- [`Triton (triton-lang)`](https://github.com/triton-lang/triton)：类 Python 的 DSL，用于编写自定义 GPU 内核，广泛用于 vLLM 和 SGLang
- [`tilelang`](https://github.com/tile-ai/tilelang)：针对 AI 工作负载的高性能分块内核生成的 DSL

### Transformer 内核

- [`FlashInfer`](https://github.com/flashinfer-ai/flashinfer)：高性能推理端内核（attention、MoE 等），被 vLLM 和 SGLang 使用
- [`FlashAttention`](https://github.com/Dao-AILab/flash-attention)：经典的内存高效注意力内核
- [`xFormers`](https://github.com/facebookresearch/xformers)：优化的 Transformer 构建模块和相关内核（Meta）

### 数学原语

- [`CUTLASS`](https://github.com/NVIDIA/cutlass)：NVIDIA 的 CUDA GEMM/算子构建模块
- [`DeepGEMM`](https://github.com/deepseek-ai/DeepGEMM)：DeepSeek 的高性能 GEMM 实现

## 量化与低精度（量化与低精度）

模型量化工具可减少内存占用并提高推理速度：

| 工具 | 组织 | 描述 |
| --- | --- | --- |
| [bitsandbytes](https://github.com/bitsandbytes-foundation/bitsandbytes) | bitsandbytes | 8-bit/4-bit 训练和推理量化 |
| [GPTQ](https://github.com/IST-DASLab/gptq) / [AutoGPTQ](https://github.com/AutoGPTQ/AutoGPTQ) | IST-DASLab | GPT 模型的训练后量化 |
| [AWQ](https://github.com/mit-han-lab/llm-awq) | MIT Han Lab | 激活感知权重量化 |
| [AMD Quark](https://quark.docs.amd.com/) | AMD | AMD 生态系统的模型优化和量化 |
| [LLM Compressor](https://github.com/vllm-project/llm-compressor) | vLLM Project | LLM 压缩/量化工具链 |
| [NVIDIA ModelOpt](https://github.com/NVIDIA/TensorRT-Model-Optimizer) | NVIDIA | NVIDIA 生态系统的模型优化和量化 |

## 成本优化

GPU 基础设施成本高昂。推理工作负载的有效成本优化涉及多种策略：

### GPU 共享

在单个 GPU 上运行多个工作负载以提高利用率：

| 方法 | 机制 | 最适合 |
| --- | --- | --- |
| **时间分片** | Pod 之间的 CUDA 时分复用 | 小模型、开发/测试 |
| **MIG（多实例 GPU）** | 硬件 GPU 分区（A100/H100） | 隔离、可预测的工作负载 |
| **MPS（多进程服务）** | 具有内存隔离的 CUDA 上下文共享 | 批量推理、高吞吐量 |
| **vGPU** | 具有内存限制的虚拟化 GPU | 企业多租户 |

**通过 NVIDIA GPU Operator 进行 MIG 配置：**

```yaml
# 在具有 NVIDIA A100 的节点上启用 MIG
# 7 个相等的 1/7 A100 切片（每个切片 10GB）
kubectl label node <node> nvidia.com/mig.config=all-1g.10gb
```

MIG 设置请参见 [NVIDIA GPU Operator](../kubernetes/nvidia-gpu-operator.md)。

### Spot / 抢占式实例

对启用了检查点的**容错训练**工作负载使用 spot 实例。云提供商为可中断计算提供 60–80% 的折扣：

- **AWS**：使用 Karpenter 的 EC2 Spot Instances 自动重新配置
- **GCP**：具有 GKE 自动配置的 Spot VMs
- **Azure**：具有 AKS 节点池的 Azure Spot VMs

spot 安全训练的关键要求：

- 定期检查点（每 5–15 分钟）
- 优雅关闭处理（`SIGTERM` → 检查点 → 退出）
- 具有抢占感知作业重排功能的 Kueue 或 Volcano

### 推理成本优化

- **量化**：INT8/INT4 将 GPU 内存减少 2–4 倍，每个 GPU 可运行更多副本。参见上方的 [量化](#quantization-量化与低精度) 部分。
- **推测解码**：使用小型草稿模型加速主模型（减少 TPOT，提高吞吐量）
- **连续批处理**：vLLM/SGLang 的默认批处理方式，最大化并发请求的 GPU 利用率
- **前缀缓存**：为重复的提示前缀缓存 KV 状态（参见 [缓存策略](./caching.md)）
- **模型共置**：使用 Koordinator 的混合部署模式在同一节点上调度推理和批处理工作负载

### 合理规模

分析实际 GPU 利用率以识别过度配置的工作负载：

- **DCGM Exporter** 指标：`DCGM_FI_DEV_GPU_UTIL`、`DCGM_FI_DEV_FB_USED`
- **vLLM 指标**：`vllm:gpu_cache_usage_perc`、`vllm:num_requests_running`
- 将资源请求设置为等于实际 P95 利用率（非峰值）

**参考资料：**

- [推理成本优化指南](../archive-blog/2026-01-15/2026-01-15-inference-cost-optimization_zh.md)（中文）——详细的 5 策略框架
- [无服务器推理](./serverless.md)——使用 KEDA 实现 scale-to-zero，适用于可变流量模式
