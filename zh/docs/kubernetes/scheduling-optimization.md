---
status: Active
maintainer: pacoxu
last_updated: 2026-03-24
tags: kubernetes, scheduling, optimization, large-scale
canonical_path: zh/docs/kubernetes/scheduling-optimization.md
---

# 大规模集群调度优化

本指南提供了在大规模 Kubernetes 集群中优化调度性能和资源利用率的综合策略，特别针对 AI/ML 工作负载。大规模集群（1000+ 节点，10000+ Pod）需要专门的调度技术来维持性能和最大化资源利用率。

## 为什么调度优化很重要

高效调度对于以下方面至关重要：

- **高吞吐量**：支持快速 Pod 创建（1000+ Pod/s）
- **资源效率**：最大化 GPU/CPU 利用率（>80%）
- **成本优化**：减少闲置资源和过度配置
- **SLA 合规**：满足延迟和可用性要求
- **公平性**：确保跨租户的公平资源分配

## 概述

调度优化聚焦于两个关键维度：

1. **性能**：吞吐量、延迟和 API 响应能力
2. **资源利用率**：高效分配、负载均衡和 binpack

---

## 1. 性能优化

### 1.1 高吞吐量调度

**挑战**：默认 Kubernetes 调度器每秒处理约 100 个 Pod，不足以应对大规模批处理工作负载。

**解决方案：**

#### 乐观并发

- **概念**：假设资源可用，并发调度多个 Pod，通过重试处理冲突
- **权衡**：吞吐量更高，但资源有限时可能产生冲突
- **最适合**：资源丰富、Pod 需求多样的集群

**示例项目：**

- <a href="https://github.com/kubewharf/godel-scheduler">`Godel Scheduler`</a>：字节跳动调度器，通过乐观并发和多级调度队列实现 5000 Pod/s

#### 多调度器架构

- **概念**：并行运行多个调度器实例，每个处理部分工作负载
- **策略**：
  - **基于工作负载**：批处理、服务、训练分别使用独立调度器
  - **基于租户**：每个团队/命名空间使用专用调度器
  - **基于优先级**：高/低优先级工作负载使用不同调度器
- **协调**：使用领导者选举或分区避免冲突

**示例项目：**

- <a href="https://github.com/kubernetes-sigs/kueue">`Kueue`</a>：Kubernetes SIG 项目，支持多队列和配额管理
- <a href="https://github.com/volcano-sh/volcano">`Volcano`</a>：CNCF 孵化项目，具有 Gang 调度和队列管理

**最佳实践：**

- 监控冲突率；如果超过 10%，减少并发度或调度器数量
- 使用节点亲和性/反亲和性跨调度器划分工作负载
- 配置适当的缓存同步间隔（默认：30s）

### 1.2 多节点池策略

**概念**：将集群节点划分为池，为特定池分配调度器，以减少调度冲突并提高缓存局部性。

**节点池类型：**

- **专用池**：GPU 节点、推理节点、训练节点
- **共享池**：具有资源溢出的一般工作负载
- **优先级池**：具有预留容量的高优先级工作负载

**实现：**

- 使用节点标签/污点定义池：`node-pool=gpu-training`、`node-pool=cpu-batch`
- 通过 `nodeSelector` 或 `nodeAffinity` 配置调度器亲和性
- 为利用不足的资源启用跨池调度

**示例项目：**

- <a href="https://github.com/koordinator-sh/koordinator">`Koordinator`</a>：CNCF Sandbox 项目，具有混合部署和 QoS 管理
- <a href="https://github.com/ai-dynamo/grove">`NVIDIA Grove`</a>：具有拓扑感知的 AI 工作负载 Gang 调度

### 1.3 分层调度

**概念**：将调度分为多个层级，以更高效地处理超大规模集群（50,000+ 节点）。每个层级做出更粗粒度的决策，为后续层级缩小搜索空间。

**调度层级：**

- **集群级**：根据工作负载特征和资源可用性，决定考虑哪些节点子集或节点池
- **池级**：根据拓扑、资源适配性和亲和性规则，在选定的池中选择具体节点
- **节点级**：最终放置决策，考虑 GPU 拓扑、NUMA 对齐和其他细粒度约束

**对大规模集群的好处：**

- 通过限制每个层级的搜索空间减少调度延迟
- 支持不同粒度下的专门调度逻辑
- 改善缓存局部性，减少 API server 负载
- 支持每种工作负载类型的不同调度策略

**实现模式：**

- **预过滤**：在调度开始前使用节点池和标签划分节点
- **多遍调度**：第一遍选择区域/可用区，第二遍选择具体节点
- **分阶段决策**：在细粒度约束（拓扑、亲和性）之前应用粗粒度过滤器（CPU/内存/GPU 数量）

**示例用例：**

对于包含 GPU 训练工作负载的 100,000 节点集群：

1. **集群级**：根据 GPU 类型要求选择 GPU 节点池（A100 vs H100）
2. **池级**：选择具有足够容量和低网络延迟的可用区
3. **节点级**：考虑 NVLink 拓扑进行最优多 GPU 训练的 Pod 放置

**参考：**

- 参见[大规模集群](./large-scale-clusters.md)了解大规模部署中使用的其他模式

