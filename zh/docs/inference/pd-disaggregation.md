---
status: Active
maintainer: pacoxu
last_updated: 2026-04-15
tags: inference, pd-disaggregation, prefill-decode, llm-optimization
canonical_path: zh/docs/inference/pd-disaggregation.md
---

# LLM 推理中的 PD 分离

本文档介绍大型语言模型（LLM）推理中的 Prefill-Decode（PD）分离概念、其优势、在常见 AI Infra 项目中的当前实现状态以及未来采用的路线图。

## 目录

- [什么是 PD 分离](#what-is-pd-disaggregation)
- [LLM 推理中的并行策略](#parallelism-strategies-in-llm-inference)
  - [Tensor 并行（TP）](#tensor-parallelism-tp)
  - [数据并行（DP）](#data-parallelism-dp)
  - [流水线并行（PP）](#pipeline-parallelism-pp)
  - [专家并行（EP）](#expert-parallelism-ep)
  - [序列并行（SP）](#sequence-parallelism-sp)
  - [零冗余并行（ZP）](#zero-redundancy-parallelism-zp)
- [工作负载解决方案](#workload-solutions)
  - [LWS（LeaderWorkSet）](#lws-leaderworkset)
  - [SGLang RBG](#sglang-rbg)
  - [AIBrix StormService](#aibrix-stormservice)
  - [Volcano Kthena](#volcano-kthena)
  - [llm-d](#llm-d)
- [项目支持状态](#project-support-status)
  - [NVIDIA Dynamo](#nvidia-dynamo)
  - [vLLM production stack](#vllm-production-stack)
  - [AIBrix](#aibrix)
  - [KServe](#kserve)
- [扩展 P/D 工作负载](#scaling-pd-workloads)
  - [传统自动缩放的挑战](#challenges-with-traditional-autoscaling)
  - [使用 AIConfigurator 进行配置优化](#configuration-optimization-with-aiconfigurator)
  - [Kubernetes 中基于 SLA 的调度](#sla-based-scheduling-in-kubernetes)
- [参考资料](#references)

---

## 什么是 PD 分离

在 LLM 推理中，该过程可分为两个不同的阶段：

- **Prefill**：并行处理整个输入提示，构建 KV 缓存。
- **Decode**：使用 KV 缓存逐个生成输出 token。

在单体设置中，prefill 和 decode 在同一个 GPU 上运行，这会导致资源争用（例如 prefill 延迟影响 decode 吞吐量）。**PD 分离**通过将 prefill 和 decode 阶段分离到不同的 GPU 工作节点或节点上解决此问题。

**优势：**

- 改善延迟（TTFT 和 TPOT）
- 每 GPU 更高吞吐量
- 每个阶段的独立缩放和调优
- 调度和负载均衡的灵活性

---

## LLM 推理中的并行策略

大型语言模型的推理和训练需要分布式计算策略来处理巨大的模型规模和高计算需求。不同的并行策略可以组合以优化性能、内存使用和吞吐量。这些策略在实现 PD 分离架构时尤为重要。

### Tensor 并行（TP）

**Tensor 并行**通过沿特定维度拆分张量（权重矩阵和激活），将神经网络的各个层分布到多个设备上。

**工作原理：**

- 将权重矩阵和激活拆分到多个 GPU 上
- 每个 GPU 处理每层计算的一部分
- 通过 all-reduce 操作同步结果
- 所有 GPU 同时处理相同的输入数据

**示例：**

```text
输入 → [GPU 1: W1_part1] → AllReduce → 输出
        [GPU 2: W1_part2] →
        [GPU 3: W1_part3] →
```

**特征：**

- **粒度**：每层并行
- **通信**：GPU 间高频 all-reduce 操作
- **内存**：模型权重分布在设备间
- **用例**：不适合单个 GPU 的大型模型

**优势：**

- 支持运行大于单 GPU 内存的模型
- 小批量的低延迟
- 对注意力层和 FFN 网络高效

**挑战：**

- GPU 间高通信开销
- 需要快速互连（NVLink、InfiniBand）
- 受限于可拆分的维度数

**PD 分离上下文：**

在 PD 分离中，当模型大小超过单 GPU 容量时，TP 通常用于 prefill 或 decode worker 内部。例如，prefill worker 可能使用 TP=4 将模型分布在 4 个 GPU 上，而 decode worker 使用 TP=2。

### 数据并行（DP）

**数据并行**在每个设备上复制完整模型，并将不同的输入数据批次分布到各设备上。

**工作原理：**

- 每个 GPU 上有完整模型副本
- 每个 GPU 处理不同的输入样本
- 训练期间同步梯度（与推理关系较小）
- 独立处理不同请求

**示例：**

```text
请求 1 → [GPU 1: 完整模型] → 响应 1
请求 2 → [GPU 2: 完整模型] → 响应 2
请求 3 → [GPU 3: 完整模型] → 响应 3
```

**特征：**

- **粒度**：样本/批次级并行
- **通信**：推理时极小（仅在训练期间）
- **内存**：完整模型在每个设备上复制
- **用例**：较小模型的高吞吐量服务

**优势：**

- 随 GPU 数量线性扩展吞吐量
- 推理时无 GPU 间通信
- 简单实现和调试
- 容错性（一个副本故障不影响其他）

**挑战：**

- 内存效率低（每个 GPU 上有完整模型）
- 限于适合单 GPU 的模型
- 对单个大请求无优势

**PD 分离上下文：**

DP 用于独立扩展 prefill 和 decode worker。例如，根据工作负载特性部署 4 个 prefill 副本和 8 个 decode 副本。每个副本独立处理不同的请求。

### 流水线并行（PP）

**流水线并行**将模型划分为顺序阶段，并将每个阶段分配到不同设备，形成计算流水线。

**工作原理：**

- 模型拆分为顺序块（层 1-10、11-20、21-30 等）
- 每个阶段分配给不同的 GPU
- 前向传播流经流水线
- 为提高效率同时处理多个微批次

**示例：**

```text
输入 → [GPU 1: 层 1-10] → [GPU 2: 层 11-20] →
        [GPU 3: 层 21-30] → [GPU 4: 层 31-40] → 输出
```

**特征：**

- **粒度**：层组并行
- **通信**：相邻阶段间的点对点通信
- **内存**：模型顺序分区到设备上
- **用例**：具有顺序依赖性的深度模型

**优势：**

- 减少每 GPU 内存（仅部分层）
- 比 TP 通信量更低（仅在阶段之间）
- 适用于非常深的模型
- 与异构 GPU 集群兼容

**挑战：**

- 流水线气泡（填充/排空流水线时的空闲时间）
- 顺序依赖性限制并行度
- 需要复杂调度才能高效
- 如果各阶段计算时间不同，会出现负载不均

**PD 分离上下文：**

PP 可用于极大模型的 prefill 或 decode worker 内部。例如，prefill worker 可能使用 PP=4 将 176B 参数模型拆分为 4 个阶段，每个阶段可能在内部使用 TP。

### 专家并行（EP）

**专家并行**特定于混合专家（MoE）模型，将不同的专家分布到各设备上。

**工作原理：**

- 每个 GPU 托管一部分专家
- 路由器网络决定激活哪些专家
- token 根据专家选择路由到适当的 GPU
- 仅激活的专家执行计算

**示例：**

```text
输入 Token → 路由器 → [GPU 1: 专家 1, 2]
                    → [GPU 2: 专家 3, 4]
                    → [GPU 3: 专家 5, 6]
                    → [GPU 4: 专家 7, 8] → 输出
```

**特征：**

- **粒度**：专家级并行
- **通信**：设备间动态路由
- **内存**：专家分布，路由器复制
- **用例**：具有多个专家的 MoE 模型

**优势：**

- 支持扩展到数百个专家
- 稀疏激活减少每 token 计算量
- 跨 GPU 的自然负载均衡
- 大型专家集的内存效率

**挑战：**

- 如果某些专家使用更频繁，会出现负载不均
- token 路由的全对全通信开销
- 复杂故障处理（专家不可用）
- 路由器网络成为瓶颈

**PD 分离上下文：**

在分离式 MoE 服务中，EP 可以不同方式应用于 prefill 和 decode worker。Prefill worker 可能使用 EP 分布专家以高效处理提示，而 decode worker 可以使用针对自回归生成优化的不同专家分布策略。

### 序列并行（SP）

**序列并行**将序列维度（输入长度）拆分到多个设备上，将 tensor 并行扩展到非 tensor 并行区域。

**工作原理：**

- 沿序列维度将输入序列分区到 GPU 上
- 应用于 LayerNorm、Dropout 和其他逐元素操作
- 与 tensor 并行结合实现内存效率
- 减少激活的内存占用

**示例：**

```text
序列 [Token 1-1024] → [GPU 1: Tokens 1-256]   → 处理 → 收集
                    → [GPU 2: Tokens 257-512]  → 处理 →
                    → [GPU 3: Tokens 513-768]  → 处理 →
                    → [GPU 4: Tokens 769-1024] → 处理 →
```

**特征：**

- **粒度**：序列长度并行
- **通信**：散播/收集操作
- **内存**：激活沿序列维度拆分
- **用例**：长上下文场景（32K+ tokens）

**优势：**

- 减少长序列的激活内存
- 支持处理更长的上下文
- 补充 tensor 并行
- 对 LayerNorm 和 Dropout 特别有效

**挑战：**

- 要求序列长度可被 GPU 数量整除
- 散播/收集带来的额外通信
- 适用性有限（并非所有操作都支持 SP）
- 与注意力机制的复杂交互

**PD 分离上下文：**

SP 对处理长上下文窗口的 prefill worker 特别相关。在处理 32K-token 提示时，SP 可以将序列分布到多个 GPU 上以减少每 GPU 内存需求。由于序列长度短（一次 1 个 token），在 decode worker 中使用较少。

### 零冗余并行（ZP）

**零冗余并行**（ZeRO）通过将优化器状态、梯度和模型参数分区到设备上来消除数据并行中的内存冗余。

**工作原理：**

- ZeRO Stage 1：仅分区优化器状态
- ZeRO Stage 2：分区优化器状态 + 梯度
- ZeRO Stage 3：分区优化器状态 + 梯度 + 参数
- 在计算期间即时收集所需参数

**示例（ZeRO Stage 3）：**

```text
模型参数分布：
[GPU 1: 参数 1-25%] → 需要时广播 → 计算
[GPU 2: 参数 26-50%] → 需要时广播 → 计算
[GPU 3: 参数 51-75%] → 需要时广播 → 计算
[GPU 4: 参数 76-100%] → 需要时广播 → 计算
```

**特征：**

- **粒度**：跨数据并行秩的参数分区
- **通信**：前向/后向传递期间的 all-gather
- **内存**：随设备数量线性扩展
- **用例**：训练大模型（推理中较少使用）

**优势：**

- 大幅减少内存（使用 N 个 GPU 时最多减少 N 倍）
- 支持训练否则无法容纳的模型
- 保持数据并行语义
- 与其他并行策略兼容

**挑战：**

- 增加通信开销
- 更复杂的实现
- all-gather 操作带来的额外延迟
- 主要为训练工作负载设计

**PD 分离上下文：**

虽然 ZeRO 主要是一种训练优化，但 ZeRO-Inference 技术可以适应分离式服务。例如，在处理许多并发序列的 decode worker 中，ZeRO 风格的参数分区可以在结合高效参数收集时减少内存占用。

### 组合并行策略

实际部署通常组合多种并行策略：

#### 示例 1：大型模型服务

```text
- 8 个数据并行副本
  - 每个副本使用 TP=4（跨 4 个 GPU 的 tensor 并行）
  - 总计：32 个 GPU
  - 配置：DP=8、TP=4
```

#### 示例 2：具有 P/D 分离的 MoE 模型

```text
Prefill Worker：
  - DP=2、TP=2、EP=4
  - 2 个副本，每个具有 2 路 tensor 并行和 4 路专家并行

Decode Worker：
  - DP=4、TP=2、EP=4
  - 4 个副本，不同的专家分布策略
```

#### 示例 3：长上下文服务

```text
- DP=4、TP=2、SP=2、PP=2
- 4 个数据并行副本
- 每个副本：2 路 tensor 并行、2 路序列并行、2 阶段流水线
- 总计：32 个 GPU
```

**选择指南：**

- **TP**：当模型无法容纳在单个 GPU 上时，与快速互连一起使用
- **DP**：用于较小模型的吞吐量扩展
- **PP**：用于极深模型或异构集群
- **EP**：对具有多个专家的 MoE 模型至关重要
- **SP**：用于长上下文场景（>16K tokens）
- **ZP**：训练或微调大型模型时

---

## 工作负载解决方案

本节介绍在 Kubernetes 环境中支持高效 PD 分离架构的特定工作负载编排解决方案。

### LWS（LeaderWorkSet）

[`LWS（LeaderWorkSet）`](https://github.com/kubernetes-sigs/lws) 是一个 Kubernetes SIG 项目，提供用于管理具有领导者-追随者关系的 Pod 组的工作负载 API。对于 P/D 分离，LWS 可以与 StatefulSet 和 Pod 资源一起使用来编排分离式推理工作负载。

**P/D 分离的关键特性：**

- **双 LWS 架构**：使用两个独立的 LWS 实例——一个用于 prefill worker，另一个用于 decode worker
- **领导者-追随者协调**：通过领导者选择实现 prefill 和 decode 阶段之间的协调
- **StatefulSet 集成**：与 StatefulSet 配合使用以实现持久存储和网络需求
- **Pod 组管理**：将 Pod 组作为 cohesive 单元进行管理和缩放

**架构模式：**

```text
Prefill LWS + StatefulSet + Pods → KV Cache → Decode LWS + StatefulSet + Pods
```

这种方法支持每个阶段的独立缩放和资源管理，同时通过 LWS 领导者-追随者模式保持协调。

### SGLang RBG

[`SGLang RBG`](https://github.com/sgl-project/rbg) 是一个 Kubernetes API（`RoleBasedGroup`），用于将多角色 AI 推理工作负载（例如 `processor` / `prefill` / `decode`）编排为一个逻辑单元。

**关键特性：**

- **基于角色的工作负载模型**：将分布式推理服务作为协调的角色组进行管理，而非孤立的 Deployment/StatefulSet
- **多种部署模式**：支持 `standalonePattern` 和 `leaderWorkerPattern`，用于单节点和多节点拓扑
- **角色级生命周期控制**：提供推出策略控制（包括 `InPlaceIfPossible` 和分区的滚动更新）
- **缩放适配器集成**：为 HPA、KEDA 和 Planner 驱动的自动缩放暴露角色级缩放端点
- **协调/Gang 调度**：支持协调策略和可选的组级 Gang 调度行为

**与 P/D 分离的集成：**

RBG 通过以下方式增强 P/D 分离：

- 为 `prefill` 和 `decode` 提供原生多角色编排
- 角色粒度的独立缩放控制
- 跨分离组件的协调推出/更新语义

**近期上游状态（截至 2026-04-15）：**

- **已合并**：[sgl-project/rbg#255](https://github.com/sgl-project/rbg/pull/255)（2026-04-03）在 `examples/inference/ecosystem/dynamo/` 下添加了 Dynamo 集成示例：`agg.yaml`、`agg-multi-nodes.yaml`、`pd-disagg.yaml`、`pd-disagg-multi-nodes.yaml` 以及依赖清单（`etcd.yaml`、`nats.yaml`）
- **进行中**：[ai-dynamo/dynamo#5457](https://github.com/ai-dynamo/dynamo/pull/5457)（开放）在 Dynamo 文档中添加了专用的 RBG 集成指南

### AIBrix StormService

[`AIBrix StormService`](https://github.com/vllm-project/aibrix/blob/fd8ddd8062602313c5f7b3b7ecbda20e845da647/docs/source/designs/aibrix-stormservice.rst) 是一个专门设计的组件，用于管理和编排 Prefill/Decode 分离式架构中推理容器的生命周期。

**关键能力：**

- **P/D 生命周期管理**：分离式 prefill 和 decode 容器生命周期的专用编排
- **多模式支持**：管理各种部署模式，包括：
  - Prefill/Decode 分离
  - Tensor 并行（TP）
  - 流水线并行（PP）
  - 单 GPU 模型部署
- **容器编排**：对推理容器部署和缩放提供细粒度控制
- **资源协调**：确保分离组件间的适当资源分配和协调

**StormService 架构：**

StormService 充当专门的控制器：

1. 管理 prefill 和 decode 容器的部署
2. 协调分离组件间的资源分配
3. 处理生命周期事件（缩放、更新、故障）
4. 优化不同并行模式下的资源利用率

这实现了企业级 P/D 分离，具有稳健的生命周期管理和多租户支持。

### Volcano Kthena

[`Volcano Kthena`](https://github.com/volcano-sh/kthena) 是一个 Kubernetes 原生 LLM 推理平台，为在生产环境中部署和管理大型语言模型提供全面基础设施。作为 Volcano 生态系统的一部分，Kthena 为 LLM 推理工作负载带来企业级能力。**最新版本：v0.3.0** 确立了 Kthena 作为稳健可扩展 AI 推理平台的地位，对 P/D 分离进行了重大增强。

**关键能力：**

- **Kubernetes 原生架构**：与 Kubernetes 深度集成，实现无缝工作负载编排和资源管理
- **生产就绪推理**：用于部署和管理大规模 LLM 推理的企业级平台
- **Volcano 集成**：利用 Volcano 的高级调度能力实现最优资源分配
- **工作负载编排**：推理工作负载的全面生命周期管理，包括缩放和故障处理

**v0.3.0 P/D 分离新特性：**

- **LeaderWorkerSet 支持**：与 LeaderWorkerSet（LWS）API 的原生集成，支持具有领导者-工作者拓扑的分布式推理工作负载的精细管理，简化 P/D 分离部署
- **网络拓扑感知调度**：使用 Volcano 的 `subGroupPolicy` 功能，在角色级别对 Gang 调度和网络拓扑感知进行细粒度控制。通过在网络邻近节点（同一交换机/机架）上共置 prefill 和 decode 实例，最小化角色间通信延迟（对 P/D 分离至关重要）。需要 Volcano v1.14+
- **角色级 Gang 调度**：确保属于特定角色（例如 prefill-0）的所有 Pod 作为原子单元一起调度，防止可能破坏分布式推理工作负载的部分部署
- **ModelServing 版本控制**：原生版本控制系统，支持基于分区的更新，用于金丝雀部署和回滚能力
- **路由器可观测性**：全面的可观测性框架，具有调试端口（默认 15000）、用于监控延迟/吞吐量的详细指标以及用于生产可靠性的 E2E 测试套件

**与 P/D 分离的集成：**

Kthena 通过以下方式为 P/D 分离提供基础设施支持：

- 分离式架构的原生 Kubernetes 工作负载管理
- 具有拓扑感知的 prefill 和 decode 组件的协调调度
- 跨分离推理阶段的资源优化
- 与 Volcano 的批量调度和 `subGroupPolicy` 集成，实现高效的拓扑感知工作负载放置
- LeaderWorkerSet 集成，管理分布式推理中复杂的领导者-工作者关系

**生产优势：**

v0.3.0 使 Kthena 特别适合构建具有 P/D 分离需求的生产 LLM 推理平台的机构：

- 网络拓扑感知显著减少 prefill 和 decode 实例间的通信开销
- 角色级 Gang 调度确保分布式工作负载的原子化部署
- 全面的可观测性支持生产监控和调试
- 原生版本控制支持安全的金丝雀部署和回滚

### llm-d

[`llm-d`](https://github.com/llm-d/llm-d) 是一个生产就绪的 LLM 推理平台，使用双 LeaderWorkSet（LWS）架构实现 Prefill-Decode 分离。作为 Kubernetes 上 P/D 分离的参考实现，llm-d 展示了编排分离式推理工作负载的最佳实践。

**关键架构：**

- **双 LWS 设计**：使用两个独立的 LeaderWorkSet 实例：
  - 一个 LWS 用于 prefill worker
  - 一个 LWS 用于 decode worker
- **KV 缓存传输**：实现 prefill 和 decode 阶段间的高效 KV 缓存传输
- **LMCache 集成**：支持 LMCache 进行 KV 缓存卸载和管理
- **路由 Sidecar**：包含 [`llm-d 路由 sidecar`](https://github.com/llm-d/llm-d-routing-sidecar) 用于智能请求路由和缓存优化

**P/D 分离实现：**

llm-d 的双 LWS 架构支持：

- prefill 和 decode 工作负载的独立缩放
- 每个阶段的优化资源分配
- 通过专用 prefill worker 减少 TTFT
- 使用隔离的 decode worker 改善 decode 吞吐量
- 跨分离组件的高效 KV 缓存管理

**架构模式：**

```text
客户端 → 路由 Sidecar → Prefill LWS → KV 缓存（LMCache）→ Decode LWS → 响应
```

此架构展示了平衡性能、可扩展性和运维简便性的生产级 P/D 分离实现。

---

## 项目支持状态

### NVIDIA Dynamo

[`Dynamo`](https://github.com/ai-dynamo/dynamo) 是 NVIDIA 的高性能 LLM 推理引擎，为企业级服务能力提供全面的 Prefill-Decode 分离支持。

**P/D 分离支持：**

- **已支持**：Dynamo 已实现 P/D 分离以获得增强的性能和资源效率
- **多节点架构**：在 Kubernetes 部署中，Dynamo 支持基于领导者-工作者的拓扑，社区已添加用于 Dynamo 运行时编排的 RoleBasedGroup 示例
- **设计文档**：[分离式服务](https://github.com/ai-dynamo/dynamo/blob/main/docs/design_docs/disagg_serving.md)——分离 prefill 和 decode 阶段的详细设计
- **特性对比**：[特性支持对比](https://github.com/ai-dynamo/dynamo/blob/6deeecb1d6a9f4eb1770b4272bfa85a4b6226e0a/deploy/helm/README.md#feature-support-comparison)——展示与其他特性并列的分离能力

**关键实现：**

- 支持分离式 prefill 和 decode 服务 [#998](https://github.com/ai-dynamo/dynamo/pull/998)
- 增强的分离特性 [#1511](https://github.com/ai-dynamo/dynamo/pull/1511)
- 持续统一工作 [#1728](https://github.com/ai-dynamo/dynamo/issues/1728)
- RoleBasedGroup 集成指南（开放）[#5457](https://github.com/ai-dynamo/dynamo/pull/5457)
- RoleBasedGroup Dynamo 示例已合并到配套项目 [sgl-project/rbg#255](https://github.com/sgl-project/rbg/pull/255)

**架构：**

Dynamo 的分离实现支持：

- prefill 和 decode 工作负载的独立缩放
- GPU 集群间的优化资源利用率
- 在多节点场景中使用 LWS 的协调工作负载管理
- 与 Kubernetes 集成的企业就绪部署

### vLLM production stack

- vLLM 社区正在探索 [production-stack PR #340](https://github.com/vllm-project/production-stack/pull/340) 中更深层次的原生集成。

### AIBrix

- WIP issue：[在推理网关中添加 Prefill/Decode 分离支持](https://github.com/vllm-project/aibrix/issues/1223) 和 [#958](https://github.com/vllm-project/aibrix/issues/958)。

### KServe

[`KServe`](https://github.com/kserve/kserve) 是一个 CNCF Incubating 项目，提供 Kubernetes 原生平台用于大规模部署和服务机器学习模型。KServe 通过其 LLMInferenceService CRD 和与 llm-d 的集成支持 LLM 推理和 P/D 分离。

**当前能力：**

- **LMCache 集成**：基于 LMCache 的 KV 缓存卸载，改善 TTFT 和吞吐量
- **分块 Prefill**：支持分块 prefill 以优化内存使用并减少延迟尖峰
- **LLMInferenceService CRD**：用于统一 LLM 推理服务管理的自定义资源定义，原生支持 vLLM 和 LMCache

**与 llm-d 的 P/D 分离：**

KServe 通过其与 [`llm-d`](https://github.com/llm-d/llm-d) 的集成实现生产级 P/D 分离：

- **双 LWS 架构**：llm-d 基于 KServe 的 InferenceService 为 prefill 和 decode worker 部署独立的 LeaderWorkerSet
- **KV 缓存传输**：LMCache 处理 prefill 和 decode 阶段间的高效 KV 缓存移动
- **路由 Sidecar**：智能的缓存感知请求路由到 prefill 或 decode worker
- **原生 Kubernetes 操作**：通过 KServe 现有能力实现金丝雀部署、scale-to-zero 和流量分割

完整联合架构介绍请参见 [云原生 AI 推理的最佳实践：使用 KServe 和 llm-d](https://kserve.github.io/website/blog/cloud-native-ai-inference-kserve-llm-d)。

**架构：**

```text
客户端 → KServe InferenceService / Gateway
              ↓
    路由 Sidecar（缓存感知）
     ↙                   ↘
Prefill LWS          Decode LWS
(KV 构建)  →→→→  (KV 通过 LMCache 重用)
```

**统一 LLM 推理服务 API：**

- P/D 分离的声明式配置
- prefill 和 decode 服务的自动编排
- 内置 KV 缓存管理和传输
- 与 KServe 推理图集成，实现复杂服务模式

### LMCache

LMCache 是一个 LLM 服务引擎扩展，旨在减少 TTFT 并提高吞吐量，特别是在长上下文场景下。

- LMCache 在 vLLM production stack、llm-d 和 KServe 中得到支持。
- 对非前缀 KV 缓存的稳定支持。

---

## 扩展 P/D 工作负载

扩展分离式 Prefill/Decode 工作负载面临与传统单体推理部署不同的独特挑战。本节探讨标准自动缩放方法的局限性，并介绍用于优化 P/D 配置的工具。

### 传统自动缩放的挑战

传统的 Kubernetes 自动缩放机制如 HPA 和 KPA 在应用于 P/D 分离工作负载时面临显著限制：

**关键挑战：**

- **独立阶段特征**：Prefill 和 decode 阶段具有独特的资源利用模式和性能特征。标准自动缩放器无法基于其独特指标（prefill 的 TTFT、decode 的 TPOT）独立优化每个阶段。
- **复杂配置空间**：分离式部署需要关于以下方面的决策：
  - prefill worker 与 decode worker 的数量
  - 每个阶段的并行策略（tensor 并行、流水线并行）
  - 为每个阶段优化的批处理大小
  - 跨阶段的 GPU 分配
- **相互依赖的缩放**：Prefill 和 decode 组件必须协调缩放以维持最优吞吐量和延迟。不考慮另一方而缩放一个组件可能造成瓶颈或浪费资源。
- **SLA 感知**：传统自动缩放器不了解 LLM 特定的 SLA 目标，如 TTFT 和 TPOT，而这些对于分离式服务中的用户体验至关重要。
- **资源异构性**：P/D 工作负载通常受益于异构 GPU 分配（例如 decode 比 prefill 需要更多 GPU），标准自动缩放器不会优化这一点。

**HPA/KPA 为何不足：**

- **反应性而非预测性**：它们对当前指标做出反应，而非主动针对工作负载特性进行优化
- **无配置优化**：它们缩放副本数但不优化底层配置（并行度、批处理大小等）
- **单一指标关注**：它们通常基于 CPU/GPU 利用率或请求率缩放，忽略 TTFT、TPOT 和吞吐量的复杂相互作用

### 使用 AIConfigurator 进行配置优化

NVIDIA 的 [`AIConfigurator`](https://github.com/ai-dynamo/aiconfigurator) 通过离线优化分离式部署配置来解决这些挑战。它帮助在部署前确定最优配置，而不是依赖反应式自动缩放。

**AIConfigurator 工作原理：**

AIConfigurator 搜索分离式服务部署的配置空间：

1. **建模 LLM 推理性能**：使用为目标硬件收集的数据估计不同配置的执行时间
2. **配置空间搜索**：评估数千种组合，包括：
   - prefill 和 decode worker 的数量
   - 并行策略（TP、PP、DP）
   - 每个阶段的批处理大小
   - GPU 分配模式
3. **SLA 约束优化**：找到满足 TTFT 和 TPOT 目标的同时最大化吞吐量的配置
4. **帕累托前沿分析**：识别吞吐量、延迟和资源利用率之间的权衡

**关键特性：**

- **xPyD 配置规划**：确定最优副本配置，其中每个副本包含 x 个 prefill worker 和 y 个 decode worker
- **硬件特定优化**：支持各种 GPU 类型（H100、H200、B200 等），具有收集的性能数据
- **量化支持**：评估每个组件的不同量化策略（FP16、FP8、INT4 等）
- **部署文件生成**：为 Dynamo 部署生成可直接使用的配置文件

**使用示例：**

```bash
# 为 Qwen3-32B 在 32 个 H200 GPU 上寻找最优配置
# SLA 目标：TTFT ≤ 300ms、TPOT ≤ 10ms
aiconfigurator cli default \
  --model QWEN3_32B \
  --total_gpus 32 \
  --system h200_sxm \
  --isl 4000 \
  --osl 500 \
  --ttft 300 \
  --tpot 10
```

**对 P/D 工作负载的优势：**

- **知情缩放决策**：提供数据驱动的见解，指导如何缩放 prefill 与 decode 组件
- **配置模板**：生成可作为自动缩放策略基线的优化起始配置
- **性能预测**：在部署前估计吞吐量和延迟，减少试错
- **资源效率**：识别在满足 SLA 目标的同时最大化 GPU 利用率的配置

**与自动缩放的集成：**

虽然 AIConfigurator 是一个离线工具，但其输出可以为自动缩放策略提供信息：

- 使用 AIConfigurator 为不同负载级别建立基线配置
- 配置自动缩放器在基于工作负载模式的 AIConfigurator 优化配置之间进行转换
- 利用 AIConfigurator 的见解设置适当的缩放阈值和 prefill 与 decode worker 之间的副本比例

这种方法将 AIConfigurator 的预测优化与 Kubernetes 自动缩放的反应能力相结合，实现更有效的 P/D 工作负载管理。

### Kubernetes 中基于 SLA 的调度

Kubernetes 正在通过 [KEP-5471：用于基于阈值放置的扩展容忍操作符](https://github.com/kubernetes/enhancements/pull/5473)引入原生基于 SLA 的调度能力。此增强使 Pod 能够在容忍中使用数值比较表达 SLA 要求，为 SLA 感知调度提供 Kubernetes 原生机制。

**关键能力：**

- **数值比较操作符**：扩展容忍以支持 `Lt`（小于）和 `Gt`（大于）操作符，用于匹配具有数值的节点污点
- **SLA 阈值基放置**：允许 Pod 指定最小 SLA 要求（例如"仅在 SLA > 95% 的节点上调度"）
- **驱逐支持**：与 `NoExecute` 污点效果结合，在节点 SLA 低于阈值时自动驱逐 Pod
- **特性门控**：`TaintTolerationComparisonOperators`（Kubernetes v1.35 中为 alpha）

**与 P/D 工作负载的相关性：**

对于分离式 P/D 部署，基于 SLA 的调度提供了对工作负载放置的细粒度控制：

- **Prefill 阶段**：可以要求高 SLA 节点以最小化 TTFT 并确保首 token 生成的一致延迟
- **Decode 阶段**：可能容忍较低 SLA 节点以优化成本，同时保持可接受的 TPOT
- **异构集群**：支持混合按需（高 SLA）和 spot（低 SLA）节点，具有基于可靠性要求的自动工作负载引导

**P/D 工作负载的使用示例：**

```yaml
# 具有 95% SLA 污点的高 SLA 按需节点
apiVersion: v1
kind: Node
metadata:
  name: ondemand-gpu-node-1
spec:
  taints:
  - key: node.kubernetes.io/sla
    value: "950"  # 95.0% SLA
    effect: NoExecute
---
# Prefill Pod 需要 SLA > 95%，具有驱逐支持
apiVersion: v1
kind: Pod
metadata:
  name: prefill-worker
spec:
  tolerations:
  - key: node.kubernetes.io/sla
    operator: Gt
    value: "950"
    effect: NoExecute
    tolerationSeconds: 30  # 驱逐前的宽限期
  containers:
  - name: prefill
    image: inference-engine:latest
    args: ["--mode=prefill"]
---
# Decode Pod 容忍较低 SLA 以节省成本
apiVersion: v1
kind: Pod
metadata:
  name: decode-worker
spec:
  tolerations:
  - key: node.kubernetes.io/sla
    operator: Gt
    value: "800"  # 80% SLA 可接受
    effect: NoExecute
    tolerationSeconds: 60
  containers:
  - name: decode
    image: inference-engine:latest
    args: ["--mode=decode"]
```

**与 AIConfigurator 的集成：**

基于 SLA 的调度通过提供运行时放置控制来补充 AIConfigurator：

1. **离线规划**：AIConfigurator 确定不同 SLA 层级的最优 xPyD 配置和资源分配
2. **运行时执行**：基于 SLA 的容忍确保工作负载被放置到满足其可靠性要求的节点上
3. **动态适应**：具有 `NoExecute` 效果的污点在节点 SLA 降级时触发自动 Pod 驱逐，支持优雅故障转移
4. **成本优化**：将 AIConfigurator 的吞吐量优化与基于 SLA 的放置相结合，平衡性能和基础设施成本

这种组合支持具有可预测性能、自动故障处理和跨异构 GPU 集群高效资源利用的生产级 P/D 部署。

---

## 参考资料

- <https://github.com/kubernetes-sigs/lws>
- <https://github.com/llm-d/llm-d>
- <https://github.com/llm-d/llm-d-routing-sidecar>
- <https://github.com/sgl-project/rbg>
- <https://github.com/sgl-project/rbg/pull/255>
- <https://github.com/sgl-project/rbg/tree/main/examples/inference/ecosystem/dynamo>
- <https://github.com/volcano-sh/kthena>
- <https://github.com/ai-dynamo/dynamo>
- <https://github.com/ai-dynamo/dynamo/pull/5457>
- <https://github.com/ai-dynamo/aiconfigurator>
- <https://github.com/kubernetes/enhancements/pull/5473> - KEP-5471：用于基于阈值放置的扩展容忍操作符（基于 SLA 的调度）
- <https://github.com/vllm-project/vllm>
- <https://github.com/vllm-project/production-stack>
- <https://github.com/vllm-project/aibrix>
- <https://github.com/kserve/kserve>
- <https://kserve.github.io/website/blog/cloud-native-ai-inference-kserve-llm-d> - 云原生 AI 推理的最佳实践：使用 KServe 和 llm-d
- <https://github.com/LMCache/lmcache>
- DistServe (OSDI'24)：<https://www.usenix.org/system/files/osdi24-zhong-yinmin.pdf>

**部分内容由 ChatGPT 生成。使用时请谨慎。这些是个人学习笔记。**
