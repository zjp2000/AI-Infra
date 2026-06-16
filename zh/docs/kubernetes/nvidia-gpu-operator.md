---
status: Active
maintainer: pacoxu
last_updated: 2025-10-31
tags: kubernetes, nvidia, gpu, dra, device-plugin, gpu-operator
canonical_path: zh/docs/kubernetes/nvidia-gpu-operator.md
---

# NVIDIA GPU Operator

<a href="https://github.com/NVIDIA/gpu-operator">NVIDIA GPU Operator</a> 通过自动化 GPU 驱动、容器运行时、设备插件和监控工具的安装和生命周期管理，简化了 Kubernetes 集群中 NVIDIA GPU 资源的部署和管理。

## 概述

GPU Operator 利用 Kubernetes Operator 模式管理运行 GPU 加速工作负载所需的所有 NVIDIA 软件组件。它消除了在每个节点上手动安装和配置 GPU 驱动和 CUDA 库的需要。

### 主要特性

- 自动 GPU 驱动安装和升级
- 容器运行时配置（nvidia-container-toolkit）
- 用于 GPU 资源管理的设备插件部署
- GPU 监控和遥测（DCGM Exporter）
- 支持多种 GPU 架构和配置
- 动态资源分配（DRA）集成（即将推出）

## 核心组件

### 1. 设备插件（当前默认）

<a href="https://github.com/NVIDIA/k8s-device-plugin">**NVIDIA 设备插件**</a>通过 Kubernetes 的传统设备插件框架实现 GPU 资源管理。

**主要特性：**

- 向 kubelet 通告 GPU
- 通过时间切片支持 GPU 共享
- MIG（多实例 GPU）支持
- GPU 资源分配和跟踪

**使用：**

```yaml
resources:
  limits:
    nvidia.com/gpu: 1
```

### 2. GPU DRA 驱动（未来方向）

<a href="https://github.com/NVIDIA/k8s-dra-driver-gpu">**NVIDIA GPU DRA 驱动**</a>使用动态资源分配提供下一代 GPU 资源管理，支持更灵活和拓扑感知的调度。

**架构：**

DRA 驱动包含两个 kubelet 插件：

- <a href="https://github.com/NVIDIA/k8s-dra-driver-gpu/tree/main/cmd/gpu-kubelet-plugin">**gpu-kubelet-plugin**</a>：标准 GPU 资源分配插件
- <a href="https://github.com/NVIDIA/k8s-dra-driver-gpu/tree/main/cmd/compute-domain-kubelet-plugin">**compute-domain-kubelet-plugin**</a>：用于计算域管理的高级插件

**高级能力：**

- GB200 支持，带有专门的计算域管理
- 拓扑感知 GPU 调度（NVLink、NVSwitch 感知）
- 细粒度资源分配
- 支持复杂的多 GPU 配置

**GB200 支持：**

DRA 驱动通过专门的计算域管理为 NVIDIA GB200 Grace Blackwell Superchip 平台提供增强支持。有关技术细节，请参见 <a href="https://docs.google.com/presentation/d/1Xupr8IZVAjs5bNFKJnYaK0LE7QWETnJjkz6KOfLu87E/edit?pli=1&slide=id.g373e0ebfa8e_1_233#slide=id.g373e0ebfa8e_1_233">GB200 DRA 架构介绍</a>。

**集成路线图：**

DRA 驱动将在未来版本中集成到 GPU Operator，提供从设备插件到基于 DRA 的资源管理的迁移路径。

**另见：** [动态资源分配（DRA）](./dra.md)了解 DRA 概念和使用模式的全面介绍。

### 3. 驱动管理器

<a href="https://github.com/NVIDIA/k8s-driver-manager">**驱动管理器**</a>自动化 Kubernetes 节点上的 NVIDIA GPU 驱动安装和生命周期管理。

**特性：**