### 1.4 API 性能优化

**常见瓶颈：**

- **etcd 延迟**：慢速读写影响 Pod 创建
- **API Server 负载**：来自调度器、控制器的高请求率
- **准入 webhook**：同步验证增加延迟
- **大对象**：具有大量卷、环境变量的 Pod 拖慢 API 操作

**解决方案：**

- **etcd 优化**：
  - 为 etcd 存储使用 SSD（P99 延迟 <10ms）
  - 大型部署使用独立的 etcd 集群（5+ 节点）
  - 启用 etcd 碎片整理和压缩
- **API Server 扩展**：
  - 运行多个 API Server 副本（每 1000 节点 1 个）
  - 增加请求限制：`--max-requests-inflight=3000`、`--max-mutating-requests-inflight=1000`
  - 使用优先级和公平性（APF）防止调度器饥饿
- **准入 webhook 优化**：
  - 尽可能将同步 webhook 转为异步
  - 设置激进的超时（默认：10s，减少到 2-3s）
  - 对非关键验证使用 webhook 失败策略：`Ignore`
  - 缓存验证结果以避免重复调用
- **对象大小优化**：
  - 使用 ConfigMaps/Secrets 代替内联 Pod 规格
  - 限制每个 Pod 的卷数量（<20）
  - 使用策略合并补丁进行更新

**监控：**

- 跟踪 API Server 请求延迟（P95/P99）
- 监控 etcd 后端提交持续时间
- 对准入 webhook 超时发出告警

### 1.5 批量分发优化

**概念**：批量而非单独调度 Pod，以分摊调度开销并提高吞吐量。

**批量调度策略：**

- **固定批量大小**：一次调度 N 个 Pod（例如每次 100 个 Pod）
- **基于时间的批处理**：收集 T 秒的 Pod 然后一起调度
- **基于优先级的批处理**：按优先级类分组 Pod

**实现：**

- **Kueue**：提供工作负载批处理和准入控制
- **Volcano**：支持具有作业级队列管理的批量调度
- **自定义控制器**：使用 Operator 模式实现批量准入

**最佳实践：**

- 批量大小应在吞吐量和延迟之间取得平衡（50-200 个 Pod）
- 对交互式和批处理工作负载使用独立的队列
- 监控队列深度并动态调整批处理参数

---

## 2. 资源利用率优化

### 2.1 Dominant Resource Fairness（DRF）

**概念**：考虑多种资源类型（CPU、内存、GPU、存储、带宽）的公平资源分配，而非单一维度。

**DRF 工作原理：**

- 计算每个用户的 dominant 资源（最高利用率百分比）
- 分配资源以最大化最小 dominant 资源份额
- 确保没有用户在其 dominant 资源上成为瓶颈

**示例**：用户 A 需要 2 CPU + 4 GB，用户 B 需要 1 CPU + 8 GB。在 8 CPU、32 GB 的集群中：

- 用户 A 的 dominant 资源：内存（4/32 = 12.5%）
- 用户 B 的 dominant 资源：内存（8/32 = 25%）
- DRF 分配以均衡 dominant 份额

**实现：**

- <a href="https://github.com/apache/yunikorn-core">`Apache YuniKorn`</a>：为多租户调度实现 DRF
- <a href="https://github.com/volcano-sh/volcano">`Volcano`</a>：通过 proportion 插件支持 DRF

**最佳实践：**

- 为 AI 工作负载将 GPU 内存作为单独的资源维度
- 考虑分布式训练的网络带宽（RDMA/IB）
- 使用权重优先考虑某些资源（例如 GPU 权重 = 10x CPU）

### 2.2 负载感知调度

**概念**：基于实时节点负载指标（CPU 使用率、内存压力、GPU 利用率）而非仅容量做出调度决策。

**需要考虑的指标：**

- **CPU**：系统负载均值、CPU 使用率
- **内存**：可用内存、内存压力（页缓存、swap）
- **GPU**：GPU 利用率、GPU 内存使用、温度
- **网络**：带宽使用、丢包率
- **磁盘**：I/O 等待、磁盘使用率

**实现：**

- <a href="https://github.com/koordinator-sh/koordinator">`Koordinator`</a>：具有节点指标的实时负载感知调度
- <a href="https://github.com/kubewharf/godel-scheduler">`Godel Scheduler`</a>：具有 katalyst 运行时 QoS 的负载感知评分
- **Trimaran**：用于负载感知调度的 Kubernetes 调度器插件

**最佳实践：**

- 使用短指标收集间隔（10-30s）确保响应性
- 结合请求/限制与负载指标进行混合评分
- 设置负载阈值，防止调度到过载节点（>80% CPU）
- 考虑多维负载（避免调度到 CPU 和内存压力都高的节点）

### 2.3 Binpack 策略

**概念**：将 Pod 打包到更少的节点上，以最大化利用率并支持节点合并/自动伸缩。

**Binpack 算法：**

- **Best Fit**：将 Pod 放置在分配后剩余容量最少的节点上
- **First Fit**：将 Pod 放置在第一个有足够容量的节点上
- **Worst Fit**：将 Pod 放置在剩余容量最多的节点上（用于分散）
- **多维 binpack**：同时考虑 CPU、内存、GPU

