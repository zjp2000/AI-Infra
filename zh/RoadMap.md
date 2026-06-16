---
status: Active
maintainer: pacoxu
last_updated: 2026-04-03
tags: roadmap, planning, future-work, ai-native, 路线图
canonical_path: zh/RoadMap.md
---

# AI-Infra 路线图 🗺️

本文档概述了 AI-Infra 仓库即将推出的功能、正在考虑的主题以及可能超出范围的项目。路线图有助于引导本学习资源的演进，并为未来内容设定预期。

## 🌅 AI Native 时代：云原生的下半场（2025–2035）

云原生的旅程可分为两个阶段：

- **上半场 (2015–2025)**：聚焦容器、微服务和应用交付
- **下半场 (2025–2035)**：聚焦 AI Native 平台工程，模型、智能体和推理工作负载成为一等公民

idea from https://mp.weixin.qq.com/s/di2HEsjLoCO4NB2Act6Qmg

### 关键范式转变

| 维度 | 上半场 (2015–2025) | 下半场 (2025–2035, AI Native) |
| --- | --- | --- |
| 核心对象 | 容器、Pod、微服务 | 模型、推理任务、智能体、数据管道 |
| 平台目标 | 稳定的应用交付 | 持续高效的 AI 工作负载与智能体编排 |
| 抽象层 | Deployment / Service / Ingress / Job | Model / Endpoint / Graph / Policy / Agent |
| 资源调度 | CPU / 内存 / 节点 | GPU / TPU / ASIC / KV Cache / 带宽 / 功耗 |
| 工程重点 | DevOps / GitOps / 平台工程 1.0 | AI Native 平台工程 / AI SRE |
| 安全与合规 | 镜像安全、CVE、供应链 SBOM | 模型安全、数据安全、AI 供应链与"幻觉依赖" |
| 运行时形态 | 容器 + VM + 无服务器 | 容器 + WASM + Nix + Agent Runtime |

### AI Native 重点方向

本仓库正在演进以应对 AI Native 时代的关键挑战：

| 方向 | 技术重点 | 与上半场的本质区别 |
| --- | --- | --- |
| **AI Native 平台** | Model/Agent 一等公民，统一抽象与治理 | 对象从服务转向模型和推理 |
| **资源调度** | DRA、异构算力、拓扑感知、功耗与成本优化 | 从静态配额到基于策略的动态分配 |
| **运行时** | 容器 + WASM + Nix + Agent Runtime | 从"进程容器化"到"执行图容器化" |
| **平台工程** | IDP + AI SRE + 安全 + 成本 + 合规 | 从工具组装到"自治平台" |
| **安全与供应链** | LLM 依赖、模型权重、数据集、向量数据库的全链路治理 | 受保护资产从镜像扩展到"所有 AI 工程资产" |
| **开源与生态** | AI Infra / Model Runtime / Agent Runtime 上游协作 | 不仅是"使用开源"，而是"在开源中构建未来" |

## 🚀 即将推出

以下主题按 AI Native 重点方向组织，计划在未来添加到仓库中：

### 🔧 执行积压（已整理，2026 Q2）

对于可直接转化为 issue 的可操作项，请使用：
[Executable Backlog 2026 Q2](../docs/planning/executable-backlog-2026Q2.md).

本季度的执行节奏：

- **第 1 波（第 1-4 周）**：智能体可观测性 + 运行时安全基线 + eval 门控
- **第 2 波（第 5-8 周）**：调度控制 + 推理一致性门控
- **第 3 波（第 9-12 周）**：多集群突发 + 模型存储基线

RoadMap 保持战略方向；积压文档跟踪条目级执行和验收标准。

### AI Native 平台

- **Model/Agent 抽象模式**：如何将模型和智能体视为 Kubernetes 一等公民，包括 CRD 设计模式
- **统一模型治理**：模型版本管理、发布策略和多模型编排的最佳实践

### 资源调度

