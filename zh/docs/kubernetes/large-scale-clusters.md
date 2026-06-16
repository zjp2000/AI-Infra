---
status: Active
maintainer: pacoxu
last_updated: 2025-11-24
tags: kubernetes, scalability, large-scale, api-server, etcd
canonical_path: zh/docs/kubernetes/large-scale-clusters.md
---

# 大规模 Kubernetes 集群

本文档涵盖支持大规模 Kubernetes 集群的关键技术和架构模式。它讨论了来自各种实现（包括 Google 的 130,000 节点 GKE 集群和其他大规模部署）的经过验证的方法。

## 概述

大规模 Kubernetes 集群（10,000+ 节点）在 API 服务器性能、存储后端可扩展性、网络资源管理和分布式存储等方面带来了独特的挑战。本指南探讨了应对这些挑战的技术和模式。

**关键资源：**

- [谷歌如何做到：构建已知最大的 Kubernetes 集群，拥有 130,000 个节点](https://cloud.google.com/blog/products/containers-kubernetes/how-we-built-a-130000-node-gke-cluster/)
  （[中文翻译](https://mp.weixin.qq.com/s/Cp9X574wpMnJsPAasPpkVw)）
- [超大集群还是多集群：识别瓶颈](https://github.com/user-attachments/files/23701506/A.Huge.Cluster.or.Multi-Clusters_.Identifying.the.Bottleneck.rc.2.pptx)
  - 讨论大规模部署的瓶颈和架构决策的演示文稿

## 关键技术

### 1. 从缓存进行一致性读取（KEP-2340）

**状态**：已在 Kubernetes 中实现

**问题**：在大规模场景下，etcd 成为瓶颈，因为 Kubernetes API 服务器需要处理数百万的读取请求。传统架构要求所有读取都经过 etcd，带来了性能和可扩展性限制。

**解决方案**：KEP-2340 在 API 服务器中引入了一个缓存层，允许一致性读取直接从缓存服务，无需访问 etcd。这在大幅降低 etcd 负载的同时保持了一致性保证。

**关键特性**：

- **基于 Watch 的缓存**：API 服务器维护对所有资源的 watch，并保持缓存最新
- **一致性保证**：确保从缓存读取的内容与 etcd 中的实际状态一致
- **资源版本化**：使用资源版本检测过期的读取
- **降低 etcd 负载**：显著减少对 etcd 集群的读取压力

**对大集群的益处**：

- 支持处理数百万 API 请求而不压垮 etcd
- 改善集群整体响应能力
- 降低 etcd 集群规模需求
- 允许 API 服务器的水平扩展

**参考**：

- [KEP-2340：从缓存进行一致性读取](https://github.com/kubernetes/enhancements/tree/master/keps/sig-api-machinery/2340-Consistent-reads-from-cache)
- [Kubernetes API 服务器缓存](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation/)

### 2. 可快照的 API 服务器缓存（KEP-4988）

**状态**：开发中/Alpha

**问题**：当 API 服务器重启或扩容时，它们需要通过从 etcd 列出所有资源来重建整个缓存。在 130,000 节点规模下，这种初始缓存填充可能花费大量时间，并对 etcd 造成巨大的负载峰值。

**解决方案**：KEP-4988 引入了对 API 服务器缓存进行快照并在启动时恢复的能力，大大减少了上线新 API 服务器所需的时间和负载。

**关键特性**：

- **快速缓存恢复**：新 API 服务器可以在数秒（而非数分钟）内开始服务请求
- **降低 etcd 负载**：消除了 API 服务器启动期间大规模 LIST 操作的需求
- **持久化存储**：缓存快照可以存储在对象存储中（GCS、S3 等）
- **增量更新**：从快照恢复后，API 服务器只需捕获快照之后的变更

**对大集群的益处**：

- 更快的 API 服务器扩展和恢复
- 降低 API 服务器重启对集群稳定性的影响
- 降低扩展事件期间的 etcd 资源需求
- 改善集群的韧性和可用性

**实现细节**：

- 快照格式包含资源版本以确保一致性
- 支持增量快照更新
- 与现有的基于 watch 的缓存机制兼容
- 可与一致性读取缓存配合使用以获得最大效率

**参考**：

- [KEP-4988：可快照的 API 服务器缓存](https://github.com/kubernetes/enhancements/issues/4988)

### 3. DRANET（网络动态资源分配）

**状态**：属于更广泛的 DRA 计划的一部分

**问题**：传统的 Kubernetes 网络假设相对简单的网络需求。在大规模 AI/ML 工作负载场景下，集群需要复杂的网络资源管理，包括 RDMA、SR-IOV 和专用网络设备。

**解决方案**：DRANET 将动态资源分配（DRA）框架扩展到网络资源，支持对网络设备和能力的细粒度分配和管理。

**关键特性**：

- **网络设备管理**：将网络设备视为 Kubernetes 中的一等资源
- **RDMA 支持**：支持为高性能 AI/ML 工作负载高效分配 RDMA（远程直接内存访问）
- **SR-IOV 集成**：管理 SR-IOV 虚拟功能以实现网络隔离
- **拓扑感知**：在调度 Pod 时考虑网络拓扑
- **多网络支持**：允许 Pod 请求具有不同特性的多个网络接口

**对大集群的益处**：

- 对需要高带宽、低延迟网络的 AI/ML 训练工作负载至关重要
- 支持高效使用专用网络硬件
- 支持复杂的多租户网络需求
- 改善网络密集型工作负载的性能可预测性

**在大集群中的用例**：

- 分布式训练的 GPU 间通信
- 高吞吐量数据摄取管道
- 低延迟推理服务
- 多租户网络隔离

**参考**：

- [Kubernetes 动态资源分配](https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/)
- [KEP-3063：动态资源分配](https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/3063-dynamic-resource-allocation)
- 另请参阅：[DRA 文档](./dra.md)

### 4. 可扩展存储后端（Spanner 及替代方案）

**问题**：etcd 虽然在大多数 Kubernetes 部署中表现出色，但在集群规模上存在实际限制。在非常大的规模（50,000+ 节点）下，资源和 watch 流的数量可能超出传统 etcd 集群的高效处理能力。

**解决方案**：存在几种超越 etcd 限制的方法：

#### Google Cloud Spanner

**状态**：生产就绪，Google Cloud 特有

Google Cloud 使用 Cloud Spanner 作为 Kubernetes API 服务器的替代后端，替换 etcd 进行数据存储，同时保持 API 兼容性。

**关键特性**：

- **大规模可扩展**：Spanner 设计为可扩展到数百万节点和 PB 级数据
- **全局分布**：支持多区域部署，具有强一致性
- **自动分片**：数据自动分布到多个服务器
- **无需手动扩展**：根据负载自动扩展
- **强一致性**：提供像 etcd 一样的线性化读写

**对大集群的益处**：

- 消除了 etcd 作为可扩展性瓶颈的问题
- 支持远超典型 Kubernetes 规模限制的集群
- 提供更好的可用性和灾难恢复
- 降低了管理 etcd 集群的运维开销

**与 etcd 的对比**：

| 特性 | etcd | Cloud Spanner |
| --- | --- | --- |
| 最大集群规模 | ~5,000 节点（实际） | 100,000+ 节点 |
| 扩展方式 | 手动，有限 | 自动，无限 |
| 多区域 | 需要联邦 | 原生支持 |
| 运维开销 | 大规模下高 | 低（托管服务） |
| 成本 | 仅基础设施 | 按使用付费 |

**技术细节**：

- 使用与 etcd 相同的 API 服务器接口
- 实现了与 Kubernetes 期望兼容的 watch 机制
- 针对 Kubernetes 访问模式优化（多 watch，少写入）
- 与一致性读取缓存集成以获得最佳性能

#### 替代方案

虽然 Spanner 是 Google Cloud 特有的，但其他组织使用不同的方法来超越 etcd 的限制：

- **TiKV**：云原生分布式键值存储，带有 Kubernetes 兼容层
- **多集群联邦**：将工作负载拆分到多个较小的集群，而非单个巨型集群
- **优化的 etcd 配置**：仔细调优 etcd 参数、使用专用高性能硬件和策略性分片

**注意**：选择单个大集群还是多个小集群取决于工作负载特性、运维复杂性和成本考虑。请参阅演示文稿"[超大集群还是多集群：识别瓶颈](https://github.com/user-attachments/files/23701506/A.Huge.Cluster.or.Multi-Clusters_.Identifying.the.Bottleneck.rc.2.pptx)"以了解此权衡的更多细节。

**参考**：

- [Cloud Spanner](https://cloud.google.com/spanner)
- [Kubernetes API 服务器存储接口](https://kubernetes.io/docs/concepts/overview/kubernetes-api/)

### 5. 高性能分布式存储（Lustre 及替代方案）

**状态**：多种开源和商业解决方案可用

**问题**：在大规模 AI/ML 工作负载场景下，集群需要处理海量数据用于模型训练、推理和检查点保存。传统存储解决方案无法提供所需的吞吐量和 IOPS。

**解决方案**：在大规模 Kubernetes 部署中使用了多种高性能分布式文件系统：

#### Lustre

Lustre 提供了一个针对大规模并行 I/O 工作负载优化的高性能分布式文件系统，使其成为在 Kubernetes 上运行 AI/ML 集群的理想选择。

**关键特性**：

- **并行 I/O**：多个客户端可以同时读写同一个文件
- **高吞吐量**：可实现数百 GB/s 的聚合吞吐量
- **大规模**：支持数千节点和 PB 级数据的集群
- **POSIX 兼容**：与标准文件系统 API 兼容
- **灵活架构**：分离的元数据和对象存储服务器

**架构组件**：

- **元数据服务器（MDS）**：管理文件命名空间和权限
- **对象存储服务器（OSS）**：存储实际文件数据
- **客户端**：挂载 Lustre 文件系统并访问数据
- **管理服务器（MGS）**：协调集群配置

**对大型 AI 集群的益处**：

- **训练数据访问**：支持跨数千 GPU 高效加载训练数据集
- **检查点存储**：分布式训练期间的快速检查点写入
- **模型存储**：高性能模型制品存储和版本管理
- **共享存储**：多个 Pod 可以同时访问相同的数据

**Kubernetes 集成**：

- **CSI 驱动**：第三方 Lustre CSI 驱动可用于动态卷供应
- **StatefulSets**：支持训练作业的稳定、持久存储
- **Persistent Volumes**：Lustre 卷的标准 PV/PVC 接口
- **ReadWriteMany**：支持共享数据集的 RWX 访问模式

**在 130K 节点集群中的用例**：

1. **分布式训练**：跨多个节点共享访问训练数据集
2. **检查点保存**：用于容错的高速检查点写入
3. **模型注册中心**：训练模型的集中式存储
4. **数据预处理**：并行数据转换管道
5. **推理缓存**：模型制品的共享缓存

**性能特性**：

- **顺序读取**：100+ GB/s 聚合吞吐量
- **随机 I/O**：集群范围内数百万 IOPS
- **元数据操作**：每秒数万次操作
- **可扩展性**：随服务器数量线性扩展

**与其他存储解决方案的对比**：

| 特性 | Lustre | NFS | Ceph | 本地 SSD |
| --- | --- | --- | --- | --- |
| 吞吐量 | 非常高 | 低 | 中等 | 非常高 |
| IOPS | 非常高 | 低 | 中等 | 非常高 |
| 可扩展性 | 优秀 | 差 | 良好 | 不适用 |
| 共享访问 | 是 | 是 | 是 | 否 |
| 复杂度 | 高 | 低 | 中等 | 低 |
| 用例 | HPC/AI 训练 | 通用 | 通用 | 节点本地 |

**部署考虑**：

- **网络要求**：需要高带宽、低延迟网络（InfiniBand 或 100GbE+）
- **服务器规格**：MDS 和 OSS 节点需要充足的资源
- **客户端配置**：针对 Kubernetes 工作负载的适当调优
- **监控**：全面的 Lustre 组件监控
- **备份策略**：定期快照和灾难恢复规划

**替代解决方案**：

对于不使用 Lustre 的组织，可以通过以下方案实现类似能力：

- **WekaFS**：具有类似性能的现代并行文件系统
- **BeeGFS**：HPC 中流行的替代并行文件系统
- **IBM Spectrum Scale (GPFS)**：企业级并行文件系统
- **云原生**：带有客户端缓存的对象存储（S3、GCS）

**参考**：

- [Lustre 文件系统](https://www.lustre.org/)
- [Lustre 文档](https://doc.lustre.org/)
- [Kubernetes 中的高性能存储](https://kubernetes.io/blog/2019/01/15/container-storage-interface-ga/)

## 大规模集群的架构模式

### 多 API 服务器架构

运行多个 API 服务器实例并配以负载均衡：

- **负载均衡器**：在各 API 服务器之间分发请求
- **健康检查**：从不健康的 API 服务器轮换中移除
- **缓存一致性**：确保各 API 服务器缓存间的一致性
- **优雅扩展**：无中断地添加/移除 API 服务器

### 资源分区

将集群划分为逻辑分区：

- **命名空间隔离**：分离工作负载类型
- **节点池**：特定工作负载类型的专用节点
- **优先级类**：确保关键工作负载获得资源
- **资源配额**：防止任何租户消耗所有资源

## 大规模集群的最佳实践

### 1. 监控和可观测性

- **指标聚合**：使用高效的指标收集（例如 Prometheus 联邦）
- **分布式追踪**：跨组件追踪请求
- **日志聚合**：集中式日志记录并设置适当的保留期
- **告警**：对性能降级进行主动告警

### 2. 升级策略

- **滚动升级**：增量升级 API 服务器和节点
- **金丝雀测试**：先在小部分节点上测试升级
- **自动验证**：每一步后验证集群健康
- **回滚能力**：为故障提供快速回滚机制

### 3. 大规模安全

- **RBAC 优化**：高效的基于角色的访问控制
- **网络策略**：分段工作负载间的流量
- **准入控制器**：执行安全策略
- **审计日志**：追踪所有 API 服务器访问

### 4. 性能优化

- **API 服务器调优**：优化缓存大小、速率限制
- **etcd/Spanner 优化**：正确的存储后端配置
- **网络优化**：为控制平面使用快速网络
- **客户端缓存**：使用 informer 减少 API 服务器负载

## 未来方向

### 持续改进

- **增强的缓存**：对 API 服务器缓存的进一步优化
- **更好的分片**：更高效地分发负载
- **存储后端**：etcd 和 Spanner 之外的更多选择
- **可观测性**：更好的集群内部可见性

### 社区倡议

- [SIG Scalability](https://github.com/kubernetes/community/tree/master/sig-scalability)：致力于 Kubernetes 可扩展性改进
- [Scalability SLIs/SLOs](https://github.com/kubernetes/community/blob/master/sig-scalability/slos/slos.md)：为大型集群定义性能目标

## 参考

### 博客文章

- [谷歌如何做到：构建已知最大的 Kubernetes 集群，拥有 130,000 个节点](https://cloud.google.com/blog/products/containers-kubernetes/how-we-built-a-130000-node-gke-cluster/)
- [中文翻译（微信公众号）](https://mp.weixin.qq.com/s/Cp9X574wpMnJsPAasPpkVw)

### Kubernetes 增强提案（KEP）

- [KEP-2340：从缓存进行一致性读取](https://github.com/kubernetes/enhancements/tree/master/keps/sig-api-machinery/2340-Consistent-reads-from-cache)
- [KEP-4988：可快照的 API 服务器缓存](https://github.com/kubernetes/enhancements/issues/4988)
- [KEP-3063：动态资源分配](https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/3063-dynamic-resource-allocation)

### 官方文档

- [Kubernetes 可扩展性](https://kubernetes.io/docs/setup/best-practices/cluster-large/)
- [API 服务器配置](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/)
- [Cloud Spanner 文档](https://cloud.google.com/spanner/docs)
- [Lustre 文档](https://doc.lustre.org/)

### 相关主题

- [调度优化](./scheduling-optimization.md)
- [动态资源分配](./dra.md)
- [Pod 启动速度](./pod-startup-speed.md)

---

**注意**：本文档基于 Google 博客文章和 Kubernetes 社区资源的公开信息。具体实现细节可能有所不同。生产部署请始终参考官方文档。