**实现：**

- **Kubernetes Scheduler**：配置 `NodeResourcesBalancedAllocation` 和 `NodeResourcesMostAllocated` 插件
- <a href="https://github.com/kubernetes-sigs/kueue">`Kueue`</a>：支持配额内的工作负载打包
- <a href="https://github.com/koordinator-sh/koordinator">`Koordinator`</a>：基于 QoS 的混合部署和 binpack

**配置示例：**

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
- pluginConfig:
  - name: NodeResourcesMostAllocated
    args:
      resources:
      - name: cpu
        weight: 1
      - name: memory
        weight: 1
      - name: nvidia.com/gpu
        weight: 10
```

**权衡：**

- **优点**：更高的利用率、更低的成本、更容易的自动伸缩
- **缺点**：减少爆炸半径、潜在的资源争用
- **最适合**：批处理工作负载、非关键服务、开发/测试环境

### 2.4 抢占和优先级

**概念**：当资源稀缺时，可以驱逐低优先级 Pod 以为高优先级 Pod 腾出空间。

**优先级类：**

- **系统关键**：kube-system 组件（优先级：2000000000）
- **业务关键**：生产服务（优先级：1000-10000）
- **标准**：常规工作负载（优先级：0）
- **尽力而为**：批处理作业、开发工作负载（优先级：-1000）

**抢占策略：**

- **PreemptLowerPriority**：默认，抢占低优先级 Pod
- **Never**：禁用特定优先级类的抢占
- **优雅抢占**：允许 Pod 终止的宽限期

**实现：**

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority-ai-training
value: 10000
globalDefault: false
description: "High priority for critical AI training jobs"
preemptionPolicy: PreemptLowerPriority
```

**最佳实践：**

- 限制优先级类数量（5-7 个）以避免复杂性
- 谨慎使用优先级以防止级联抢占
- 设置 Pod 中断预算（PDB）以限制抢占影响
- 监控抢占指标并相应调整优先级
- 考虑抢占成本（浪费的工作）与 SLA 要求

**示例项目：**

- <a href="https://github.com/volcano-sh/volcano">`Volcano`</a>：具有队列级优先级的高级抢占
- <a href="https://github.com/kubernetes-sigs/kueue">`Kueue`</a>：cohort 内的抢占和借用限制

### 2.5 Gang 调度

**概念**：分布式工作负载的全或无调度。要么组中所有 Pod 都被调度，要么都不调度，防止部分调度和死锁。

**使用场景：**

- **分布式训练**：PyTorch/TensorFlow 多节点训练
- **MPI 作业**：紧密耦合的 HPC 工作负载
- **Spark/Ray**：多进程计算框架

**Gang 调度阶段：**

1. **准入**：检查整个 gang 的资源是否可用
2. **预留**：为所有 gang 成员预留资源
3. **绑定**：同时调度所有 Pod
4. **超时**：如果 gang 无法在超时内被调度，释放资源

**实现：**

- <a href="https://github.com/volcano-sh/volcano">`Volcano`</a>：带有 minMember 字段的 PodGroup CRD 用于 Gang 调度
- <a href="https://github.com/ai-dynamo/grove">`NVIDIA Grove`</a>：针对 NVIDIA GPU 优化的 Gang 调度
- <a href="https://github.com/kubernetes-sigs/kueue">`Kueue`</a>：具有 Gang 调度的工作负载准入控制
- <a href="https://github.com/kubernetes-sigs/lws">`LeaderWorkerSet (LWS)`</a>：Kubernetes SIG 项目，支持 Gang 调度

**示例（Volcano PodGroup）：**

```yaml
apiVersion: scheduling.volcano.sh/v1beta1
kind: PodGroup
metadata:
  name: distributed-training-job
spec:
  minMember: 8  # 所有 8 个 Pod 必须被调度
  minResources:
    cpu: "64"
    memory: "256Gi"
    nvidia.com/gpu: "8"
  queue: ai-training-queue
```

**最佳实践：**

- 设置合理的 gang 超时时间（5-15 分钟）以防止无限等待
- 尽可能使用弹性 Gang 调度（minMember < maxMember）
- 结合 coscheduling 插件实现拓扑感知放置
- 监控 Gang 调度失败并调整资源配额

### 2.6 拓扑感知调度

**概念**：考虑物理拓扑（NUMA 节点、PCIe 通道、网络拓扑）的 Pod 调度，以优化局部性和带宽。

**拓扑维度：**

- **NUMA 节点**：CPU 和内存局部性
- **GPU 拓扑**：NVLink 连接、PCIe 通道
- **网络拓扑**：机架/可用区感知、RDMA 结构
- **存储拓扑**：本地 vs 远程存储
- **机架拓扑**：故障域的物理机架位置

**拓扑为何对 AI 工作负载重要：**

现代 AI 工作负载对拓扑极为敏感。硬件拓扑错位可能导致性能下降 30-50% 或更多：

- **GPU 间通信**：NVLink 提供 600 GB/s vs PCIe 的 64 GB/s
- **RDMA/GPU-Direct**：需要 GPU 和 NIC 在同一 PCIe root 下
- **NUMA 局部性**：跨 NUMA 内存访问增加 30-50% 延迟

