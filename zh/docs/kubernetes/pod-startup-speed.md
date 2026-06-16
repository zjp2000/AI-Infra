---
status: Active
maintainer: pacoxu
last_updated: 2025-10-29
tags: kubernetes, pod, startup, optimization, performance
canonical_path: zh/docs/kubernetes/pod-startup-speed.md
---

# Pod 启动速度优化

本指南提供了加速 Kubernetes 集群中 Pod 启动的综合策略，特别适用于大规模部署和 AI/GPU 工作负载。基于 KubeCon 2023 China 上 Paco Xu（DaoCloud）和 Byron Wang（Birentech）的演讲。

## 为什么 Pod 启动速度很重要

快速 Pod 启动对于以下方面至关重要：

- **快速伸缩**：快速响应负载增加
- **高效资源利用**：最小化空闲时间
- **更好用户体验**：减少服务等待时间
- **成本优化**：更快的迭代和部署周期
- **AI/GPU 工作负载**：快速模型加载和推理启动

## 典型 Pod 启动时间

Pod 启动时间因复杂度而异：

- **< 50ms**：使用本地镜像的最小 Pod
- **~ 100ms**：使用缓存镜像的简单 Pod
- **~ 秒级**：具有中等初始化过程的标准应用
- **15s - 1m**：具有大量初始化或大镜像的应用
- **< 5m**：具有广泛数据加载的复杂应用

## Pod 启动流水线

Pod 启动涉及四个主要阶段：

1. **Pod 创建**：API Server 处理 Pod 创建请求
2. **Pod 调度**：Scheduler 将 Pod 分配到节点
3. **节点上 Pod 启动**：Kubelet 创建并启动容器
4. **可观测性**：监控和测量启动性能

---

## 1. Pod 创建优化

### API Server 性能

**常见问题：**

- etcd 性能缓慢
- API Server 负载过高
- 达到速率限制
- 准入 webhook 缓慢

**解决方案：**

- 监控并优化 etcd 性能
- 水平扩展 API Server 以应对高负载
- 审查并优化准入 webhook 性能
- 根据工作负载适当调整速率限制

### Pod 创建失败

**常见原因：**

- **验证失败**：无效的 Pod 规格
- **Pod 安全准入**：安全策略阻止 Pod 创建
- **资源配额**：命名空间资源限制已达
- **Pod 优先级和抢占**：低优先级 Pod 等待资源

**解决方案：**

- 在提交前验证 Pod 规格
- 审查并调整 Pod 安全策略
- 监控命名空间资源配额
- 使用适当的 Pod 优先级类

---

## 2. Pod 调度优化

### 默认调度器

如果未指定 `pod.spec.nodeName`，调度器必须找到合适的节点。

**影响调度的因素：**

- **失败/重试**：Pod 调度失败和重试逻辑
- **规模**：集群中的节点数量
- **条件**：节点亲和性、污点、容忍度
- **负载**：同时调度的 Pod 数量

### 高级调度策略

#### 负载感知调度

- **Trimaran**：节点感知调度插件
- Pod 在负载最轻的节点上启动
- 减少资源争用

#### 分散策略

- 将 Pod 分布到多个节点
- 减少 Pod 启动期间的相互干扰
- 更好的资源分布

### GPU 调度

对于 AI/GPU 工作负载，需要专门的调度算法：

**GPU 调度器算法：**

- **Gang 调度**：组内所有 Pod 一起调度或全部不调度
- **SKU 感知**：在调度决策中考虑 GPU SKU/类型
- **Binpack**：在更少的节点上最大化 GPU 利用率
- **DRF（Dominant Resource Fairness）**：用户间的公平资源分配

**GPU 调度挑战：**

1. 设备编号或属性变化
2. 同一供应商的不同设备类型
3. 集群中多个供应商的设备
4. 将设备分配给多个 Pod 或容器
5. 分配特定的选定设备

**解决方案：**

- GPU 的自定义资源定义（例如 `birentech.com/gpu`）
- 动态资源分配（DRA）
- GPU 感知调度器（HAMI、NVIDIA Kai Scheduler、Grove）

---

## 3. 节点上 Pod 启动优化

### 简化的 Pod 启动步骤

基本 Pod 启动过程：

1. 拉取镜像
2. 创建沙箱
3. CNI（网络初始化）
4. 挂载卷或 Secret/ConfigMap
5. 容器主进程启动

**简化策略：**

- 提前拉取镜像
- 在适当的情况下使用 `hostNetwork: true`
- 最小化卷挂载
- 对于启动关键的 Pod，避免 CPU/内存限制

### 镜像优化

#### 使镜像更小

**最佳实践：**

- 添加 `.dockerignore` 排除不必要的文件
- 选择更小的基础镜像：`alpine`、`scratch`
- 使用多阶段构建排除构建时依赖
- 减少层：合并层或使用 squash
- 不要安装调试工具：使用 `kubectl debug` 和临时容器

**工具：**

- <a href="https://github.com/wagoodman/dive">`wagoodman/dive`</a>：用于探索 Docker 镜像每一层的工具

#### P2P 和延迟拉取

