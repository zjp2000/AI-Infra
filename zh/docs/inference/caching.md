---
status: Active
maintainer: pacoxu
last_updated: 2025-10-29
tags: inference, caching, kv-cache, prefix-caching, llm-optimization
canonical_path: zh/docs/inference/caching.md
---

# LLM 推理中的缓存

本文档涵盖大型语言模型（LLM）推理中的缓存机制，重点关注通过跨请求重用已计算的键值（KV）缓存来优化性能的前缀缓存方法。

有关 KV 缓存之外的 Agent 内存和长期上下文存储信息，请参见 [AI Agent 的内存、上下文与数据库](./memory-context-db.md)。

## 目录

- [什么是前缀缓存](#what-is-prefix-caching)
- [KV 缓存卸载与存储层次](#kv-cache-offloading-and-storage-hierarchy)
- [共享前缀缓存](#shared-prefix-caching)
- [独立前缀缓存](#independent-prefix-caching)
- [分布式 KV 缓存解决方案](#distributed-kv-cache-solutions)
- [KV 缓存传输的网络加速](#network-acceleration-for-kv-cache-transfer)
- [下一代 KV 缓存硬件](#next-generation-hardware-for-kv-cache)
- [KV 缓存解决方案对比](#kv-cache-solutions-comparison)
- [项目实现](#project-implementations)
- [参考资料](#references)

---

## 什么是前缀缓存

LLM 推理中的前缀缓存通过存储和重用先前计算的键值（KV）缓存数据来优化性能。当多个请求共享公共前缀（例如系统提示、聊天历史）时，这些共享部分的计算可以缓存和重用，显著减少首 token 时间（TTFT）并提高吞吐量。

**优势：**

- 减少重复前缀的计算
- 降低首 token 时间（TTFT）
- 更高的整体吞吐量
- 更好的资源利用率
- 服务场景中的成本降低

---

## KV 缓存卸载与存储层次

随着 LLM 上下文窗口的不断扩展，KV 缓存存储需求呈指数级增长。为管理这一增长，现代推理系统采用层次化存储方法，根据访问模式和性能要求将 KV 缓存数据卸载到多个存储层。

![dynamo](../../diagrams/dynamo-memory.png)

### 存储层次

KV 缓存存储层次通常遵循以下模式，按访问速度和成本排序：

1. **VRAM（GPU 内存）**：最快访问，最高成本，容量有限
   - 推理期间活跃 KV 缓存的主要存储
   - 注意力计算的最低延迟
   - 典型容量：每 GPU 16GB - 80GB

2. **RAM（主机内存）**：快速访问，中等成本，容量较大
   - 温 KV 缓存的中间存储
   - 同一节点上多个 GPU 共享
   - 典型容量：每节点 256GB - 2TB

3. **SSD（固态硬盘）**：中等访问速度，较低成本
   - 需要高速访问的冷 KV 缓存存储
   - 系统重启后持久化
   - 典型容量：每节点 1TB - 16TB

4. **本地磁盘（HDD）**：较慢访问，低成本，高容量
   - 不常访问的 KV 缓存的归档存储
   - 长期持久化
   - 典型容量：每节点 4TB - 100TB

5. **远程对象存储**：基于网络，最低成本，无限扩展
   - 跨集群共享 KV 缓存的分布式存储
   - 支持跨节点 KV 缓存共享
   - 示例：S3、OSS、Ceph、MinIO

### 卸载策略

有效的 KV 缓存卸载需要智能管理：

- **自动分层**：根据访问模式在层间移动 KV 缓存
- **预取**：在需要之前从较慢层加载 KV 缓存
- **压缩**：减少存储需求和传输带宽
- **去重**：跨请求共享相同的 KV 缓存块
- **驱逐策略**：LRU、LFU 或应用感知驱逐策略

**优势：**

- 减少重复前缀的计算
- 降低首 token 时间（TTFT）
- 更高的整体吞吐量
- 更好的资源利用率
- 服务场景中的成本降低

---

## 共享前缀缓存

共享前缀缓存允许多个请求跨不同会话或用户共享公共前缀的 KV 缓存。

### NIXL

[`NIXL`](https://github.com/ai-dynamo/nixl) 为 LLM 推理系统提供共享前缀缓存能力。

**关键特性：**

- 跨请求前缀共享
- 与多个推理引擎集成
- 高效缓存管理

### DCN（分布式计算节点）

DCN 架构支持跨多个计算节点的分布式前缀缓存。最初来自 Red Hat OpenStack Platform，DCN 提供：

- **分布式计算节点（DCN）架构**：用于边缘用例，允许远程部署计算和存储节点，同时共享公共的集中控制面
- **战略性工作负载定位**：允许将工作负载定位到更靠近运维需求的位置以获得更高性能
- **共享缓存层**：支持跨分布式节点的前缀缓存共享

更多详情请参见 [Red Hat DCN 文档](https://docs.redhat.com/en/documentation/red_hat_openstack_platform/17.1/html/deploying_a_distributed_compute_node_dcn_architecture/understanding_dcn)。

> **注意**：此处引用的 DCN（分布式计算节点）架构主要是 OpenStack 云基础设施概念。虽然分布式缓存原则可能适用于 LLM 推理场景，但本文档链接指向的是通用边缘计算部署模式，而非 LLM 特定的缓存实现。需要进一步研究以验证特定 DCN 在 LLM 推理前缀缓存中的应用。

---

## 分布式 KV 缓存解决方案

分布式 KV 缓存系统跨多个节点管理 KV 缓存，支持高效的共享和卸载到远程存储。这些解决方案对于扩展 LLM 推理以处理大上下文窗口和高吞吐量服务场景至关重要。

### Mooncake

[`Mooncake`](https://github.com/kvcache-ai/Mooncake) 是一个分布式 KV 缓存服务，旨在缓解 LLM 推理系统中的存储压力。它解决了随着模型上下文窗口扩展而呈指数级增长的 KV 缓存存储需求。

**关键特性：**

- **分布式 KV 缓存管理**：跨多个节点共享 KV 缓存
- **L3 KV 缓存存储**：提供超越 GPU 和主机内存的额外缓存层
- **高吞吐量传输**：针对跨节点数据移动优化
- **与推理引擎集成**：与 SGlang 和其他推理框架配合使用
- **持久存储**：支持服务重启后的 KV 缓存持久化

**架构：**

Mooncake 实现了将缓存存储与计算节点分离的分布式架构：

- **缓存存储节点**：用于存储 KV 缓存数据的专用节点
- **计算节点**：生成和消费 KV 缓存的推理节点
- **元数据管理**：跟踪 KV 缓存位置和访问模式
- **传输优化**：最小化跨节点数据传输开销

**性能考虑：**

虽然 Mooncake 和类似的分布式 KV 缓存服务减少了存储压力，但跨节点数据传输仍然是主要的性能瓶颈。网络延迟和带宽限制可能影响整体系统吞吐量，特别是在长上下文场景中。

### 下一代硬件集成

现代分布式 KV 缓存解决方案越来越多地利用下一代硬件能力来克服网络瓶颈：

- **超高带宽（UB）互连**：400Gbps+ 带宽用于高速跨节点通信
- **CXL（Compute Express Link）**：具有低延迟的远程内存内存语义访问
- **灵桥超节点共享内存**：跨节点的硬件加速共享内存

这些硬件进步与分布式 KV 缓存管理需求高度兼容，提供：

- 大型 KV 缓存传输的高带宽
- 实时推理工作负载的低延迟
- 匹配 KV 缓存使用的内存语义访问模式
- 跨分布式节点的统一内存视图

**行业倡议：**

中国 AI 基础设施社区正在积极探索将下一代硬件与分布式 KV 缓存服务集成的解决方案。例如，已有提案将灵桥超节点共享内存能力集成到 Mooncake 中，利用下一代硬件特性为推理服务提供通用加速。

### 其他分布式 KV 缓存解决方案

还有几个其他项目提供分布式 KV 缓存能力：

- **AIBrix**：在 vLLM 生态系统中包含分布式推理和 KV 缓存能力
- **Vineyard**：被 AIBrix 用于分布式 KV 缓存管理
- **Ray 分布式运行时**：为多节点推理提供分布式内存管理

---

## 独立前缀缓存

独立前缀缓存系统为每个会话或服务独立管理 KV 缓存，在提供隔离性的同时，仍优化单个上下文内的重复模式。

### LMCache

[`LMCache`](https://github.com/LMCache/lmcache) 是一个 LLM 服务引擎扩展，旨在减少 TTFT 并提高吞吐量，特别是在长上下文场景中。

![nixl-design](../../diagrams/nixl-design.png)

- GPU-GPU 传输
- GPU-CPU 卸载
- CPU-CPU 传输
- 存储卸载

**关键特性：**

- 非前缀 KV 缓存支持
- 与 vLLM production stack、llm-d 和 KServe 集成
- 针对长上下文场景优化
- 对各种缓存模式的稳定支持

### Dynamo KVBM

Dynamo 的 KV Block Manager（KVBM）在 Dynamo 生态系统中提供高效的 KV 缓存管理。

**关键特性：**

- 基于块的 KV 缓存管理
- 与 Dynamo 的分离式服务架构集成
- 高效内存利用

架构详情请参见 [KVBM 架构文档](https://github.com/ai-dynamo/dynamo/blob/main/docs/architecture/kvbm_architecture.md)。

### 主机内存缓存

主机内存缓存利用系统 RAM 存储 KV 缓存数据，提供：

- 相比 GPU 内存更低的成本
- 更大的缓存容量
- 跨 GPU 共享能力
- 模型重载后的持久性

---

## KV 缓存传输的网络加速

随着分布式 KV 缓存系统跨节点移动数据，网络性能变得至关重要。现代推理系统利用高性能网络技术来最小化传输延迟和最大化吞吐量。

### RDMA（远程直接内存访问）

RDMA 支持从一台计算机直接访问另一台计算机的内存，无需涉及操作系统，减少延迟和 CPU 开销。

**KV 缓存传输的优势：**

- **零拷贝传输**：无需 CPU 参与的直接内存到内存复制
- **内核旁路**：消除操作系统开销
- **低延迟**：亚微秒级内存访问延迟
- **高带宽**：100Gbps 到 400Gbps+ 吞吐量
- **CPU 卸载**：释放 CPU 资源用于推理计算

**RDMA 技术：**

- **InfiniBand**：专为高性能计算构建
  - 最低延迟和最高带宽
  - 常见于 AI/ML 集群和数据中心
  - 典型带宽：200Gbps（HDR）、400Gbps（NDR）、800Gbps（XDR，新兴）

- **RoCE（基于融合以太网的 RDMA）**：以太网网络上的 RDMA
  - 比 InfiniBand 更具成本效益
  - 利用现有以太网基础设施
  - 典型带宽：100Gbps、200Gbps、400Gbps

- **iWARP**：互联网广域 RDMA 协议
  - 在标准 TCP/IP 网络上工作
  - 更适合 WAN 场景

**实施考虑因素：**

要实现有效的基于 RDMA 的 KV 缓存传输：

- 使用 verbs API 进行直接 RDMA 操作
- 为多个推理请求实现连接池
- 考虑频繁访问缓存的内存注册开销
- 优化缓冲区管理以最小化复制
- 监控网络拥塞并调整传输模式

### InfiniBand 用于 LLM 推理

InfiniBand 特别适合分布式 LLM 推理，原因如下：

- **低延迟**：对具有严格 SLO 要求的实时推理至关重要
- **高带宽**：支持长上下文的大型 KV 缓存传输
- **原生 RDMA 支持**：针对直接内存访问进行硬件优化
- **可扩展性**：支持数千节点的大规模集群
- **可靠性**：内置错误检测和恢复机制

**典型部署：**

在生产 LLM 服务集群中：

- InfiniBand 连接数据中心内的 GPU 节点
- 支持跨分布式推理工作者的高效 KV 缓存共享
- 支持分离式 prefill-decode 架构
- 促进模型并行和 tensor 并行

---

## 下一代 KV 缓存硬件

超越传统 RDMA 和 InfiniBand，新兴的硬件技术正通过内存语义访问和超高带宽改变分布式 KV 缓存管理。

### CXL（Compute Express Link）

CXL 是一个开放标准互连，支持 CPU、内存、加速器和其他设备之间的高速高效通信。

**KV 缓存的关键能力：**

- **内存池化**：跨多个主机的共享内存池
- **缓存一致性**：硬件管理的连贯内存访问
- **低延迟**：远程内存接近本地内存访问速度
- **带宽**：每链路 64-256 GB/s（CXL 2.0/3.0）
- **内存扩展**：动态扩展可用内存容量

**分布式 KV 缓存优势：**

- 以最小延迟将远程 KV 缓存视为本地内存
- 硬件管理的一致性消除软件开销
- 跨分布式节点的灵活内存分配
- 无需完全系统升级的经济高效内存扩展

### 超高带宽（UB）互连

下一代互连提供前所未有的数据传输带宽：

- **400GbE 及更高**：下一代以太网 400Gbps、800Gbps
- **800G InfiniBand（XDR）**：新兴 InfiniBand 标准
- **自定义 AI 互连**：供应商特定的高带宽解决方案

**对 KV 缓存性能的影响：**

更高带宽互连支持：

- 长上下文场景的更快速 KV 缓存传输
- 传输时间占总推理时间的百分比降低
- 支持更大批处理大小的分布式缓存
- 高效的预取和后台缓存移动

### 灵桥超节点共享内存

灵桥超节点技术提供跨物理分布式节点的硬件加速共享内存，创建统一的内存空间。

**关键特性：**

- **内存语义访问**：像访问本地内存一样访问远程内存
- **硬件加速**：用于内存虚拟化的专用硬件
- **低延迟**：针对实时推理要求优化
- **高带宽**：支持大规模 KV 缓存传输

**与分布式 KV 缓存的集成：**

已有提案将灵桥超节点能力与 Mooncake 等服务集成，支持：

- 无需显式数据传输的无缝跨节点 KV 缓存共享
- 降低分布式缓存管理的软件复杂度
- 推理加速的硬件优化性能
- 下一代推理基础设施的通用解决方案

---

## KV 缓存解决方案对比

下表对比了三大 KV 缓存解决方案：NIXL、LMCache 和 Mooncake。

| 特性 | NIXL | LMCache | Mooncake |
|---------|------|---------|----------|
| **项目类型** | 共享前缀缓存 | 独立前缀缓存 | 分布式 KV 缓存服务 |
| **主要用例** | 跨请求前缀共享 | 长上下文 TTFT 减少 | 分布式存储管理 |
| **缓存范围** | 跨请求/用户共享 | 每会话/服务 | 集群范围分布式 |
| **集成** | 多个推理引擎 | vLLM、llm-d、KServe | SGlang、推理框架 |
| **存储层** | L2 缓存（前缀共享） | L2 缓存（会话缓存） | L3 缓存（分布式存储） |
| **分布** | 单/多节点 | 单/多节点 | 多节点分布式 |
| **上下文窗口** | 短到中等 | 长上下文优化 | 非常长的上下文 |
| **RDMA 支持** | 可能 | 可能 | 专为高速网络设计 |
| **硬件加速** | 标准 | 标准 | 下一代硬件就绪（CXL、UB） |
| **持久性** | 内存中 | 可配置 | 持久分布式存储 |
| **最适合** | 公共前缀场景 | 长上下文推理 | 大规模分布式推理 |
| **成熟度** | 生产就绪 | 生产就绪 | 活跃开发中 |

**选择合适解决方案：**

- **使用 NIXL 当**：您有许多请求共享公共前缀时（例如系统提示、公共上下文）
- **使用 LMCache 当**：您需要在长上下文场景中优化 TTFT，且在 vLLM 生产栈中有稳定集成
- **使用 Mooncake 当**：您需要跨多个节点的分布式 KV 缓存管理以实现极致扩展
- **组合使用**：许多生产系统结合多种方法（例如 LMCache 用于本地缓存 + Mooncake 用于分布式存储）

---

## 项目实现

### SGlang 存储选项

[`SGlang`](https://github.com/sgl-project/sglang) 为 KV 缓存管理提供多个存储后端：

- **hf3fs deepseek HF3FS**：基于 HuggingFace 的分布式文件系统
- **mooncake_store**：与 Mooncake 集成，用于 L3 KV 缓存分布式存储
  - 支持跨节点 KV 缓存共享
  - 支持服务重启后的持久存储
  - 针对大规模推理部署优化
- **nixl**：与 NIXL 集成，用于共享前缀缓存

存储实现可在 [SGlang 内存缓存存储目录](https://github.com/sgl-project/sglang/tree/9b5f0f64f52033f5965d5b593df5df45c9be8c24/python/sglang/srt/mem_cache/storage) 中找到。

### llm-d 路由 Sidecar

[`llm-d 路由 sidecar`](https://github.com/llm-d/llm-d-routing-sidecar) 支持多个缓存代理：

- **NIXL 集成**：通过 NIXL v2 连接器
- **LMCache 支持**：直接与 LMCache 集成
- **路由优化**：基于缓存命中率的智能路由

实现细节可在 [NIXL v2 连接器](https://github.com/llm-d/llm-d-routing-sidecar/blob/d84279a5eecfe098b7250df731576c69aa21505c/internal/proxy/connector_nixlv2.go) 中找到。

---

## 参考资料

- [NIXL GitHub 仓库](https://github.com/ai-dynamo/nixl)
- [LMCache GitHub 仓库](https://github.com/LMCache/lmcache)
- [Mooncake GitHub 仓库](https://github.com/kvcache-ai/Mooncake)
- [Dynamo KVBM 架构](https://github.com/ai-dynamo/dynamo/blob/main/docs/architecture/kvbm_architecture.md)
- [SGlang 内存缓存存储](https://github.com/sgl-project/sglang/tree/9b5f0f64f52033f5965d5b593df5df45c9be8c24/python/sglang/srt/mem_cache/storage)
- [llm-d 路由 Sidecar](https://github.com/llm-d/llm-d-routing-sidecar)
- [Red Hat DCN 文档](https://docs.redhat.com/en/documentation/red_hat_openstack_platform/17.1/html/deploying_a_distributed_compute_node_dcn_architecture/understanding_dcn)
- [InfiniBand 贸易协会](https://www.infinibandta.org/)
- [CXL 联盟](https://www.computeexpresslink.org/)
- [RDMA 编程指南](https://www.rdmamojo.com/)

**部分内容基于所提供的 GitHub 参考资料整理。实施前请核实技术细节和当前项目状态。**