**实现方法：**

1. **Device Plugin + Topology Manager**：传统方法，跨设备协调有限
2. **Kueue Topology-Aware Scheduling**：使用节点标签进行机架/块/主机拓扑
3. **Volcano Topology Plugin**：Gang 调度中的 GPU 和网络拓扑
4. **DRA（动态资源分配）**：使用 CEL 的丰富拓扑约束

**项目支持：**

- <a href="https://github.com/kubernetes/dynamic-resource-allocation/">`DRA（动态资源分配）`</a>：设备拓扑的结构化参数（v1.34 中 GA）
- <a href="https://github.com/google/dranet">`DRANET`</a>：Google 的网络设备拓扑 DRA 实现
- <a href="https://github.com/containerd/nri">`NRI（节点资源接口）`</a>：运行时的细粒度设备管理
- <a href="https://github.com/kubernetes-sigs/kueue">`Kueue`</a>：使用节点标签的拓扑感知调度
- <a href="https://github.com/volcano-sh/volcano">`Volcano`</a>：具有拓扑插件的 Gang 调度
- <a href="https://github.com/NVIDIA/KAI-Scheduler">`NVIDIA KAI Scheduler`</a>：GPU 优化的拓扑感知调度
- **Kubernetes Topology Manager**：NUMA 感知 Pod 放置

**DRA 拓扑协调（GPU + NIC）：**

DRA 能够在同一拓扑域上协调多种设备类型：

```yaml
apiVersion: resource.k8s.io/v1beta1
kind: ResourceClaimTemplate
metadata:
  name: gpu-nic-topology
spec:
  spec:
    devices:
      requests:
      - name: gpu
        deviceClassName: nvidia-gpu
        count: 1
      - name: rdma-nic
        deviceClassName: rdma-nic
        count: 1
      constraints:
      # GPU 和 NIC 必须在同一 PCIe root 下
      - requests: ["gpu", "rdma-nic"]
        matchAttribute: pcieRoot
```

**Kueue 拓扑标签：**

```yaml
# Kueue 和 DRANET 使用的标准拓扑标签
cloud.google.com/gce-topology-block: "block-1"
cloud.google.com/gce-topology-subblock: "subblock-1"
cloud.google.com/gce-topology-host: "host-1"
kubernetes.io/hostname: "node-1"
```

**拓扑感知场景：**

- **多 GPU 训练**：将 Pod 调度到具有 NVLink/NVSwitch 的节点，实现快速 GPU 间通信
- **RDMA 工作负载**：将 Pod 放置在具有 InfiniBand 连接和 GPU-Direct 支持的节点上
- **内存密集型**：调度到具有本地内存访问的 NUMA 节点
- **分布式训练**：协调同一 PCIe 结构上的 GPU + NIC

**最佳实践：**

- 对复杂设备需求（GPU + NIC + 存储）使用 DRA
- 配置 Topology Manager 策略：对延迟敏感的工作负载使用 `single-numa-node`
- 使用拓扑信息标记节点：`topology.kubernetes.io/zone`、`gpu-interconnect=nvlink`
- 使用 Kueue 或 Volcano 实现机架级拓扑感知
- 测试拓扑影响：比较对齐和未对齐时的基准性能
- 考虑权衡：拓扑优化可能减少调度灵活性

**相关博客文章：**
[拓扑感知调度博客](../archive-blog/2025-11-25/2025-11-25-topology-aware-scheduling.md) 详细了解 DRA 拓扑管理。

### 2.7 AI 工作负载存储

AI 工作负载需要专门的存储策略来处理大型模型权重、训练数据集和检查点。存储是一等调度关注点，因为模型加载时间直接影响 Pod 启动延迟和 GPU 空闲时间。

#### 模型分发

大型语言模型（数十到数百 GB）必须高效分发到推理 Pod。选项：

| 方法 | 描述 | 最适合 |
| --- | --- | --- |
| OCI Image Volume | 将模型打包为 OCI 工件（KEP-4639，v1.33+ Beta） | 不可变模型权重，基于 registry 的分发 |
| ReadOnlyMany PVC | 跨多个 Pod 的共享卷 | 共置推理副本 |
| 对象存储 | 启动时从 S3/GCS/OSS 流式加载权重 | 大型模型，不频繁更新 |
| 节点本地缓存 | 通过 DaemonSet 预拉到 NVMe | 低延迟冷启动 |

**用于模型权重的 OCI Image Volume**（推荐用于 Kubernetes v1.33+）：

将模型文件打包为存储在标准容器 registry 中的 OCI 工件。Kubernetes 将镜像内容挂载为只读卷，实现 registry 级去重和层缓存。

```yaml
# 通过 OCI Image Volume 提供 Llama 模型权重
spec:
  volumes:
  - name: model-weights
    image:
      reference: registry.example.com/models/llama3:70b
      pullPolicy: IfNotPresent   # 首次拉取后在节点上缓存
  containers:
  - name: vllm
    image: vllm/vllm-openai:latest
    args: ["--model", "/models"]
    volumeMounts:
    - name: model-weights
      mountPath: /models
    resources:
      limits:
        nvidia.com/gpu: "1"
```