- **DRA 更新**：新的动态资源分配实现，如 [dranet](https://github.com/google/dranet)
- **异构计算**：跨 GPU、TPU、NPU 和定制 ASIC 的多加速器调度
- **拓扑感知调度**：面向分布式训练和推理的 GPU/网络拓扑优化
- **功耗与成本优化**：AI 工作负载的能耗感知调度和成本模型
- **Kueue + HAMi 集成**：将 [Kueue](https://github.com/kubernetes-sigs/kueue) 批处理调度与 [HAMi](https://github.com/Project-HAMi/HAMi) GPU 共享/隔离相结合的云原生 GPU 资源调度——参考：[ForceInjection Kueue+HAMi](https://github.com/ForceInjection/AI-fundermentals/blob/main/04_cloud_native_ai_platform/k8s/Kueue%20+%20HAMi.md)
- **NVIDIA Container Toolkit 原理分析**：深入 NVIDIA Container Toolkit 的底层机制，实现容器化 GPU 支持——参考：[ForceInjection NVIDIA Container Toolkit](https://github.com/ForceInjection/AI-fundermentals/blob/main/04_cloud_native_ai_platform/k8s/Nvidia%20Container%20Toolkit%20%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90.md)
- **NVIDIA K8s Device Plugin 分析**：NVIDIA Kubernetes 设备插件的架构和实现细节——参考：[ForceInjection Device Plugin](https://github.com/ForceInjection/AI-fundermentals/blob/main/04_cloud_native_ai_platform/k8s/nvidia-k8s-device-plugin-analysis.md)

### GPU OS（虚拟化与性能分析）

面向优化 GPU 利用率和调试性能瓶颈的 GPU 级虚拟化、资源分区和性能分析工具。

#### GPU 虚拟化

- **MIG（多实例 GPU）**：将单个 GPU 分区为多个隔离实例，用于多租户工作负载
- **vGPU / SR-IOV**：通过虚拟化技术跨多个 VM/容器共享 GPU
- **CUDA MPS（多进程服务）**：在单个 GPU 上实现并发多进程执行以提高利用率

#### GPU 性能分析 / 仪表化 / 监控

- **Nsight Systems**：端到端时间线分析（CPU↔GPU、内核启动、通信/IO 重叠）
- **Nsight Compute**：每个内核的指标（占用率、内存带宽、warp 效率等）
- **CUPTI**：用于收集 GPU 事件和性能计数器的底层接口
- **NVTX（NVIDIA Tools Extension）**：代码注释/范围标记，用于改善 Nsight 可读性和阶段对齐

### 内核与算子优化

面向 AI 工作负载的 GPU 内核开发、优化工具和高性能算子库，跨 GPU 世代充分利用硬件能力。

#### 内核 DSL 与代码生成

- **[Triton (triton-lang)](https://github.com/triton-lang/triton)**：用于在 Python 中编写高性能内核的自定义 GPU 内核 DSL 和编译器
- **[tilelang](https://github.com/tile-ai/tilelang)**：面向高性能内核开发的 DSL

#### Transformer 内核

- **[FlashInfer](https://github.com/flashinfer-ai/flashinfer)**：高性能推理侧内核（attention, MoE 等）
- **[FlashAttention](https://github.com/Dao-AILab/flash-attention)**：经典内存高效 attention 内核
- **[xFormers](https://github.com/facebookresearch/xformers)**：优化的 transformer 构建块及相关内核

#### 数学原语

- **[CUTLASS](https://github.com/NVIDIA/cutlass)**：NVIDIA 的 GEMM 和算子基础设施
- **[DeepGEMM](https://github.com/deepseek-ai/DeepGEMM)**：DeepSeek 的高性能 GEMM 实现和探索

#### 量化与低精度

- **[bitsandbytes](https://github.com/bitsandbytes-foundation/bitsandbytes)**：低位训练和推理工具链
- **GPTQ / AWQ**：面向推理的训练后量化方法
- **[AMD Quark](https://quark.docs.amd.com/)**: AMD 生态系统模型优化和量化工具链
- **[LLM Compressor](https://github.com/vllm-project/llm-compressor)**：面向 LLM 的压缩和量化工具链（vLLM 项目）
- **[NVIDIA ModelOpt](https://github.com/NVIDIA/TensorRT-Model-Optimizer)**：NVIDIA 生态系统模型优化和量化工具链

### 分布式通信（NCCL / UCX / Gloo）

面向分布式训练和推理的集合通信库和高性能网络框架。

- **[NCCL](https://github.com/NVIDIA/nccl)**：核心 GPU 集合通信库（AllReduce, AllGather 等）——分布式训练的 backbone
- **[UCX](https://github.com/openucx/ucx)**：支持 RDMA、共享内存和网络协议的高性能通信框架
- **[Gloo](https://github.com/facebookincubator/gloo)**：PyTorch 生态系统中常用的集合通信库

### 运行时演进

- **智能体运行时环境**：AI 智能体的沙箱和隔离模式（Agent Sandbox）
- **AI 中的 WASM**：WebAssembly 在 AI 推理和边缘部署中的用例
- **执行图容器化**：从进程容器化迈向完整的推理 DAG 管理

### 平台工程 2.0

- 🎓 **每周学习挑战与案例研究**：帮助工程师实践 AI 基础设施概念的交互式学习材料
- **智能体工作流开发**：深入探索 Dify、KAgent 或 Dagger 等平台——[Issue #30](https://github.com/pacoxu/AI-Infra/issues/30)
- **AI SRE 实践**：为 AI 工作负载调整的站点可靠性工程
- **自治平台模式**：自愈、自动扩缩容和自优化的 AI 基础设施

### 硬件与基础设施

- **SuperNode 架构**：涵盖大规模 AI 硬件系统：
  - 华为 CloudMatrix 384（UnifiedBus）
  - NVIDIA GB200 NVL72（36 个 Grace CPU + 72 个 Blackwell GPU，机架级液冷设计）
  - 沐曦耀龙 S8000 G2 超节点 (Moffett YaoLong S8000 G2 SuperNode)

### 可观测性

- **可观测性**：AI 工作负载的高级监控和指标
  - eBPF for LLM (Deepflow)
  - 自动扩缩容指标和策略：TTFT, TPOT, ITL 等
  - [Issue #78](https://github.com/pacoxu/AI-Infra/issues/78)

### 推理框架与分布式服务

- **llm-d**：基于 Kubernetes 的云原生高性能分布式 LLM 推理框架——参考：[ForceInjection llm-d intro](https://github.com/ForceInjection/AI-fundermentals/blob/main/04_cloud_native_ai_platform/k8s/llm-d-intro.md)
- **vLLM + LWS 多节点推理**：使用 LeaderWorkerSet (LWS) 控制器在 Kubernetes 上进行多节点、多 GPU 推理部署——参考：[ForceInjection LWS intro](https://github.com/ForceInjection/AI-fundermentals/blob/main/04_cloud_native_ai_platform/k8s/lws_intro.md)

### 强化学习与对齐（后训练 / RLHF）

面向大型语言模型的人类反馈强化学习 (RLHF) 和后训练对齐框架。

- **[verl](https://github.com/volcengine/verl)**：面向 LLM 的 RL 训练系统框架，具有灵活的 rollout 后端（vLLM/SGLang）——字节跳动 / 火山引擎
- **[SLIME](https://github.com/THUDM/SLIME)**：专为 LLM 后训练设计的 RL 扩展框架——THUDM
- **[OpenRLHF](https://github.com/OpenRLHF/OpenRLHF)**：支持 PPO、DPO 和 GRPO 的开源 RLHF 训练框架
- **[TRL](https://github.com/huggingface/trl)**：HuggingFace 生态系统中的 PPO/DPO 及其他对齐训练器

### KV Cache 深入分析

- **Mooncake 架构**：以 KV-cache 为中心的 LLM 推理系统设计详细分析——参考：[ForceInjection Mooncake architecture](https://github.com/ForceInjection/AI-fundermentals/blob/main/09_inference_system/kv_cache/mooncake_architecture.md)
- **LMCache 详细分析**：LMCache 源码级深入，涵盖其四层存储架构 (L1-L4)、连接器、引擎、存储后端、控制器、服务器、CacheBlend 和 CacheGen——参考：
  - [LMCache overview](https://github.com/ForceInjection/AI-fundermentals/blob/main/09_inference_system/kv_cache/lmcache/lmcache_overview.md)
  - [LMCacheConnector](https://github.com/ForceInjection/AI-fundermentals/blob/main/09_inference_system/kv_cache/lmcache/lmcache_connector.md)
  - [LMCacheEngine](https://github.com/ForceInjection/AI-fundermentals/blob/main/09_inference_system/kv_cache/lmcache/lmcache_engine.md)
  - [CacheBlend](https://github.com/ForceInjection/AI-fundermentals/blob/main/09_inference_system/kv_cache/lmcache/cache_blend.md)
  - [CacheGen](https://github.com/ForceInjection/AI-fundermentals/blob/main/09_inference_system/kv_cache/lmcache/cachegen.md)
- **阿里云 Tair KVCache**：企业级集中式 KVCache 管理系统设计与与 LMCache 的对比——参考：[ForceInjection Tair KVCache](https://github.com/ForceInjection/AI-fundermentals/blob/main/09_inference_system/kv_cache/ali_tair_kvcache/tair-kvcache-architecture-design.md)
- **vLLM KV Offloading vs LMCache**：涵盖存储层次和跨实例共享能力的架构对比——参考：[ForceInjection KV Offloading analysis](https://github.com/ForceInjection/AI-fundermentals/blob/main/09_inference_system/kv_cache/kv_offloading_analysis.md)

### 存储与检查点 IO

面向训练检查点和模型权重的高性能分布式文件系统和张量序列化格式。

#### 分布式文件存储

- **[3FS (Fire-Flyer File System)](https://github.com/deepseek-ai/3FS)**：针对 AI 训练工作负载优化的高性能分布式文件存储，特别是高并发随机读取（模型加载）和小文件/元数据 IO——来自 DeepSeek

#### 检查点 / 张量 IO

- **[safetensors](https://github.com/huggingface/safetensors)**：模型权重的简单、安全的张量序列化格式——零拷贝、无任意代码执行 (HuggingFace)
- **[TensorStore](https://github.com/google/tensorstore)**：用于读取/写入大型多维数组和分片检查点的库——Google

### 向量数据库

面向嵌入存储、相似性搜索和检索增强生成 (RAG) 管道的向量数据库解决方案。

- **[Milvus](https://github.com/milvus-io/milvus)**：CNCF Incubating——专为可扩展相似性搜索构建的开源向量数据库
- **[Qdrant](https://github.com/qdrant/qdrant)**：用 Rust 编写的高性能向量相似性搜索引擎
- **[Weaviate](https://github.com/weaviate/weaviate)**：具有内置 ML 模型集成的云原生向量数据库
- **[pgvector](https://github.com/pgvector/pgvector)**：PostgreSQL 的开源向量相似性搜索扩展
- **[FAISS](https://github.com/facebookresearch/faiss)**：针对密集相似性搜索优化的本地向量索引库——Meta

### 部署最佳实践

- **DeepSeek-V3 MoE 在 H20 上部署**：使用 vLLM 在 H20 硬件上部署 DeepSeek-V3 MoE 模型的指南和 SLO 验证——参考：[ForceInjection DeepSeek-V3 deployment](https://github.com/ForceInjection/AI-fundermentals/blob/main/09_inference_system/inference_solutions/deepseek_v3_moe_vllm_h20_deployment.md)
- **Qwen2-VL-7B 在华为昇腾上部署**：多模态模型在国产华为昇腾硬件上的部署优化——参考：[ForceInjection Qwen2-VL Huawei](https://github.com/ForceInjection/AI-fundermentals/blob/main/09_inference_system/inference_solutions/qwen2_vl_7b_huawei.md)

### 模型优化

- **模型量化**：减小模型尺寸和提升推理性能的技术
- **LLM 通用/基础指南**：涵盖 LLM 基础知识、MoE 架构、Ollama 及相关工具
- **GPU 架构与 CUDA 编程**：面向 GPU 计算 AI 工作负载的硬件基础、并行计算原理和性能优化策略——参考：[ForceInjection AI Infra Course](https://github.com/ForceInjection/AI-fundermentals/blob/main/10_ai_related_course/ai_infra_course/%E5%85%A5%E9%97%A8%E7%BA%A7/%E8%AE%B2%E7%A8%BF.md)

### LLM 能力评估

跨知识、推理、代码、指令遵循、真实性、安全性、多语言能力和 RAG 质量等维度评估 LLM 质量的工具和基准。

#### 评估框架

- **[lm-evaluation-harness](https://github.com/EleutherAI/lm-evaluation-harness)**：具有广泛任务覆盖和统一报告的通用评估框架——EleutherAI
- **[OpenCompass](https://github.com/open-compass/opencompass)**：一站式评估系统，具有丰富的任务集，适合构建内部评估平台
- **[HELM](https://github.com/stanford-crfm/helm)**：具有全面度量维度的整体评估框架——Stanford CRFM
- **[AlpacaEval](https://github.com/tatsu-lab/alpaca_eval)**：面向指令遵循和对话质量的自动成对评估
- **[FastChat / MT-Bench](https://github.com/lm-sys/FastChat)**：社区广泛使用的对话评估脚本和基准
- **[lighteval](https://github.com/huggingface/lighteval)**：轻量级评估框架 (HuggingFace)，适合 CI 集成
- **[EvalPlus](https://github.com/evalplus/evalplus)**：通过更严格的测试用例和评判扩展 HumanEval 的增强代码评估
- **[SWE-bench](https://github.com/swe-bench/SWE-bench)**：真实世界软件工程任务修复基准和工具链
- **[Ragas](https://github.com/explodinggradients/ragas) / [TruLens](https://github.com/truera/trulens) / [DeepEval](https://github.com/confident-ai/deepeval)**：RAG 和应用管道质量评估工具

#### 关键基准

| 类别 | 基准 |
| --- | --- |
| **知识与语言** | MMLU / MMLU-Pro, ARC (Easy/Challenge), HellaSwag, WinoGrande, TruthfulQA, PIQA / OpenBookQA |
| **推理与数学** | GSM8K, MATH, BBH (Big-Bench Hard) / Big-Bench, AIME |
| **代码** | HumanEval / MBPP, EvalPlus, SWE-bench, LiveCodeBench |
| **指令与聊天** | MT-Bench (FastChat), AlpacaEval, IFEval |
| **真实性 / 幻觉** | TruthfulQA, FEVER, FactScore |
| **安全与鲁棒性** | AdvBench, HarmBench, Do-Not-Answer, RealToxicityPrompts / ToxiGen |
| **多语言** | XNLI, XQuAD / TyDiQA, MIRACL / BEIR (retrieval) |
| **检索 / RAG QA** | HotpotQA / 2WikiMultihopQA / MuSiQue, NaturalQuestions / TriviaQA, BEIR |

### 安全与供应链

- **LLM 安全与合规/策略**：LLM 部署的安全最佳实践和策略执行
- **AI 供应链安全**：模型权重来源、数据集治理和"幻觉依赖"管理
- **向量数据库安全**：嵌入存储和检索系统的安全模式

### 智能体通信

- **MCP 与 A2A**：模型上下文协议和智能体间通信——[Issue #32](https://github.com/pacoxu/AI-Infra/issues/32)

### 典型端到端堆栈架构

结合 AI Native 堆栈各层的参考架构，适用于常见的大规模 AI 工作负载：

#### 大规模预训练 / 微调

GPU OS (MIG / Profiling) → 分布式通信 (NCCL + K8s / Slurm) → PyTorch + Megatron-LM / DeepSpeed / FSDP → 存储 (3FS) + 检查点 IO (safetensors / TensorStore)

#### 后训练 / RLHF

PyTorch → TRL / verl / OpenRLHF / SLIME → rollout 后端 (SGLang / vLLM) → K8s + Ray (KubeRay)

#### 大规模在线推理（引擎到集群）

权重 / 模型 (3FS) → 引擎 (vLLM / TensorRT-LLM / SGLang) → 服务 (KServe / Ray Serve / Triton Inference Server / BentoML) → 网关 / 路由 (SGLang Router / AIBrix) → 控制面 (llm-d / NVIDIA Dynamo) → KV Cache (LMCache) → 内核加速 (Triton / tilelang / FlashInfer / FlashAttention / CUTLASS / DeepGEMM)

## 🚫 超出范围

以下主题目前被视为超出本仓库范围。它们可能具有价值，但要么过于宽泛、在其他地方已有广泛覆盖，或与本仓库基于 Kubernetes 的 AI 基础设施重点不够紧密：

- **通用可观测性项目**：Prometheus、Grafana、OpenTelemetry 等工具（这些在其他地方已有充分文档，且并非 AI 特有）
- **RAG**：检索增强生成架构和实现

## 💬 讨论与贡献

对于某些主题，我们可能会在决定是否纳入之前开设专门的 issue 进行讨论。如果您有新主题的建议或想贡献学习材料，请：

- 开启 issue 讨论该主题
- 查看现有 issue 了解正在进行的讨论
- 提交 pull request，内容需与本仓库的重点保持一致

## 📅 更新计划

**此路线图每季度审查和更新一次**，以确保其与快速演变的 AI 基础设施保持同步。下一次计划审查定于 2026 年 Q2。

如果您发现此路线图上的项目已过时，或想要提出新主题，请开启 issue 或提交 pull request。

---

最后更新：2026-04-03