- 自动驱动版本检测和安装
- 节点特定的驱动配置
- 驱动升级协调
- 支持跨节点的多个驱动版本

### 4. DCGM Exporter

<a href="https://github.com/NVIDIA/dcgm-exporter">**DCGM Exporter**</a> 通过将 NVIDIA Data Center GPU Manager（DCGM）的指标暴露给 Prometheus，提供 GPU 遥测和监控。

**提供的指标：**

- GPU 利用率和温度
- 内存使用和错误
- 功耗
- PCIe 吞吐量
- 时钟频率
- ECC 错误和健康状态

## 其他组件

GPU Operator 管理用于专门用例的多个其他组件：

### 运行时和发现

- **NVIDIA GPU Feature Discovery**：自动发现并标记节点上的 GPU 能力
- **GPU 驱动**：NVIDIA 数据中心 GPU 驱动
- **容器工具包**：nvidia-container-toolkit 用于容器运行时集成

### 高级特性

- **MIG Manager**：多实例 GPU 配置和管理
- **Validator**：部署后验证 GPU 栈
- **DCGM**：用于健康监控的 NVIDIA Data Center GPU Manager
- **KubeVirt GPU Device Plugin**：虚拟化工作负载的 GPU 支持
- **vGPU Manager**：虚拟化平台的虚拟 GPU 管理

### 专门配置

- **GDS 驱动**：用于直接 GPU 到存储传输的 GPUDirect Storage 驱动
- **机密计算管理器**：GPU 工作负载的 TEE（可信执行环境）支持
- **Kata Manager**：Kata Containers 运行时的 GPU 支持
- **GDRCopy 驱动**：用于高速网络传输的 GPUDirect RDMA 支持

## 部署

### 前提条件

- Kubernetes 集群（v1.24+）
- 具有 NVIDIA GPU 的 GPU 节点
- Node Feature Discovery（NFD）Operator（推荐）

### 安装

通过 Helm 安装：

```bash
# 添加 NVIDIA Helm 仓库
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm repo update

# 安装 GPU Operator
helm install gpu-operator nvidia/gpu-operator \
  --namespace gpu-operator \
  --create-namespace
```

### 验证

```bash
# 检查 GPU Operator Pod
kubectl get pods -n gpu-operator

# 验证 GPU 资源
kubectl get nodes -o=custom-columns=NAME:.metadata.name,GPUs:.status.capacity."nvidia\.com/gpu"

# 测试 GPU 工作负载
kubectl run gpu-test --image=nvidia/cuda:12.0.0-base-ubuntu20.04 \
  --limits=nvidia.com/gpu=1 -- nvidia-smi
```

## 配置

### 设备插件配置

配置 GPU 共享和资源策略：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: device-plugin-config
  namespace: gpu-operator
data:
  config.yaml: |
    version: v1
    sharing:
      timeSlicing:
        replicas: 4
    mig:
      strategy: mixed
```

### DCGM Exporter 配置

自定义指标采集：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: dcgm-exporter-config
  namespace: gpu-operator
data:
  metrics.csv: |
    # GPU utilization
    DCGM_FI_DEV_GPU_UTIL, gauge, GPU utilization
    # Memory usage
    DCGM_FI_DEV_FB_USED, gauge, GPU memory used
    DCGM_FI_DEV_FB_FREE, gauge, GPU memory free
```

## 最佳实践

### 生产部署

- 使用 Node Feature Discovery（NFD）进行自动节点标记
- 配置 GPU 命名空间的资源配额和限制
- 启用 DCGM 指标采集用于监控
- 设置 GPU 错误和热事件的告警
- 谨慎使用 GPU 时间切片（影响性能）

### 升级策略

- 在 staging 环境中测试 GPU Operator 升级
- 升级前审查驱动兼容性矩阵
- 规划驱动升级期间的节点维护窗口
- 升级期间和之后监控工作负载

### 故障排除

常见问题和解决方案：