**为什么使用 OCI Image Volume？**

- **Registry 去重**：层跨模型版本共享
- **原子更新**：模型版本与镜像标签关联
- **标准工具**：与现有容器 registry 基础设施配合使用
- **预拉取支持**：基于 DaemonSet 的预热节点预拉取

参见 [OCI Taking Over Everything](../archive-blog/2025-12-22/2025-12-22-oci-taking-over-everything_en.md) 了解详细背景。

#### Checkpointing 存储

分布式训练需要高带宽检查点写入。存储建议：

- **并行文件系统**（Lustre、GPFS、WekaFS）：跨多个 GPU 节点的高吞吐量共享存储，用于大型检查点
- **ReadWriteMany PVC**：NFS 或并行 FS 支持的卷，用于分布式检查点协调
- **对象存储后端**：异步检查点上传到 S3/GCS/OSS，减少 GPU 阻塞时间（训练继续时异步写入检查点）
- **GPUDirect Storage（GDS）**：绕过 CPU 的直接 GPU 到存储传输，由 NVIDIA GPU Operator 支持

**AI 存储的 CSI 驱动：**

- <a href="https://github.com/lustre/lustre-release">`Lustre CSI`</a>：用于 HPC/AI 检查点的高性能并行文件系统
- <a href="https://github.com/kubernetes-csi/csi-driver-nfs">`NFS CSI Driver`</a>：用于中等检查点负载的简单 ReadWriteMany
- <a href="https://github.com/weka/csi-wekafs">`WekaFS CSI`</a>：高性能 AI 原生存储

**最佳实践：**

- 在部署前使用 DaemonSet 在 GPU 节点上预拉取模型镜像，以消除冷启动模型加载延迟
- 使用 `ReadOnlyMany` PVC 在推理副本间共享模型权重
- 为检查点（高吞吐量）和模型服务（高 IOPS）配置不同的存储类
- 监控存储带宽和 GPU 指标——存储饱和是模型加载和检查点操作的常见瓶颈

### 2.8 作为资源层的 GPU 共享（中立观点）

最近的社区讨论显示了一个明确趋势：GPU 管理和 LLM 服务正成为紧密耦合的平台关注点。对于平台团队来说，这意味着 GPU 应被视为可调度的资源层，而不仅仅是节点级设备标志。

| 选项 | 最适合 | 主要权衡 |
| --- | --- | --- |
| MIG（硬件分区） | 强隔离和可预测的切片 | 静态分区，混合工作负载弹性较低 |
| 时间切片 / MPS | 突发推理时更高的利用率 | 隔离较弱，存在噪声邻居风险 |
| HAMI | 具有调度器可见 GPU 分数的多租户共享 | 额外的控制平面复杂性和兼容性验证 |

**何时值得评估 HAMI：**

- 您的需求超出基本的 MIG 分区，需要多租户 GPU 共享
- 您需要调度器可见的共享比例/配额用于混合工作负载
- 您运行异构 GPU 池，需要统一的共享控制

**护栏（保持务实采用）：**

- 将 HAMI 视为可选实现，而非强制架构层
- 先在 staging 中验证兼容性矩阵：Kubernetes 版本、驱动、容器运行时、CUDA 栈和可观测性流水线
- 在生产部署前与基线（`MIG + 原生调度器` 或 `时间切片 + 原生调度器`）进行基准对比
- 在关键集群启用前定义回滚路径到原生 device-plugin/DRA 模型

**相关参考：**

