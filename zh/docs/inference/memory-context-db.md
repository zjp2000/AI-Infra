---
status: Active
maintainer: pacoxu
last_updated: 2025-10-29
tags: inference, memory, context, ai-agents, database
canonical_path: zh/docs/inference/memory-context-db.md
---

# AI Agent 的内存、上下文与数据库

本文档涵盖 AI Agent 和 LLM 推理系统的内存管理、上下文处理和数据库解决方案，重点关注短期和长期记忆策略。

## 目录

- [概述](#overview)
- [短期记忆 vs 长期记忆](#short-term-vs-long-term-memory)
- [Agent 内存架构](#agent-memory-architecture)
- [内存后端](#memory-backends)
- [向量数据库](#vector-databases)
- [状态管理解决方案](#state-management-solutions)
- [Agent 特定数据库系统](#agent-specific-database-systems)
- [一致性与垃圾回收](#consistency-and-garbage-collection)
- [项目实现](#project-implementations)
- [参考资料](#references)

---

## 概述

内存和上下文管理是 AI Agent 和 LLM 应用的关键组件。随着 Agent 变得越来越复杂，它们需要高效的机制来跨会话存储、检索和管理短期对话上下文和长期知识。

**关键挑战：**

- 管理不断增长的上下文窗口和 KV 缓存
- 平衡短期聊天历史与长期知识
- 确保跨数千个并发 Agent 的一致性
- 非活跃 Agent 会话的垃圾回收
- 内存操作的 API 标准化

---

## 短期记忆 vs 长期记忆

### 短期记忆（聊天上下文）

短期记忆指活跃会话中的即时对话上下文，通常存储在 KV 缓存或工作内存中。

**特征：**

- **持续时间**：仅活跃会话（秒到小时）
- **存储**：GPU VRAM、主机 RAM 或快速缓存
- **大小**：受上下文窗口限制（通常 4K-128K tokens）
- **访问模式**：顺序、高频读写
- **持久性**：通常为易失性，会话结束后清除

**用例：**

- 多轮对话
- 会话内上下文跟踪
- 即时响应生成
- 重复前缀的提示缓存

**技术：**

- KV 缓存（参见 [LLM 推理中的缓存](./caching.md)）
- 内存缓冲区
- 会话特定存储

### 长期记忆（知识库）

长期记忆指跨会话持久化的知识，使 Agent 能够回忆过去的交互和学习到的信息。

**特征：**

- **持续时间**：跨会话持久化（天到年）
- **存储**：数据库、向量存储、对象存储
- **大小**：潜在无限（按数据库容量扩展）
- **访问模式**：随机访问，基于检索的查询
- **持久性**：具有备份和恢复的持久存储

**用例：**

- 用户偏好跟踪
- 历史对话检索
- 领域特定知识库
- 跨会话个性化
- 多 Agent 协作状态

**技术：**

- 向量数据库（Milvus、Chroma、Pinecone）
- 传统数据库（PostgreSQL、MySQL）
- 对象存储（S3、GCS、MinIO）
- 用于关系跟踪的图数据库

---

## Agent 内存架构

现代 AI Agent 通常采用结合多个存储层的分层内存架构以实现最佳性能。

### 内存层次

```text
┌─────────────────────────────────────────────────────────┐
│                     Agent 应用                            │
└─────────────────────────────────────────────────────────┘
                          │
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
   ┌──────────┐    ┌──────────┐    ┌──────────┐
   │ 短期记忆 │    │ 工作记忆 │    │ 长期记忆 │
   └──────────┘    └──────────┘    └──────────┘
        │               │               │
   ┌────────┐    ┌──────────┐    ┌──────────┐
   │KV Cache│    │ 内存状态 │    │ 向量 DB  │
   │ GPU/RAM│    │          │    │ 对象存储 │
   └────────┘    └──────────┘    └──────────┘
```

### Agent 内存的 API 形态

根据 CNCF TOC 讨论，需要标准化的 API 形态用于：

- **状态持久化**：保存和恢复 Agent 状态
- **内存检索**：查询历史交互
- **上下文注入**：向提示添加外部知识
- **内存更新**：增量更新 Agent 知识
- **垃圾回收**：清理非活跃或过期内存

**API 操作示例：**

```text
GET    /agent/{id}/memory/recent       # 获取最近对话
POST   /agent/{id}/memory/store        # 存储新记忆
GET    /agent/{id}/memory/search       # 在记忆中进行语义搜索
DELETE /agent/{id}/memory/prune        # 垃圾回收旧条目
GET    /agent/{id}/state               # 获取完整 Agent 状态
POST   /agent/{id}/state/restore       # 恢复 Agent 状态
```

---

## 内存后端

不同的后端系统满足不同的内存管理需求。

### 对象存储（S3、GCS、MinIO）

**最适合**：大型上下文窗口和对话日志的长期存储

**特征：**

- 无限可扩展性
- 冷存储经济高效
- 高持久性和可用性
- 与数据库相比访问速度较慢

**用例：**

- 对话历史归档
- 长时间运行 Agent 的检查点存储
- 备份和灾难恢复
- 合规和审计日志

### Redis（内存缓存）

**最适合**：最近状态和会话数据的快速访问

**特征：**

- 亚毫秒级延迟
- 临时或持久存储模式
- 发布/订阅用于实时更新
- 基于 TTL 的自动过期

**用例：**

- 活跃会话状态
- 最近的对话历史
- Agent 状态的分布式缓存
- 速率限制和请求跟踪

### 向量数据库

**最适合**：语义搜索和基于相似性的检索

**特征：**

- 基于嵌入的搜索
- 相似性评分
- 元数据过滤
- 水平可扩展性

**用例：**

- 语义记忆搜索
- RAG（检索增强生成）
- 相似对话检索
- 知识库查询

---

## 向量数据库实现

向量数据库对于 Agent 内存系统中的语义搜索至关重要。

### 关键项目

#### Chroma

[`Chroma`](https://github.com/chroma-core/chroma) 是一个为 AI 应用设计的开源嵌入数据库。

**关键特性：**

- 易于与 LangChain 和 LlamaIndex 集成
- 本地或服务器部署
- Python 和 JavaScript 客户端
- 元数据过滤和混合搜索

#### Milvus

<a href="https://github.com/milvus-io/milvus">`Milvus`</a> 是一个为可扩展相似性搜索构建的云原生向量数据库。

**关键特性：**

- CNCF 毕业项目
- 混合搜索（密集 + 稀疏向量）
- 多租户支持
- Kubernetes 原生部署

#### Qdrant

<a href="https://github.com/qdrant/qdrant">`Qdrant`</a> 是一个用 Rust 编写的向量相似性搜索引擎，专为高性能和生产使用而设计。

**关键特性：**

- 使用 Rust 构建，实现高性能和可靠性
- 负载过滤和混合搜索
- 分布式部署支持
- REST 和 gRPC API

#### Weaviate

<a href="https://github.com/weaviate/weaviate">`Weaviate`</a> 是一个开源向量数据库，具有内置向量化器和混合搜索功能。

**关键特性：**

- 内置文本和多模态向量化器
- GraphQL 和 REST API
- 通过 Operator 实现 Kubernetes 原生部署

#### FAISS

<a href="https://github.com/facebookresearch/faiss">`FAISS`</a>（Facebook AI 相似性搜索）是一个用于密集向量相似性搜索的本地索引库。

**关键特性：**

- 高度优化的 CPU/GPU 相似性搜索
- 多种索引类型（IVF、HNSW、PQ）
- Python 和 C++ 接口
- 适用于离线索引和本地嵌入搜索

#### pgvector

<a href="https://github.com/pgvector/pgvector">`pgvector`</a> 是一个 PostgreSQL 扩展，为 Postgres 添加向量相似性搜索功能。

**关键特性：**

- 原生 PostgreSQL 集成
- 精确和近似最近邻搜索（HNSW、IVFFlat）
- 将向量搜索与关系查询结合
- 对于已使用 Postgres 的团队部署简单

### Kubernetes Operator

专用 Operator 简化了向量数据库在 Kubernetes 上的部署：

- **Milvus Operator**：[自动化 Milvus 集群管理](https://github.com/milvus-io/milvus-operator)
- **Chroma Operator**：[管理 Chroma 部署](https://github.com/chroma-core/chroma-operator)（社区驱动，仓库已归档）
- **Weaviate Operator**：[处理 Weaviate 向量数据库集群](https://github.com/weaviate/weaviate-operator)

---

## 状态管理解决方案

### Dapr State Components

<a href="https://github.com/dapr/dapr">`Dapr`</a>（分布式应用运行时）提供跨多个后端的统一状态管理 API。

**关键特性：**

- 多后端支持（Redis、PostgreSQL、CosmosDB 等）
- 跨不同存储的一致 API
- 事务支持
- 状态加密和并发控制

**对 Agent 系统的优势：**

- 抽象后端差异
- 不同环境的轻松后端切换
- 内置重试和弹性模式
- 服务网格集成

**状态存储示例：**

- Redis：快速、临时状态
- PostgreSQL：持久、事务状态
- CosmosDB：全局分布式状态
- Azure Blob Storage：大型对象状态

### Model Context Protocol（MCP）

Model Context Protocol 旨在标准化 LLM 与包括内存系统在内的外部上下文源的交互方式。

**目标：**

- 标准化的上下文注入
- 与提供商无关的内存访问
- 高效的上下文检索
- 多模态上下文支持

---

## Agent 特定数据库系统

### memU

<a href="https://github.com/NevaMind-AI/memU">`memU`</a> 是一个专为 AI Agent 设计的内存管理解决方案。

**关键特性：**

- Agent 会话间的持久内存
- 分层内存组织
- 高效检索机制
- 与主流 Agent 框架集成

**架构：**

- 用于快速检索的内存索引
- 用于语义搜索的向量相似性
- 用于上下文过滤的元数据标记
- 自动内存整合

### DB-GPT

<a href="https://github.com/eosphoros-ai/DB-GPT">`DB-GPT`</a> 是一个实验性 AI 原生数据库框架，内置对 Agent 内存的支持。

**关键特性：**

- AI 原生数据管理
- 自然语言查询接口
- Agent 内存原语
- 知识图谱集成

**用例：**

- 具有共享知识的多 Agent 系统
- 对结构化数据的复杂推理
- 基于数据库的对话 Agent
- 企业知识管理

---

## 一致性与垃圾回收

跨数千个并发 Agent 管理状态一致性和内存清理带来了独特挑战。

### 一致性挑战

**问题领域：**

- **多 Agent 协调**：Agent 读写共享状态
- **分布式部署**：状态分布在多个节点上
- **会话交接**：在副本间传输 Agent 状态
- **并发更新**：多个请求修改同一 Agent 状态

**解决方案：**

- **乐观锁定**：基于版本号的冲突检测
- **事件溯源**：仅追加的状态变更
- **CQRS**：为可扩展性分离读写模型
- **分布式事务**：关键操作的二阶段提交

### 垃圾回收策略

自动清理非活跃或过期的 Agent 内存对系统健康至关重要。

**基于 TTL 的过期：**

- 为内存条目设置生存时间
- 过期后自动删除
- Redis 原生支持 TTL
- 适用于基于会话的状态

**引用计数：**

- 跟踪对 Agent 状态的活跃引用
- 引用计数归零时删除
- 适用于跨 Agent 的共享内存

**LRU/LFU 驱逐：**

- 最近最少使用或最不频繁使用
- 保持热数据在缓存中，驱逐冷数据
- 内存有界缓存

**手动清理：**

- 显式 Agent 会话终止
- 用户发起的数据删除
- 合规驱动的数据保留策略

**后台任务：**

- 定期扫描非活跃会话
- 批量删除旧条目
- 非高峰执行以最小化影响

---

## 项目实现

### llm-d with LMCache

<a href="https://llm-d.ai/blog/kvcache-wins-you-can-see">`KV-Cache Wins You Can See: From Prefix Caching in vLLM to Distributed Scheduling with llm-d`</a> 展示了实用的 KV 缓存管理。

**关键见解：**

- 前缀缓存大幅减少 TTFT
- 分布式调度支持高效缓存利用
- 分离式推理的生产部署策略
- 与 LMCache 集成实现跨节点缓存共享

**实现细节：**

- LMCache 集成用于持久 KV 缓存
- 路由 Sidecar 用于智能请求分发
- 缓存感知调度以最大化命中率
- 缓存性能的监控和可观测性

有关 LMCache 集成的更多详情，请参见：

- [LMCache on GKE](https://blog.lmcache.ai/2025-10-07-LMCache-on-GKE/)
- [Dynamo LMCache Integration](https://blog.lmcache.ai/2025-09-18-dynamo-lmcache/)

### CNCF TOC 关于 Agent 内存的讨论

CNCF 技术监督委员会正在 <a href="https://github.com/cncf/toc/issues/1746">TOC Issue #1746</a> 中讨论 Agent 内存需求。

**关键主题：**

- **状态与内存后端**：哪些后端（对象存储、向量 DB、Redis）和 API 形态适合短期和长期 Agent 内存
- **一致性**：如何确保跨数千个 Agent 的一致性
- **垃圾回收**：清理非活跃 Agent 状态的策略
- **差距识别**：当前生态系统中缺失的部分，例如：
  - 标准化的 Agent 内存 API
  - MCP-Kubernetes 发现机制
  - 状态管理的集成模式

**提议的解决方案：**

- Dapr 状态组件用于后端抽象
- 向量数据库 Operator（Chroma、Milvus）用于语义搜索
- S3/GCS 用于长期存储
- Redis 用于热状态和缓存

---

## 参考资料

### 项目

- [memU (NevaMind-AI)](https://github.com/NevaMind-AI/memU)
- [DB-GPT (eosphoros-ai)](https://github.com/eosphoros-ai/DB-GPT)
- [NIXL (ai-dynamo)](https://github.com/ai-dynamo/nixl)
- [LMCache](https://github.com/LMCache/lmcache)
- [Chroma](https://github.com/chroma-core/chroma)
- [Milvus](https://github.com/milvus-io/milvus)
- [Dapr](https://github.com/dapr/dapr)

### 博客文章

- [KV-Cache Wins You Can See (llm-d)](https://llm-d.ai/blog/kvcache-wins-you-can-see)
- [LMCache on GKE](https://blog.lmcache.ai/2025-10-07-LMCache-on-GKE/)
- [Dynamo LMCache Integration](https://blog.lmcache.ai/2025-09-18-dynamo-lmcache/)

### 社区讨论

- [CNCF TOC Issue #1746: Agent Memory and State Management](https://github.com/cncf/toc/issues/1746)

### 相关文档

- [LLM 推理中的缓存](./caching.md) - KV 缓存和前缀缓存
- [Prefill-Decode 分离](./pd-disaggregation.md) - 分布式推理架构
- [模型生命周期管理](./model-lifecycle.md) - 冷启动、休眠模式和卸载

---

**注意**：本文档汇总了来自多个来源的信息，包括 GitHub 仓库、博客文章和社区讨论。某些技术和 API 正在活跃开发中。在生产实施前请验证当前项目状态和能力。