**镜像分发解决方案：**

- <a href="https://github.com/dragonflyoss/dragonfly2">`Dragonfly`</a>：基于 P2P 的开源文件分发和镜像加速系统
- <a href="https://github.com/dragonflyoss/nydus">`Nydus`</a>：Dragonfly 容器镜像服务，支持延迟拉取
- <a href="https://github.com/spegel-org/spegel">`Spegel`</a>：无状态集群本地 OCI registry 镜像（轻量级）
- <a href="https://github.com/uber/kraken">`uber/kraken`</a>：P2P Docker registry，能够在几秒内分发 TB 级数据

**Containerd 集成：**

- <a href="https://github.com/containerd/stargz-snapshotter">`containerd/stargz-snapshotter`</a>：使用 Nydus Snapshotter 进行延迟拉取

#### 并行镜像拉取

默认情况下，kubelet 每次只发送一个镜像拉取请求。

**配置：**

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
serializeImagePulls: false
maxParallelImagePulls: 5  # v1.27 起可用
```

**为什么 `serializeImagePulls` 默认为 true？**

- Docker v1.9 之前 aufs 的老问题
- containerd 1.6.3 和 1.7.0 之前的性能问题
- 欢迎在 <a href="https://github.com/kubernetes/kubernetes/issues/108405">#108405</a> 提供反馈

### Init 容器优化

**最佳实践：**

- 只做必要的初始化
- 尽可能并行运行 init 容器
- 在适当的情况下使用 `postStart` 钩子
- 尽早做准备：
  - 这可以在镜像构建时完成吗？
  - 这可以通过 DaemonSet/Pod 在节点上完成吗？

### CPU 管理

#### 避免启动时 CPU 节流

**策略：**

- 在开始时使用更大的 CPU 限制
- 避免初始化期间的 CPU 节流

**更好的解决方案（v1.27+）：**

- 使用 Vertical Pod Autoscaler（VPA）和原地 Pod 垂直伸缩：

```bash
--feature-gates=InPlacePodVerticalScaling=true
```

#### 静态 CPU 策略

对于需要专用 CPU 核心的保证 QoS Pod：

**更改 CPU Manager 策略的步骤：**

1. 排空节点
2. 停止 kubelet
3. 删除旧的 CPU manager 状态文件：`/var/lib/kubelet/cpu_manager_state`
4. 编辑 kubelet 配置以设置所需的 CPU manager 策略
5. 启动 kubelet

**⚠️ 限制：**

- 参见 <a href="https://github.com/kubernetes/kubernetes/issues/116086">#116086</a>
- 如果您的 Pod 有 sidecar 容器，sidecar 的 CPU 限制/请求必须是 `Guaranteed`

### 探针配置

**启动探针：**

- 如果设置，减少就绪探针的 `initialDelaySeconds`
- 保护慢启动容器

**就绪探针：**

- 如果没有配置启动探针，设置 `initialDelaySeconds`
- 防止过早的路由流量

**未来路线图：**

- 亚秒级/更细粒度的探针：毫秒级精度
- 参考：<a href="https://www.padok.fr/en/blog/kubernetes-probes">Kubernetes Probes Guide</a>

### 取证容器 Checkpointing

对于由于以下原因启动缓慢的 Pod：

- 将大量数据加载到内存中
- 耗时的初始化过程

**解决方案：取证容器 Checkpointing**（v1.25 中 Alpha）

**工作原理：**

1. 创建完全初始化 Pod 的快照
2. 存储快照以便快速重用
3. 在后续启动时从 checkpoint 恢复

**优势：**

- 对用户请求的瞬时响应
- 高效的资源利用
- 跨重启的一致性性能

### gVisor 快照用于 AI Agent

对于需要快速启动和强隔离的 AI Agent 工作负载：

**解决方案：带容器快照的 gVisor**（GKE 生产可用）

**工作原理：**

1. 初始化容器，包含依赖、库和运行时状态
2. 创建预预热容器的 gVisor 沙箱快照
3. 在毫秒级从快照恢复新实例
4. 每个实例在隔离的 gVisor 沙箱中运行

**性能影响：**

- **冷启动时间提升高达 90%**，相比传统容器
- 复杂 Agent 工作负载的亚秒级启动
- 消除重复 Agent 调用的初始化开销

**使用场景：**

- **LLM Agent**：在安全沙箱中快速执行 LLM 生成的代码
- **函数调用**：快速调用隔离的 Agent 函数
- **Serverless AI**：无服务器 Agent 平台的近乎即时冷启动
- **多租户 AI**：客户 Agent 工作负载的安全快速启动

**可用性：**

- 在 <a href="https://cloud.google.com/blog/products/containers-kubernetes/agentic-ai-on-kubernetes-and-gke">GKE Agent Sandbox</a> 中生产就绪
- 结合 gVisor 安全性和快照性能
- 参见[工作负载隔离指南](./isolation.md#6-agent-sandbox-kubernetes-sig-project)了解详细安全架构

### 通用因素

**容器运行时：**

- 优先使用 containerd 而非 Docker 以获得更好性能
- containerd 是现代 Kubernetes 的默认运行时

**CNI 性能：**

- 大多数节点上的 IPAM 非常快
- 为工作负载选择合适的 CNI 插件

**SELinux 优化（v1.27+）：**

- 使用挂载选项的 SELinux 重新标记（Beta）
- 使用正确的 SELinux 标签挂载卷，而不是递归更改文件
- 对于大卷显著更快

**Kubelet 性能（v1.27+）：**

- 默认 API QPS 限制提高了 10 倍
- 事件触发的容器状态更新（Evented PLEG 为 Beta）

---

## 4. AI/GPU 特定优化

**关于 GPU Pod 冷启动优化策略的全面指南，请参见专门文档：[GPU Pod 冷启动优化](./gpu-pod-cold-start.md)**

本节涵盖了 Pod 启动的 GPU 特定优化，重点关注资源分配、调度和数据加载。关于最小化 GPU Pod 冷启动延迟的详细策略，包括预预热池、更快模型序列化和延迟加载模式，请参考 GPU 冷启动指南。

### 动态资源分配（DRA）

DRA 提供灵活的 GPU 资源管理：

**能力：**

1. 多资源参数
2. 网络附加资源
3. 初始化和清理策略
4. 用户友好的 API
5. 用于资源管理的集群附加组件
6. 复杂的分配规则

**注意事项：**

1. 可能减慢 Pod 调度
2. 描述 Pod 需求的额外复杂性

### GPU 管理方法

**四种主要方法：**

1. **新的容器运行时**：支持 GPU 的自定义运行时
2. **CRI 代理**：拦截 CRI 调用以添加 GPU 功能
3. **修改 kubelet 核心代码**：直接修改 kubelet
4. **DRA 和 CDI**：使用动态资源分配和容器设备接口

### 加速 AI/GPU 数据加载

**典型数据加载流水线：**

1. **准备**：在存储中暂存数据
2. **复制到内存**：从存储加载数据到主机内存
3. **复制到 GPU 内存**：从主机传输数据到 GPU 内存
4. **GPU 计算**：执行计算

**优化策略：**

- 在节点上预加载数据
- 使用快速存储（NVMe、本地 SSD）
- 优化数据传输流水线
- 考虑使用 GPU Direct Storage 实现直接 GPU 访问

### 假设备和自定义调度器

用于测试和开发：

- 使用假设备模拟 GPU 环境
- 为特殊需求实现自定义调度器
- 在没有真实硬件的情况下测试伸缩和性能

---

## 5. 可观测性

### 指标

**关键指标（v1.26+）：**

```text
pod_start_sli_duration_seconds
```

**日志示例：**

```text
"Observed pod startup duration"
pod="kube-system/konnectivity-agent-gnc9k"
podStartSLOduration=-9.223372029479458e+09
pod.CreationTimestamp="2022-12-30 15:33:06"
firstStartedPulling="2022-12-30 15:33:09"
lastFinishedPulling="0001-01-01 00:00:00"
observedRunningTime="2022-12-30 15:33:13"
watchObservedRunningTime="2022-12-30 15:33:13"
```

**使用场景：**

- 基于此指标创建告警
- 监控 Pod 启动 SLO 合规性
- 识别启动缓慢的 Pod
- 跟踪启动时间趋势

### Linux 调试工具

使用标准 Linux 性能工具：

- `top`、`htop`：CPU 和内存使用
- `iostat`：I/O 统计
- `perf`：性能分析
- `strace`：系统调用追踪
- `tcpdump`：网络包分析
- `bpftrace`、`bcc`：基于 eBPF 的追踪

---

## 总结

Pod 启动优化是一个多方面的挑战，需要关注：

1. **API Server 和 etcd 性能**
2. **调度器效率和放置决策**
3. **镜像大小和分发策略**
4. **容器运行时和 CNI 性能**
5. **资源分配和 CPU 管理**
6. **AI/GPU 特定考量**
7. **快照和 checkpointing** 用于预热容器
8. **全面的可观测性**

通过系统地应用这些策略，您可以显著减少 Pod 启动时间并提高整体集群性能。对于 AI Agent 工作负载，GKE Agent Sandbox 中的 gVisor 快照可以在冷启动性能上提供高达 90% 的提升，同时保持强隔离保证。

---

## 参考

- KubeCon 2023 China："How Can Pod Start-up Be Accelerated on Nodes in Large Clusters?" by Paco Xu (DaoCloud) and Byron Wang (Birentech)
- <a href="https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/">Pod Lifecycle Official Documentation</a>
- <a href="https://github.com/kubernetes/kubernetes/issues/108405">Kubernetes Issue #108405: Serial Image Pulls</a>
- <a href="https://github.com/kubernetes/kubernetes/issues/116086">Kubernetes Issue #116086: CPU Manager Limitations</a>
- <a href="https://www.padok.fr/en/blog/kubernetes-probes">Kubernetes Probes Guide</a>
- <a href="https://cloud.google.com/blog/products/containers-kubernetes/agentic-ai-on-kubernetes-and-gke">GKE Agent Sandbox: Strong Guardrails for Agentic AI on Kubernetes</a>