- [KubeCon EU 2026 Day 1 notes (signal, not benchmark): Kubernetes 正在重新理解 AI](https://jimmysong.io/zh/blog/kubecon-eu-2026-day1-ai-infra)
- <a href="https://github.com/Project-HAMi/HAMi">`Project-HAMi`</a>

### 2.9 借用与 LendingLimit

**概念**：允许命名空间/团队从其他团队借用未使用的资源，同时强制执行借出限制以确保公平访问。

**资源借用模型：**

- **弹性配额**：命名空间如果可用，可以使用超过保证量的资源
- **Cohort 基础共享**：同一 cohort 中的团队可以互相借用
- **基于时间的借出**：在非高峰时段借用空闲资源

**参数：**

- **保证配额**：始终可用的最小资源
- **最大配额**：包括借用容量在内的最大资源
- **借出限制**：可以借给他人的最大资源
- **回收策略**：如何回收借出的资源（抢占、优雅方式）

**实现：**

- <a href="https://github.com/kubernetes-sigs/kueue">`Kueue`</a>：具有抢占功能的 Cohort 和借用限制
- <a href="https://github.com/koordinator-sh/koordinator">`Koordinator`</a>：弹性配额管理

**示例（Kueue ClusterQueue）：**

```yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: ClusterQueue
metadata:
  name: team-a-queue
spec:
  namespaceSelector: {}
  resourceGroups:
  - coveredResources: ["cpu", "memory", "nvidia.com/gpu"]
    flavors:
    - name: default-flavor
      resources:
      - name: "cpu"
        nominalQuota: 100      # 保证量
        borrowingLimit: 50     # 最多可借用 50
        lendingLimit: 30       # 最多可借出 30
      - name: "nvidia.com/gpu"
        nominalQuota: 10
        borrowingLimit: 5
        lendingLimit: 3
```

**最佳实践：**

- 设置借出限制以防止资源饥饿（例如最多借出保证量的 30%）
- 使用抢占回收借出资源
- 监控借用模式并根据实际使用调整配额
- 结合优先级类：借出工作负载使用较低优先级

### 2.10 去调度器

**概念**：持续在节点间重新平衡 Pod，以优化利用率、纠正调度错误并适应变化条件。

**去调度策略：**

- **RemoveDuplicates**：确保每个 ReplicaSet 在每个节点上只有一个 Pod
- **LowNodeUtilization**：将 Pod 从利用率低的节点移到利用率高的节点
- **RemovePodsViolatingNodeAffinity**：修复亲和性违规
- **RemovePodsViolatingInterPodAntiAffinity**：修复反亲和性违规
- **RemovePodsViolatingTopologySpreadConstraint**：重新平衡拓扑分布
- **RemoveFailedPods**：清理失败的 Pod
- **RemovePodsHavingTooManyRestarts**：移除崩溃循环的 Pod

**实现：**

- <a href="https://github.com/kubernetes-sigs/descheduler">`Descheduler`</a>：Kubernetes SIG 项目，用于 Pod 驱逐
- <a href="https://github.com/koordinator-sh/koordinator">`Koordinator`</a>：内置去调度，具有负载感知重新平衡

**配置示例：**

```yaml
apiVersion: descheduler/v1alpha2
kind: DeschedulerPolicy
profiles:
- name: default
  pluginConfig:
  - name: LowNodeUtilization
    args:
      thresholds:
        cpu: 20
        memory: 20
      targetThresholds:
        cpu: 50
        memory: 50
  - name: RemovePodsViolatingTopologySpreadConstraint
    args:
      constraints:
      - topologyKey: topology.kubernetes.io/zone
```

**最佳实践：**

- 定期运行去调度器（每 10-30 分钟）而非持续运行
- 设置 Pod 中断预算（PDB）以限制驱逐影响
- 使用命名空间/标签选择器排除关键 Pod
- 监控去调度对应用 SLA 的影响
- 在应用前使用 dry-run 模式预览驱逐

### 2.11 基于 SLA 的调度

**概念**：使用数字阈值根据服务级别协议（SLA）要求调度 Pod，使工作负载能够放置在满足特定可靠性和性能标准的节点上。

Kubernetes 正在通过 [KEP-5471: Extended Toleration Operators for Threshold-Based Placement](https://github.com/kubernetes/enhancements/pull/5473) 引入原生的基于 SLA 的调度能力。这一增强使 Pod 能够在容忍度中使用数值比较来表达 SLA 要求，提供 Kubernetes 原生的 SLA 感知调度机制。

**关键能力：**

- **数值比较运算符**：扩展容忍度以支持 `Lt`（小于）和 `Gt`（大于）运算符，用于匹配具有数值的节点污点
- **SLA 阈值放置**：允许 Pod 指定最低 SLA 要求（例如"仅在 SLA > 95% 的节点上调度"）
- **驱逐支持**：结合 `NoExecute` 污点效果，在节点 SLA 低于阈值时自动驱逐 Pod
- **特性门控**：`TaintTolerationComparisonOperators`（Kubernetes v1.35 中 Alpha）

**使用场景：**

对于大规模部署，基于 SLA 的调度提供了对工作负载放置的细粒度控制：

- **关键工作负载**：需要高 SLA 节点（>95%）以最小化延迟并确保一致性能
- **尽力而为工作负载**：容忍较低 SLA 节点以优化成本，同时保持可接受的性能
- **异构集群**：支持混合按需（高 SLA）和 spot（低 SLA）节点，根据可靠性要求自动引导工作负载
- **AI/ML 工作负载**：prefill 阶段可能需要高 SLA 节点以优化 TTFT，而 decode 阶段可能容忍较低 SLA 节点

**配置示例：**

```yaml
# 具有 95% SLA 污点的高 SLA 按需节点
apiVersion: v1
kind: Node
metadata:
  name: ondemand-gpu-node-1
spec:
  taints:
  - key: node.kubernetes.io/sla
    value: "950"  # 95.0% SLA（乘以 10）
    effect: NoExecute
---
# 关键 Pod 要求 SLA > 95%，支持驱逐
apiVersion: v1
kind: Pod
metadata:
  name: critical-worker
spec:
  tolerations:
  - key: node.kubernetes.io/sla
    operator: Gt
    value: "950"
    effect: NoExecute
    tolerationSeconds: 30  # 驱逐前宽限期
  containers:
  - name: app
    image: critical-app:latest
---
# 尽力而为 Pod 容忍较低 SLA 以节省成本
apiVersion: v1
kind: Pod
metadata:
  name: batch-worker
spec:
  tolerations:
  - key: node.kubernetes.io/sla
    operator: Gt
    value: "800"  # 80% SLA 可接受
    effect: NoExecute
    tolerationSeconds: 60
  containers:
  - name: batch
    image: batch-app:latest
```

**与其他策略的集成：**

基于 SLA 的调度补充了其他优化策略：

1. **运行时放置控制**：确保工作负载放置在满足其可靠性要求的节点上
2. **动态适应**：`NoExecute` 效果的污点触发节点 SLA 下降时的自动 Pod 驱逐，启用优雅故障转移
3. **成本优化**：通过在适当的 SLA 层级放置工作负载，平衡性能要求与基础设施成本
4. **拓扑感知**：结合拓扑感知调度，同时满足物理放置和 SLA 要求

**最佳实践：**

- 使用一致的 SLA 值缩放（例如百分比乘以 10）
- 根据工作负载关键性设置适当的 `tolerationSeconds`
- 监控节点 SLA 指标并动态更新污点
- 结合 PriorityClasses 实现全面工作负载管理
- 使用抢占策略确保关键工作负载获得高 SLA 节点
- 在负载下测试 SLA 阈值以验证正确的工作负载分布

---

## 学习主题

- **调度器架构**：Queue、Cache、Framework、Plugins
- **调度算法**：Binpack、Spread、Load-aware、Topology-aware、SLA-based
- **多租户**：资源配额、优先级、公平共享
- **调度指标**：吞吐量、延迟、利用率、队列深度
- **冲突解决**：乐观并发、重试、退避

## 项目和资源

**调度器：**

- <a href="https://github.com/kubernetes/kubernetes">`Kubernetes Default Scheduler`</a>：通过调度框架可扩展
- <a href="https://github.com/volcano-sh/volcano">`Volcano`</a>：CNCF 孵化，批处理调度
- <a href="https://github.com/kubernetes-sigs/kueue">`Kueue`</a>：Kubernetes SIG 项目，作业排队
- <a href="https://github.com/koordinator-sh/koordinator">`Koordinator`</a>：CNCF Sandbox，QoS 和混合部署
- <a href="https://github.com/kubewharf/godel-scheduler">`Godel Scheduler`</a>：字节跳动，高吞吐量调度
- <a href="https://github.com/apache/yunikorn-core">`Apache YuniKorn`</a>：Apache，DRF 和多租户
- <a href="https://github.com/NVIDIA/kai-scheduler">`NVIDIA Kai Scheduler`</a>：NVIDIA，GPU 优化调度
- <a href="https://github.com/ai-dynamo/grove">`NVIDIA Grove`</a>：AI 工作负载的 Gang 调度

**设备管理：**

- <a href="https://github.com/kubernetes/dynamic-resource-allocation/">`DRA`</a>：Kubernetes 动态资源分配
- <a href="https://github.com/containerd/nri">`NRI`</a>：containerd 的节点资源接口
- <a href="https://github.com/Project-HAMi/HAMi">`HAMi`</a>：CNCF Sandbox，GPU 共享

**监控：**

- <a href="https://github.com/kubernetes-sigs/scheduler-plugins">`Scheduler Plugins`</a>：树外调度器插件
- <a href="https://github.com/kubernetes-sigs/descheduler">`Descheduler`</a>：Kubernetes SIG 项目

**文档：**

- [Kubernetes Scheduling Framework](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/)
- [Priority and Preemption](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/)
- [Dynamic Resource Allocation (DRA)](./dra.md)

## 路线图（进行中的提案）

- Kubernetes 中的 Gang 调度支持 [#4671](https://github.com/kubernetes/enhancements/pull/4671)
- LWS Gang 调度 [KEP-407](https://github.com/kubernetes-sigs/lws/blob/main/keps/407-gang-scheduling/README.md)
- DRA：结构化参数 [#4381](https://github.com/kubernetes/enhancements/issues/4381)（v1.34 中 GA）
- 基于 SLA 的调度：扩展容忍度运算符 [KEP-5471](https://github.com/kubernetes/enhancements/pull/5473)（v1.35 中 Alpha）

## 最佳实践总结

1. **从可观测性开始**：在优化前测量当前调度性能和利用率
2. **增量优化**：首先解决最大的瓶颈（API、etcd、准入 webhook）
3. **平衡吞吐量与利用率**：高并发提高吞吐量但可能降低利用率
4. **使用适当的工具**：匹配工作负载的调度器（批处理、服务、训练）
5. **持续监控**：跟踪调度延迟、队列深度、利用率、冲突
6. **大规模测试**：使用实际工作负载模式和集群规模验证优化

---

## 避免调度开销

在某些情况下，最佳优化是通过原地更新 Pod 而非重建来完全避免调度。这消除了调度延迟和 Pod 启动时间。

### Vertical Pod Autoscaler（VPA）

**概念**：自动调整运行中 Pod 的 CPU 和内存请求/限制而无需重启（使用原地 resize 时）。

**原地 Pod Resize（KEP-1287）：**

Kubernetes v1.27+ 支持不重启 Pod 的情况下原地调整 Pod 资源请求和限制，通过 `InPlacePodVerticalScaling` 特性门控启用。

**优势：**

- **无调度开销**：Pod 保留在同一节点上，无需调度器参与
- **无启动时间**：容器以调整后的资源继续运行
- **最小中断**：特别适用于启动时间长的有状态工作负载
- **更快伸缩**：资源调整在数秒内应用，而 Pod 重建需要数分钟

**使用示例：**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: inference-worker
spec:
  containers:
  - name: model-server
    image: inference-engine:latest
    resources:
      requests:
        memory: "4Gi"
        cpu: "2"
      limits:
        memory: "8Gi"
        cpu: "4"
    resizePolicy:
    - resourceName: cpu
      restartPolicy: NotRequired  # 无需重启调整
    - resourceName: memory
      restartPolicy: RestartContainer  # 如果需要可能重启
```

**用于原地更新的 VPA 配置：**

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: inference-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: inference-deployment
  updatePolicy:
    updateMode: "Auto"  # 自动应用建议
  resourcePolicy:
    containerPolicies:
    - containerName: model-server
      mode: Auto
      minAllowed:
        cpu: "1"
        memory: "2Gi"
      maxAllowed:
        cpu: "8"
        memory: "16Gi"
```

**最佳实践：**

- 用工作负载测试原地 resize（部分应用不能优雅处理资源变更）
- 设置适当的 min/max 限制以防止过度配置
- 监控资源利用率以验证 VPA 建议
- 对 CPU 使用 `NotRequired` 重启策略（通常安全），对内存进行评估
- 结合 HPA 实现全面自动伸缩（Pod 伸缩 + 资源调整）

### 容器/Pod 原地重启

**概念**：在 Pod 内重启容器而无需重建 Pod 或重新调度，适用于从故障恢复或在保持 Pod 放置和标识的情况下应用某些配置变更。

**Kubernetes KEP：**

- **KEP-5307：容器重启策略**：提供对 Pod 内单个容器重启行为的细粒度控制
- **KEP-5532：容器退出时重启所有容器**：当特定容器退出时启用重启 Pod 中所有容器，适用于协调重启

**使用场景：**

- **崩溃恢复**：自动重启失败的容器而无需 Pod 重新调度
- **协调重启**：当依赖失败时一起重启所有容器
- **容器级策略**：对 sidecar 和主容器应用不同的重启行为
- **调试/故障排除**：无需调度开销的更快恢复

**容器重启规则：**

Kubernetes 支持在容器级别配置重启行为：

- **容器重启策略**：控制单个容器在失败时是否重启
- **Pod 重启策略**：`Always`、`OnFailure`、`Never` 仍在 Pod 级别适用
- **协调重启**：KEP-5532 启用在特定容器退出时重启所有容器

**优势：**

- **无调度开销**：容器在同一节点上重启，无需调度器参与
- **保留 Pod 标识**：Pod IP、主机名和卷保持不变
- **更快恢复**：容器重启比完整 Pod 重建更快
- **细粒度控制**：Pod 内每个容器可以有不同的重启策略

#### 实际用例：JobSet 原地重启

<a href="https://github.com/kubernetes-sigs/jobset/">JobSet</a> 正在实现原地重启功能，展示了避免 Pod 重建和重新调度的显著性能优势。根据[基准测试原型](https://github.com/kubernetes-sigs/jobset/compare/main...GiuseppeTT:jobset:in-place-restart-prototype)，原地重启实现了：

- **重启时间减少**：从 2m10s 降至 10s（快 92%）
- **测试规模**：5000 节点集群
- **优势**：分布式批处理工作负载的显著更快恢复

展示了原地重启特别适用于：

- **大规模批处理作业**：协调数千 Pod 无需重新调度节省大量时间
- **分布式训练**：快速从故障中恢复而不会丢失集群放置
- **作业依赖**：一起重启相关作业，同时保持 Pod 标识和网络连接

更多详情请参见 [JobSet 原地重启设计文档](https://docs.google.com/document/d/16zexVooHKPc80F4dVtUjDYK9DOpkVPRNfSv0zRtfFpk/edit?tab=t.0#heading=h.y6xl7juq7465)。

**示例 - 具有容器重启策略的 Pod：**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-app
spec:
  restartPolicy: Always  # Pod 级策略
  containers:
  - name: main-app
    image: app:latest
    # 容器将基于 Pod 级策略重启
  - name: sidecar
    image: sidecar:latest
    # KEP-5307：未来支持容器级重启策略
```

**权衡：**

- **优点**：消除调度延迟，更快恢复
- **缺点**：节点本地资源约束可能需要 Pod 重新调度（如果资源耗尽）
- **优点**：保留 Pod IP 和网络标识
- **缺点**：不重新平衡集群负载

**何时使用：**

- 启动时间长的有状态工作负载（数据库、缓存）
- 模型加载成本高的 GPU 工作负载
- 具有 sidecar 依赖的多容器 Pod
- 快速故障恢复而不中断 Pod 放置

**参考：**

- [KEP-5307：容器重启策略](https://github.com/kubernetes/enhancements/blob/master/keps/sig-node/5307-container-restart-policy/README.md)
- [KEP-5532：容器退出时重启所有容器](https://github.com/kubernetes/enhancements/blob/master/keps/sig-node/5532-restart-all-containers-on-container-exits/README.md)

---

*部分内容由 AI 辅助生成。请针对您的具体使用场景进行验证。*
