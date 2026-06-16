---
status: Active
maintainer: pacoxu
last_updated: 2026-04-01
tags: kubernetes, kubelet, swap, memory-management, cgroupv2, eviction
canonical_path: zh/docs/kubernetes/swap-memory-management.md
---

# Kubernetes Swap 内存管理

本页面是操作启用 swap 的 Kubernetes Linux 节点的实用指南。

页面中的状态与上游状态对齐，截止日期为 **2026-04-01**。

## 当前上游状态

- `KEP-2400`（[issue #2400](https://github.com/kubernetes/enhancements/issues/2400)）已完成。
  - stage：stable
  - KEP 状态：implemented
  - milestone：v1.34
- 后续项目仍在进行中：
  - [#5359 工作负载控制的 Swap](https://github.com/kubernetes/enhancements/issues/5359)
  - [#5424 Swap 感知调度](https://github.com/kubernetes/enhancements/issues/5424)
  - [#5433 Swap 感知驱逐](https://github.com/kubernetes/enhancements/issues/5433)

## 前提条件

- Linux 节点
- 用于工作负载 swap 支持路径的 cgroup v2
- 已在主机上配置 swap（文件或分区）

## Kubelet 配置

### 基线（允许 kubelet 启动，保持工作负载非 swap）

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
failSwapOn: false
memorySwap:
  swapBehavior: NoSwap
```

### 启用工作负载 swap

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
failSwapOn: false
memorySwap:
  swapBehavior: LimitedSwap
```

## 行为模型

- `NoSwap`（默认）：Kubernetes 管理工作负载不使用 swap。
- `LimitedSwap`：工作负载可以在 kubelet/CRI 规则下使用 swap。
- Swap 分配是自动的且与请求成比例（当前模型下）：

```text
containerSwapLimit = (containerMemoryRequest / nodeTotalMemory) * totalPodsSwapAvailable
```

- 当前实现有意保守，尚未达到完整的工作负载级策略控制。

## 可观测性

### Kubelet 端点和指标

- `/metrics/resource`
  - `node_swap_usage_bytes`
  - `container_swap_usage_bytes`
  - `container_swap_limit_bytes`
- `/stats/summary`

### 有用的命令

```bash
kubectl top nodes --show-swap
kubectl top pods -A --show-swap
kubectl get nodes -o go-template='{{range .items}}{{.metadata.name}}: {{if .status.nodeInfo.swap.capacity}}{{.status.nodeInfo.swap.capacity}}{{else}}<unknown>{{end}}{{"\n"}}{{end}}'
```

### 通过 NFD 标签发现具有 swap 的节点

```bash
kubectl get nodes -o jsonpath='{range .items[?(@.metadata.labels.feature\.node\.kubernetes\.io/memory-swap)]}{.metadata.name}{"\t"}{.metadata.labels.feature\.node\.kubernetes\.io/memory-swap}{"\n"}{end}'
```

## 内存压力调优指南

将内核回收和 kubelet 驱逐作为一个控制循环一起使用。

关键旋钮：

- `vm.swappiness`
- `vm.min_free_kbytes`
- `vm.watermark_scale_factor`
- kubelet 驱逐阈值（`memory.available`）

来自 SIG Node 深入分析资料的常见起点（生产部署前请进行基准测试）：

- `vm.swappiness = 60`
- `vm.min_free_kbytes` 约为物理内存的 2-3%
- `vm.watermark_scale_factor` 显著高于默认值，以扩大回收窗口

运维目标：

- 尽早开始回收/swap，避免硬 OOM 峰值
- 避免过早回收导致工作负载抖动和延迟崩溃
- 保持 kubelet 驱逐和内核回收阈值对齐

## 风险和护栏

- Swap 如果在活动工作集上被交换，可能会降低延迟。
- 调优不当可能导致 OOM 在 kubelet 驱逐完成前发生。
- 调度器尚未完全支持 swap 感知。
- 尽可能加密 swap。
- 优先使用隔离且快速的备份存储（SSD/NVMe）。
- 保护系统关键守护进程免受 swap 和 I/O 饥饿的影响。

## 相关阅读

- [Kubernetes swap 内存管理文档](https://kubernetes.io/docs/concepts/cluster-administration/swap-memory-management/)
- [Linux 节点 swap 行为](https://kubernetes.io/docs/reference/node/swap-behavior/)
- [KEP-2400 issue](https://github.com/kubernetes/enhancements/issues/2400)
- [KEP-2400 设计](https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/2400-node-swap)
- [Kubernetes 1.28 swap beta 博客](https://kubernetes.io/blog/2023/08/24/swap-linux-beta/)
- [Kubernetes 1.32 swap 改进博客](https://kubernetes.io/blog/2025/03/25/swap-linux-improvements/)
- [Linux swap 调优深入分析](https://kubernetes.io/blog/2025/08/19/tuning-linux-swap-for-kubernetes-a-deep-dive/)
