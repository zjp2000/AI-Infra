---
status: Active
maintainer: pacoxu
last_updated: 2025-10-29
tags: kubernetes, nri, containerd, resource-management
canonical_path: zh/docs/kubernetes/nri.md
---

# 节点资源接口（NRI）

[NRI](https://github.com/containerd/nri) 是 containerd 的节点资源接口，支持节点级别的细粒度、可插拔容器资源管理。

## 概述

NRI 为 containerd 提供了一个插件接口，允许外部组件：

- 在运行时修改容器资源分配
- 注入自定义设备配置
- 实现高级资源管理策略
- 启用细粒度的 GPU 共享和分区

NRI 在运行时层面补充 DRA，而 DRA 处理集群级别的调度决策。

**另见：**
[调度优化指南](./scheduling-optimization.md#26-拓扑感知调度)了解 NRI 与调度策略的集成。

## 学习资源

### 会议演讲

- **KCD Shanghai 2024**：[基于 NRI 实现精细化且可插拔的容器资源管理](https://www.bilibili.com/video/BV1pp421Q7o5)
  - 云原生基础设施和 OS 方向
- **KubeCon NA 2024**：[Improving GPU Utilization and Accelerating Model Training with Scheduling Framework and NRI](https://www.youtube.com/watch?v=Gc5M1y4Er8g&t=3s) - He Cao

### 相关项目

- [containerd/nri](https://github.com/containerd/nri)：官方 NRI 实现
- [Project HAMi](https://github.com/Project-HAMi/HAMi)：使用 NRI 的 GPU 共享（CNCF Sandbox）
