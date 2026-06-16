---
status: Active
maintainer: pacoxu
last_updated: 2025-11-04
tags: kubernetes, dra, performance-testing, scalability, benchmarking
canonical_path: zh/docs/kubernetes/dra-performance-testing.md
---

# DRA 性能测试和可伸缩性

性能测试和可伸缩性验证对于动态资源分配（DRA）至关重要，以确保其能在大规模生产工作负载下运行。本指南涵盖了用于验证 DRA 性能以达到 GA 版本的测试策略、指标和基础设施。

## 概述

DRA 性能测试专注于验证动态资源分配框架能否高效处理大规模工作负载，而不会在 Kubernetes 控制平面中引入不可接受的调度延迟或资源消耗开销。

**关键测试目标：**

- 验证 DRA 可以扩展到数千节点
- 确保使用 DRA 时 Pod 启动延迟满足 SLO
- 测量 DRA 工作负载下的控制平面资源消耗
- 大规模测试结构化参数性能
- 基准测试真实工作负载场景

## 性能指标

### Pod 启动延迟

DRA 的主要性能指标是 **Pod 启动延迟**，根据 [Kubernetes SIG Scalability SLO](https://github.com/kubernetes/community/blob/master/sig-scalability/slos/pod_startup_latency.md) 进行测量。

**Pod 启动 SLO：**

- 与传统设备插件相比，使用 DRA 时 **P99 Pod 启动延迟**应保持在可接受范围内
- 从 Pod 创建到 Pod 运行状态的测量
- 包括调度器延迟、资源分配和容器启动

**指标采集：**

```yaml
# DRA 调度性能的 Prometheus 指标
apiserver_request_duration_seconds{verb="create",resource="resourceclaims"}
scheduler_framework_extension_point_duration_seconds{extension_point="PreFilter",plugin="DynamicResources"}
scheduler_pending_pods{queue="activeQ"}
```

### 控制平面资源消耗

运行 DRA 工作负载时监控控制平面组件的资源使用：

- **API Server**：CPU、内存、请求延迟
- **Scheduler**：CPU、内存、调度吞吐量
- **Controller Manager**：ResourceClaim 协调的 CPU、内存
- **etcd**：存储大小、读/写延迟

**关键指标：**

```yaml
# API Server 指标
apiserver_request_duration_seconds
apiserver_request_total

# 调度器指标
scheduler_schedule_attempts_total
scheduler_scheduling_duration_seconds

# etcd 指标
etcd_disk_backend_commit_duration_seconds
etcd_mvcc_db_total_size_in_bytes
```

## 结构化参数的规模测试

DRA 结构化参数在 Kubernetes v1.34 中达到 GA。规模测试验证生产级集群的性能。

### 测试基础设施

**当前测试规模：**

- **100 节点集群**：常规 CI 测试
- **5000 节点集群**：性能基准测试（计划中）

**CI 基础设施：**

- 测试环境：[ci-kubernetes-e2e-gce-100-node-dra-with-workload](https://prow.k8s.io/?job=ci-kubernetes-e2e-gce-100-node-dra-with-workload)
- 测试用例：[kubernetes/perf-tests#3368](https://github.com/kubernetes/perf-tests/pull/3368)
- CI 配置：[kubernetes/test-infra#35699](https://github.com/kubernetes/test-infra/pull/35699)

### 测试用例

#### 测试用例 1：作业调度负载

**场景：** 使用 DRA 资源声明的批处理作业调度

- **720 个作业**，每个 **1 个 Pod**
- 每个 Pod 通过 DRA ResourceClaim 请求资源
- 测试作业工作负载的调度器吞吐量
- 验证 ResourceClaim 分配性能

**验证：**

- Pod 启动延迟 P99 小于阈值
- 无因 DRA 开销导致的调度失败
- ResourceClaim 分配在 SLO 内完成

#### 测试用例 2：滚动 Pod 创建

**场景：** 模拟生产部署的渐进式 Pod 创建

- 分批创建 **800 个额外 Pod**
- 每次 **80 个 Pod**（10 波）
- 测试持续调度性能
- 验证负载下的控制平面稳定性

**验证：**

- 所有波次的 Pod 启动延迟一致
- API Server 和调度器资源使用保持稳定
- 无资源声明分配积压

#### 测试用例 3：稳态性能

**场景：** 使用 DRA 资源的长时间运行工作负载

- 维持使用 DRA 分配资源的运行工作负载
- 监控稳态控制平面性能
- 测试 ResourceClaim 生命周期管理

**测试结果示例：**

- [PrometheusSchedulingMetrics - DRA 稳态](https://storage.googleapis.com/kubernetes-ci-logs/logs/ci-kubernetes-e2e-gce-100-node-dra-with-workload/1981178480175353856/artifacts/FastFillSchedulingMetrics_PrometheusSchedulingMetrics_dra-steady-state_2025-10-23T02:12:31Z.json)

### 基准目标

**目标性能（与设备插件相比）：**

- **调度延迟**：< 10% 开销
- **API Server 负载**：< 5% 请求率增加
- **调度器 CPU**：< 15% 消耗增加
- **内存使用**：每 1000 个 Pod 增加 < 100MB

## DRANet：网络带宽优化

[DRANet](https://github.com/google/dranet) 是 Google 的网络资源 DRA 实现，展示了显著的性能提升。

**关键结果：**

- **网络密集型工作负载带宽提升高达 60%**
- 动态分配网络接口给 Pod
- RDMA 和高速网络支持
- 拓扑感知网络资源分配

**使用场景：**

- 需要高带宽节点间通信的 AI/ML 训练工作负载
- 具有模型并行性的分布式推理
- 大规模数据处理流水线
- 使用 RDMA 网络的多 GPU 工作负载

**架构：**

- 实现 DRA 接口的网络设备资源驱动
- 与 CNI 集成用于 Pod 网络配置
- 网络局部性的拓扑感知调度
- 支持异构网络设备

## 测试最佳实践

### 测试环境设置

**集群配置：**

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
maxPods: 110
# 启用 DRA 特性门控（v1.34+ 中 GA）
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: scheduler-config
data:
  scheduler-config.yaml: |
    apiVersion: kubescheduler.config.k8s.io/v1
    kind: KubeSchedulerConfiguration
    # 配置 DRA 插件设置
```

**资源驱动设置：**

- 部署 DRA 资源驱动（GPU、网络等）
- 配置设备插件用于对比测试
- 设置监控和指标采集
- 准备测试工作负载定义

### 指标采集策略

**测试前基线：**

1. 捕获无 DRA 工作负载的基线指标
2. 测量使用设备插件的控制平面性能
3. 建立 Pod 启动延迟基线

**测试期间：**

1. 每 15 秒采集 Prometheus 指标
2. 记录调度器追踪用于详细分析
3. 监控控制平面组件日志
4. 跟踪资源声明生命周期事件

**测试后分析：**

1. 比较 DRA 与设备插件的性能
2. 分析 Pod 启动延迟分布
3. 识别性能瓶颈
4. 生成性能回归报告

### 负载测试工作流

```bash
# 1. 部署 DRA 资源驱动
kubectl apply -f dra-driver-deployment.yaml

# 2. 创建测试设备的 ResourceClass
kubectl apply -f resource-class.yaml

# 3. 运行规模测试工作负载
kubectl apply -f scale-test-jobs.yaml

# 4. 监控调度性能
kubectl get --raw /metrics | grep scheduler_

# 5. 收集测试结果
kubectl logs -n kube-system -l component=kube-scheduler
```

## KEP 和提案

### DRA 结构化参数（v1.34 中 GA）

**KEP：** [kubernetes/enhancements#4381](https://github.com/kubernetes/enhancements/issues/4381)

**重点领域：**

- 细粒度分配的结构化资源参数
- 大规模集群的性能优化
- API 稳定性和向后兼容性
- 与现有调度特性的集成

**规模测试要求：**

- 在 5000 节点上验证性能
- 测试使用 DRA 的 10,000+ Pod
- 测量控制平面开销
- 确保满足调度延迟 SLO

### DRA 性能跟踪

**Kubernetes Issue：** [kubernetes/kubernetes#131198](https://github.com/kubernetes/kubernetes/issues/131198)

**测试目标：**

- 结构化参数 GA 的规模测试
- 大规模集群的性能基准
- 控制平面资源消耗验证
- Pod 启动延迟验证

**演讲：**

- [DRA KEP 和性能概述](https://docs.google.com/presentation/d/1COH8dG8qZ9jEMXzQVdco74XmroZQTMYcUgwAES8FYWU/edit?slide=id.g376578994e9_0_0&resourcekey=0-RCm3FFt8pxiK-y-_YHUkOg#slide=id.g376578994e9_0_0)

## 相关测试工具

### Kubernetes 性能测试

- **kubernetes/perf-tests**：官方 Kubernetes 性能测试套件
- **ClusterLoader2**：Kubernetes 集群的负载测试工具
- **knavigator**：NVIDIA 的集群性能测试工具（参见[性能测试指南](../inference/performance-testing.md)）

### DRA 特定测试

- **DRA 驱动模拟器**：无需物理设备的测试模拟驱动
- **ResourceClaim 生成器**：创建测试工作负载的工具
- **调度器追踪**：详细的调度决策分析

### 监控和可观测性

参见[可观测性指南](../observability/README.md)了解 DRA 工作负载的全面监控策略，包括：

- DRA 组件的 Prometheus 指标
- 调度器性能的 Grafana 仪表板
- 资源分配的分布式追踪
- 设备驱动性能的自定义指标

## 比较：DRA vs 设备插件

### 性能考量

**DRA 优势：**

- 更灵活的资源分配
- 更好地支持复杂设备拓扑
- 细粒度控制的结构化参数
- 网络附加设备支持

**DRA 开销：**

- 额外的 API 对象（ResourceClaim、ResourceClass）
- 更复杂的调度逻辑
- 控制平面资源使用增加（最小）

**性能对比矩阵：**

| 指标 | 设备插件 | DRA | 开销 |
|---------------------------|----------------|------------|-----------|
| Pod 启动延迟（P99） | 2.5s | 2.7s | +8% |
| 调度器 CPU | 150m | 165m | +10% |
| API Server QPS | 500 | 520 | +4% |
| 每 1000 Pod 内存 | 500MB | 550MB | +10% |

**注意：** 值为近似值，因工作负载和集群配置而异。

## 生产最佳实践

### 渐进式推出

1. **试点测试**：从小集群和有限工作负载开始
2. **Canary 部署**：将 DRA 部署到部分生产集群
3. **性能监控**：持续跟踪指标与基线对比
4. **回滚计划**：保持回退到设备插件的能力

### 容量规划

- **控制平面大小**：为 DRA 开销增加 10-15% 容量
- **调度器资源**：确保 DRA 插件有足够的 CPU/内存
- **API Server 速率限制**：为增加的 ResourceClaim 操作进行调整
- **etcd 存储**：规划额外的对象（ResourceClaim 等）

### 监控和告警

```yaml
groups:
  - name: dra-performance
    rules:
      - alert: DRASchedulingLatencyHigh
        expr: |
          histogram_quantile(0.99,
            scheduler_framework_extension_point_duration_seconds_bucket{
              plugin="DynamicResources"
            }
          ) > 0.5
        for: 5m
        annotations:
          summary: "DRA 调度延迟过高"

      - alert: ResourceClaimAllocationFailing
        expr: |
          rate(resourceclaim_allocation_failures_total[5m]) > 0.1
        for: 5m
        annotations:
          summary: "检测到 ResourceClaim 分配失败"
```

## 学习主题

- 理解 DRA 架构和资源分配流程
- Kubernetes 调度器插件框架和扩展点
- Kubernetes 大规模性能测试方法论
- 解读 Pod 启动延迟指标和 SLO
- DRA 工作负载的控制平面容量规划
- 比较设备分配策略（DRA vs 设备插件）

## 参考

- [DRA 文档](./dra.md)：概述和驱动实现
- [Pod 启动速度](./pod-startup-speed.md)：通用 Pod 启动优化
- [调度优化](./scheduling-optimization.md)：调度器性能
- [性能测试指南](../inference/performance-testing.md)：测试工具和框架
- [SIG Scalability Pod Startup SLO](https://github.com/kubernetes/community/blob/master/sig-scalability/slos/pod_startup_latency.md)
- [Kubernetes WG Device Management](https://github.com/kubernetes/community/blob/master/wg-device-management/README.md)

## 其他资源

- [DRA 测试用例 PR](https://github.com/kubernetes/perf-tests/pull/3368)
- [DRA CI 配置](https://github.com/kubernetes/test-infra/pull/35699)
- [DRANet GitHub 仓库](https://github.com/google/dranet)
- [所有 DRA KEP](https://github.com/kubernetes/enhancements/issues/?q=is%3Aissue%20%20DRA%20in%3Atitle)

---

**注意：** DRA 正在快速发展，性能优化持续进行。请始终参考最新的 Kubernetes 发布说明和 KEP 以获取最新的性能特征和最佳实践。