```bash
# 检查 GPU Operator 日志
kubectl logs -n gpu-operator -l app=nvidia-device-plugin-daemonset

# 验证驱动安装
kubectl exec -n gpu-operator <driver-pod> -- nvidia-smi

# 检查设备插件注册
kubectl get nodes -o json | jq '.items[].status.allocatable'

# 验证 DCGM 指标
kubectl port-forward -n gpu-operator <dcgm-pod> 9400:9400
curl http://localhost:9400/metrics
```

## 迁移路径：设备插件到 DRA

NVIDIA 正在积极开发 DRA 驱动作为下一代 GPU 资源管理解决方案。迁移路径包括：

### 当前状态（设备插件）

- 稳定且生产就绪
- 简单资源模型（`nvidia.com/gpu`）
- 有限的拓扑感知能力
- 适用于基本 GPU 分配

### 未来状态（DRA 驱动）

- 增强的拓扑感知能力
- 细粒度资源分配
- 支持复杂的多设备工作负载
- 与高级 GPU 特性更好的集成（GB200、NVLink）

### 过渡策略

1. **评估阶段**：在开发集群中测试 DRA 驱动
2. **共存阶段**：同时运行设备插件和 DRA 驱动
3. **迁移阶段**：逐步将工作负载迁移到 DRA
4. **完成阶段**：停止使用设备插件

GPU Operator 将在过渡期间同时支持两种方式，允许运维人员根据需求选择。

## 学习资源

### 官方文档

- <a href="https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/index.html">NVIDIA GPU Operator 文档</a>
- <a href="https://github.com/NVIDIA/gpu-operator">GPU Operator GitHub 仓库</a>
- <a href="https://github.com/NVIDIA/k8s-device-plugin">NVIDIA 设备插件文档</a>
- <a href="https://github.com/NVIDIA/k8s-dra-driver-gpu">NVIDIA DRA 驱动文档</a>

### 会议演讲

- **KubeCon NA 2024**：<a href="https://www.youtube.com/watch?v=Z_15EyXOnhU">Kubernetes WG Device Management - GPUs, TPUs, NICs and More With DRA</a> - Kevin Klues & Patrick Ohly
- **NVIDIA GTC 会议**：搜索"GPU Operator"和"Kubernetes"主题

### 社区资源

- <a href="https://github.com/kubernetes/community/blob/master/wg-device-management/README.md">Kubernetes WG Device Management</a>
- <a href="https://github.com/NVIDIA/gpu-operator/issues">GPU Operator Issue 跟踪器</a>
- NVIDIA 开发者论坛：云原生板块

## 相关主题

- **[动态资源分配（DRA）](./dra.md)**：了解 DRA 概念和架构
- **[调度优化](./scheduling-optimization.md)**：GPU 调度策略和最佳实践
- **[节点资源接口（NRI）](./nri.md)**：细粒度容器资源管理
- **[工作负载隔离](./isolation.md)**：GPU checkpoint/restore 用于容错

## 路线图

正在进行的开发和提案：

- DRA 驱动集成到 GPU Operator（进行中）
- 增强的 GB200 支持，带有计算域管理
- 多 GPU 拓扑优化
- 改进的 GPU 健康监控和诊断
- 跨节点 GPU 结构管理（NVLink、NVSwitch）
- TEE（可信执行环境）用于机密 GPU 计算

最新更新请关注：

- <a href="https://github.com/NVIDIA/gpu-operator/issues">GPU Operator Issues</a>
- <a href="https://github.com/NVIDIA/k8s-dra-driver-gpu/issues">DRA Driver Issues</a>
- <a href="https://github.com/kubernetes/enhancements/issues/?q=is%3Aissue%20DRA%20in%3Atitle">Kubernetes DRA KEPs</a>

---

**注意**：部分关于即将推出的功能（特别是 DRA 集成）的信息反映了计划中的功能。请查看官方文档了解当前实现状态。
