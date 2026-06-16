---
status: Active
maintainer: pacoxu
last_updated: 2026-05-09
tags: kubernetes, dra, resource-allocation, device-management, topology
canonical_path: zh/docs/kubernetes/dra.md
---

# 动态资源分配（DRA）

[DRA](https://github.com/kubernetes/dynamic-resource-allocation/) 是 Kubernetes 的动态资源分配框架，支持灵活的设备分配，具有结构化参数和拓扑感知能力。

## 概述

DRA 提供了传统设备插件框架的更灵活替代方案，支持：

- 复杂设备需求（GPU + NIC + 存储组合）
- 拓扑感知调度（NUMA、NVLink、InfiniBand）
- 细粒度设备配置的结构化参数
- 每个 Pod 支持多种设备类型

![dra](../../diagrams/dra-user-flow.svg)

## DRA KEP 架构视图

DRA 现在不仅是一个声明分配 API。活跃的 KEP 最好被理解为一组连接的控制循环：工作负载意图、API 状态、调度器适配、节点端准备以及应用和运维人员的反馈。

![DRA KEP architecture](../../diagrams/dra-kep-architecture.svg)

该图将 KEP 按架构通道而非发布顺序排列：

- **请求面**：工作负载级声明、扩展资源兼容性和 AdminAccess 决定谁可以请求 DRA 资源以及如何从设备插件迁移。
- **API 状态**：ResourceClaim 状态、Pod 状态、Downward API 和可用性摘要使分配在调度后可观测。
- **调度和绑定**：优先级替代方案、绑定条件、污点、类型化属性、可消费容量和原生资源定义调度器如何判断声明是否适配。
- **节点和驱动循环**：DRA 驱动发布 ResourceSlice、通过 kubelet 准备设备，并通过 PodResources、CDI、NRI 和驱动特定钩子暴露结果。
- **运维反馈**：健康、拓扑、容量和声明状态为调试、自动伸缩和容量规划形成闭环。

# NVIDIA GPU DRA

需要两个 DRA kubelet 插件：gpu-kubelet-plugin、compute-domain-kubelet-plugin。支持 NVIDIA GB200。未来该驱动将包含在 NVIDIA GPU Operator 中。

要求：

- Kubernetes v1.32 或更新版本。
- 在容器运行时中启用 CDI（containerd 2.0 默认启用）。
- NVIDIA GPU 驱动 565 或更高版本。GPU 驱动安装必须包含 nvidia-imex 和 nvidia-imex-ctl 二进制文件。
- 安装 GPU Operator v25.3.x 或更高版本。

![dra-driver](../../diagrams/dra-driver-architecture.png)

**另见：**
[调度优化指南](./scheduling-optimization.md#26-拓扑感知调度)了解 DRA 在生产场景中的使用。

## KEP 和路线图

- DRA：结构化参数 [#4381](https://github.com/kubernetes/enhancements/issues/4381)。v1.34 中 GA。
- 架构图中的当前 DRA KEP：

| 架构通道 | KEP | 意图 |
| --- | --- | --- |
| 请求面 | [#5729](https://github.com/kubernetes/enhancements/issues/5729)、[#5004](https://github.com/kubernetes/enhancements/issues/5004)、[#5018](https://github.com/kubernetes/enhancements/issues/5018) | 让工作负载、传统扩展资源和管理员创建适当类型的 DRA 声明。 |
| API 状态和工作负载反馈 | [#4817](https://github.com/kubernetes/enhancements/issues/4817)、[#4680](https://github.com/kubernetes/enhancements/issues/4680)、[#5304](https://github.com/kubernetes/enhancements/issues/5304) | 将分配的设备状态、健康、网络数据和属性发布回 Pod 和容器。 |
| 调度和绑定 | [#4816](https://github.com/kubernetes/enhancements/issues/4816)、[#5007](https://github.com/kubernetes/enhancements/issues/5007)、[#5055](https://github.com/kubernetes/enhancements/issues/5055) | 改善稀缺设备的适配决策，避免在外部设备就绪前绑定 Pod。 |
| 资源模型 | [#4815](https://github.com/kubernetes/enhancements/issues/4815)、[#5075](https://github.com/kubernetes/enhancements/issues/5075)、[#5491](https://github.com/kubernetes/enhancements/issues/5491)、[#5517](https://github.com/kubernetes/enhancements/issues/5517) | 表示分区设备、共享容量、多值拓扑属性和原生资源。 |
| 节点和可观测性循环 | [#3695](https://github.com/kubernetes/enhancements/issues/3695)、[#5677](https://github.com/kubernetes/enhancements/issues/5677) | 通过 kubelet PodResources 暴露 DRA 分配并总结剩余池容量。 |

- 所有关于 DRA 的 KEP：[GitHub Issues](https://github.com/kubernetes/enhancements/issues/?q=is%3Aissue%20%20DRA%20in%3Atitle)

**性能测试：**

参见 [DRA 性能测试](./dra-performance-testing.md)了解全面的规模测试、性能基准和生产最佳实践。

**驱动生态矩阵：**

参见 [公共 Kubernetes DRA 驱动功能矩阵](./dra-driver-feature-matrix.md)了解仓库级视图，显示哪些公共实现对关键 Beta/GA DRA 功能有显式支持、正在进行中或尚无公开信号。

## 近期上游发布说明

理解 DRA 演变最快的方式是将发布博客作为从"更好的 API 面"到"真实生产控制循环"的进程来阅读：

| 发布 | 主要信号 | AI 基础设施团队为何关注 |
| --- | --- | --- |
| [v1.33 DRA 更新](https://kubernetes.io/blog/2025/05/01/kubernetes-v1-33-dra-updates/) | DRA 在 Beta 阶段持续成熟；驱动拥有的声明状态改善，围绕优先级替代方案、设备污点和管理员访问的新 Alpha 工作启动。 | 这是 DRA 开始超越"设备计数"并走向运维工作流和更丰富状态的时刻。 |
| [v1.34 DRA 变得更强大](https://kubernetes.io/blog/2025/09/01/kubernetes-v1-34-dra-updates/) | 核心 DRA 达到 GA 并添加了更强的运维组件，如 AdminAccess、PodResources 可见性、扩展资源迁移、可消费容量、绑定条件和资源健康。 | 这是第一次从设备插件到 DRA 的生产迁移规划变得现实。 |
| [v1.36：更多驱动、新功能和 DRA 的下一个时代](https://kubernetes.io/blog/2026/05/07/kubernetes-v1-36-dra-136-updates/) | DRA 向更多驱动扩展，并添加了工作负载级声明、原生资源实验、更好的资源池可见性、确定性选择和可发现的设备元数据。 | 范围明显从 GPU 扩展到 NIC、CPU、内存和更大的工作负载编排。 |

与本地 AI-Infra 笔记一起阅读：

- [Kubernetes v1.36 DRA 的整体设计：从请求入口到调度、状态与拓扑](../blog/2026-04-23/2026-04-23-kubernetes-v1.36-dra-ai-infra_zh.md) - 此笔记现在也涵盖了 2026-05-07 的官方 DRA 跟进。

## v1.36 AI 基础设施功能快照

2026-05-07 上游 DRA 博客很有用，因为它将 v1.36 变更分为两类：实际改善 day-2 运维的特性，以及开启下一波平台设计的特性。

### 面向生产的改进

- **优先级列表（Stable）**：让声明表达有序的回退偏好，如"H100 first, then A100"，提高稀缺加速器利用率。
- **扩展资源支持（Beta）**：让现有的 `vendor.com/device: N` 风格工作负载在后台分配路径迁移到 DRA 时逐步过渡。
- **可分区设备（Beta）**：将 MIG 类或其他逻辑切片建模为原生 DRA 行为，而非静态预划分。
- **设备污点和容忍度（Beta）**：隔离故障或预留设备，无需排空整个节点。
- **设备绑定条件（Beta）**：避免在可附加或结构支持的设备实际准备好之前绑定 Pod。
- **资源健康状态（Beta）**：在 Pod 状态中暴露设备健康状态，实现更快的故障归因和控制器驱动的修复。

### 值得实验室验证的新方向

- **工作负载的 ResourceClaim 支持（Alpha）**：将声明共享从单个 Pod 扩展到 PodGroup 或工作负载级语义，用于大型分布式作业。
- **节点可分配资源（Alpha）**：开始探索 DRA 作为 CPU 和内存放置的控制平面，而不仅是外部加速器。
- **DRA 资源可用性可见性（Alpha）**：引入资源池容量快照的时间点视图，用于仪表板和容量规划。
- **属性的列表类型**：当设备属性是集合而非单一标量值时，改进 CEL 匹配。
- **确定性设备选择**：调度器顺序变得可预测，赋予驱动作者对最终放置质量的更多影响。
- **容器中可发现的设备元数据**：标准化驱动如何将 PCI 地址和网络配置等属性暴露到容器中，无需额外 API 调用。

## 使用 DRA 的拓扑管理

DRA 通过设备属性和约束实现复杂的拓扑感知调度。这对于需要 GPU + NIC 在同一 PCIe 结构上协调的 AI 工作负载至关重要。

### DRA 如何处理拓扑

DRA 使用**属性**和**约束**配合 CEL（通用表达式语言）表达拓扑要求：

1. **设备属性**：每个设备发布拓扑信息
   - `pcieRoot`：PCIe 层次结构标识符
   - `numaNode`：NUMA 节点关联
   - `nvlinkDomain`：NVLink 结构标识符
   - `rdmaDevice`：关联的 RDMA NIC

2. **约束**：执行拓扑规则的 CEL 表达式
   - GPU 和 NIC 共享同一 PCIe root
   - CPU 和内存共享同一 NUMA 节点
   - GPU 间的 NVLink 连接

3. **SharedID**：同一拓扑域上的设备获得共享标识符

### GPU + NIC 拓扑协调

最有用的用例是在同一 PCIe root 上协调 GPU 和 NIC 分配。这对基于 RDMA 的 GPU-Direct 分布式训练至关重要。

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

### 与 Kueue 和 DRANET 集成

[DRANET](https://github.com/google/dranet) 与 Kueue 的拓扑感知调度使用节点标签集成：

```yaml
# Kueue 和 DRANET 使用的拓扑标签
cloud.google.com/gce-topology-block
cloud.google.com/gce-topology-subblock
cloud.google.com/gce-topology-host
kubernetes.io/hostname
```

### DRA 可消费容量

一个支持灵活资源共享同时保持拓扑感知的新特性。它在结构化参数工作之后引入，跟踪为 [KEP-5075](https://github.com/kubernetes/enhancements/issues/5075)。

- **允许多次分配**跨多个资源请求
- **可消费容量**——保证的资源共享

**使用场景：**

- 虚拟 GPU 内存分区
- 虚拟 NIC（vNIC）共享
- 带宽限制的网络分配
- I/O 带宽智能存储设备共享

#### 实际变化（KEP-5075）

与早期 DRA 设备共享相比，Kubernetes 1.34 为真实多租户使用添加了更具体的共享语义：

- **跨声明共享**：当驱动启用 `allowMultipleAllocations` 时，一个设备（或分区）可以被多个 ResourceClaim 或 DeviceRequest 共享。
- **容量感知调度**：调度器跟踪每个设备的 `capacity` 使用并防止过度分配。
- **DistinctAttribute 约束**：`matchAttribute` 的反向，用于确保请求结果来自不同资源。
- **分配状态中的 ShareID**：驱动可以区分同一底层设备上的多个共享分配。

在控制平面和节点组件上启用的特性门控：

```bash
--feature-gates=...,DRAConsumableCapacity=true
```

驱动侧 ResourceSlice 示例（内存作为可消费容量）：

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceSlice
spec:
  devices:
  - name: gpu0
    allowMultipleAllocations: true
    capacity:
      memory:
        value: 40Gi
        requestPolicy:
          default: 5Gi
          validRange:
            min: 5Gi
            step: 5Gi
```

消费者侧请求示例：

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
spec:
  devices:
    requests:
    - name: req0
      exactly:
      - deviceClassName: resource.example.com
        capacity:
          requests:
            memory: 10Gi
      selectors:
      - cel:
          expression: device.allowMultipleAllocations == true
```

当请求不能落在同一底层设备上时（例如不同子网或故障域），使用 `distinctAttribute`。

### 从设备插件迁移到 DRA

许多组织在基于设备插件的解决方案上有大量投入。迁移到 DRA 面临挑战：

DRA 支持的扩展资源兼容性流程：

![DRA-backed extended resource flow](../../diagrams/dra-extended-resource-flow.svg)

可编辑源：[dra-extended-resource-flow.mmd](../../diagrams/dra-extended-resource-flow.mmd)

**共存问题：**

- **资源冲突**：同一设备由两个系统管理
- **拓扑不一致**：系统间拓扑视图不同
- **调度混乱**：调度器没有统一视图

**功能差距：**

- 设备健康监控（设备插件有内置健康检查）
- 热插拔支持（设备插件支持动态设备添加）
- 指标集成（设备插件的 Prometheus 指标）

**推荐迁移路径：**

1. 在现有设备插件旁部署 DRA 驱动
2. 使用节点污点划分工作负载
3. 逐步将工作负载迁移到基于 DRA 的资源声明
4. 在所有工作负载迁移后淘汰设备插件

**DRA 扩展能力：**

- DRA 驱动可以实现兼容层
- NVIDIA 的 DRA 驱动支持设备插件迁移路径
- NRI 集成可以桥接运行时级差距

**相关博客文章：**
[拓扑感知调度博客](../archive-blog/2025-11-25/2025-11-25-topology-aware-scheduling.md)全面了解 DRA 拓扑管理和迁移。

## DRA 驱动实现

### NVIDIA GPU DRA 驱动

<a href="https://github.com/NVIDIA/k8s-dra-driver-gpu">**NVIDIA GPU DRA 驱动**</a>是 NVIDIA 对 GPU 资源管理的 DRA 参考实现。

**架构：**

该驱动包含两个 kubelet 插件：

- <a href="https://github.com/NVIDIA/k8s-dra-driver-gpu/tree/main/cmd/gpu-kubelet-plugin">**gpu-kubelet-plugin**</a>：使用 DRA 的标准 GPU 资源分配
- <a href="https://github.com/NVIDIA/k8s-dra-driver-gpu/tree/main/cmd/compute-domain-kubelet-plugin">**compute-domain-kubelet-plugin**</a>：用于计算域管理的高级插件

**主要特性：**

- GB200 支持，带有专门的计算域管理（<a href="https://docs.google.com/presentation/d/1Xupr8IZVAjs5bNFKJnYaK0LE7QWETnJjkz6KOfLu87E/edit?pli=1&slide=id.g373e0ebfa8e_1_233#slide=id.g373e0ebfa8e_1_233">架构介绍</a>）
- 拓扑感知 GPU 调度（NVLink、NVSwitch 感知）
- 超越简单 GPU 计数的细粒度资源分配
- 支持复杂的多 GPU 配置

**集成：**

- 将在未来版本中集成到 <a href="./nvidia-gpu-operator.md">NVIDIA GPU Operator</a>
- 提供从传统设备插件到 DRA 的迁移路径

**另见：** [NVIDIA GPU Operator](./nvidia-gpu-operator.md) 了解 Kubernetes 中 NVIDIA GPU 管理的全面介绍。

### CPU 资源 DRA 驱动

<a href="https://github.com/kubernetes-sigs/dra-driver-cpu">**CPU 资源 DRA 驱动**</a>是用于管理 Kubernetes 节点上 CPU 资源的 DRA 参考实现。

**架构：**

该驱动作为 DaemonSet 部署，包含两个核心组件：

- **DRA 驱动**：发现 CPU 拓扑并通过 `ResourceSlice` 对象将可用 CPU 报告为可分配资源。生成 CDI（容器设备接口）规范用于 CPU 集分配。
- **NRI 插件**：通过节点资源接口（NRI）与容器运行时集成，以执行 CPU 绑定和管理共享 CPU 池。

**主要特性：**

- 为通过 ResourceClaim 请求 CPU 的保证 Pod 提供独占 CPU 分配
- 为没有 ResourceClaim 的容器管理共享 CPU 池
- 在保证容器创建或移除时动态更新 CPU 池
- 通过 `--reserved-cpus` 标志预留系统 CPU（与 kubelet 的静态 CPU Manager 策略对齐）
- 在 DaemonSet 重启时同步状态以处理现有 Pod 分配

**配置：**

驱动支持 `--reserved-cpus` 标志指定为系统和 kubelet 进程保留的 CPU（类似于 kubelet 的 `reservedSystemCPUs` 设置）。

**CPU 微拓扑支持：**

最近的开发（[PR #16](https://github.com/kubernetes-sigs/dra-driver-cpu/pull/16)）添加了 CPU 微拓扑支持，包括：

- NUMA 感知 CPU 分配
- 拓扑对齐的 CPU 绑定
- 与 GPU NUMA 放置的协调

**当前限制：**

- 仅 CPU 资源；不支持内存管理

## 学习资源

### 会议演讲

- **KubeCon NA 2025**：[Mind the Topology: Smarter Scheduling for AI Workloads](https://www.youtube.com/watch?v=o5i7pTWZjfo) - Roman Baron, NVIDIA
- **KubeCon NA 2025**：[Device Management Deep Dive](https://www.youtube.com/watch?v=j6zkGxrxm6o) - DRA 和设备插件
- **KubeCon NA 2024**：[Kubernetes WG Device Management - GPUs, TPUs, NICs and More With DRA](https://www.youtube.com/watch?v=Z_15EyXOnhU) - Kevin Klues & Patrick Ohly

### 其他资源

- [Kubernetes 文档](https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/)
- [GKE 文档](https://cloud.google.com/kubernetes-engine/docs/concepts/about-dynamic-resource-allocation)
- [Kubernetes v1.33 DRA 更新](https://kubernetes.io/blog/2025/05/01/kubernetes-v1-33-dra-updates/)
- [Kubernetes v1.34：DRA 变得更强大](https://kubernetes.io/blog/2025/09/01/kubernetes-v1-34-dra-updates/)
- [Kubernetes v1.36：更多驱动、新功能和 DRA 的下一个时代](https://kubernetes.io/blog/2026/05/07/kubernetes-v1-36-dra-136-updates/)
- [AI-Infra：Kubernetes v1.36 DRA 整体设计](../blog/2026-04-23/2026-04-23-kubernetes-v1.36-dra-ai-infra_zh.md)
- [YouTube 搜索：DRA](https://www.youtube.com/@cncf/search?query=DRA)
- [Kubernetes WG Device Management](https://github.com/kubernetes/community/blob/master/wg-device-management/README.md)
- [NVIDIA KAI Scheduler](https://github.com/NVIDIA/KAI-Scheduler) - 具有拓扑感知的 GPU 优化调度
