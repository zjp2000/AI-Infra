---
status: Active
maintainer: pacoxu
last_updated: 2025-12-17
tags: kubernetes, gpu, fault-detection, self-healing, observability, dcgm
canonical_path: zh/docs/kubernetes/gpu-fault-detection.md
---

# Kubernetes 中的 GPU 故障检测与自愈

在运行 AI/ML 工作负载的 Kubernetes 集群中检测、诊断和自动恢复 GPU 故障的全面指南。

## 目录

- [概述](#概述)
- [常见 GPU 故障类型](#常见-gpu-故障类型)
- [故障检测方法](#故障检测方法)
- [故障语义与 Kubernetes 集成](#故障语义与-kubernetes-集成)
- [渐进式修复策略](#渐进式修复策略)
- [故障感知调度](#故障感知调度)
- [作业级归因与计费](#作业级归因与计费)
- [生产实现](#生产实现)
- [参考](#参考)

## 概述

在运行 AI 工作负载的 Kubernetes 集群中，GPU 故障可能导致严重的停机时间、计算资源浪费和训练中断。有效的 GPU 故障检测和自愈需要：

1. **全面的监控**，覆盖 GPU 健康信号

2. **标准化的故障语义**，Kubernetes 能够理解

3. **渐进式修复**，最小化对运行工作负载的影响

4. **故障感知调度**，避免有问题的节点/设备

5. **归因**，了解哪些工作负载触发了问题

本指南综合了字节跳动、Microsoft AKS、NVIDIA 和 Kubernetes 社区的最佳实践。

## 常见 GPU 故障类型

根据生产经验，GPU 故障分为四大类：

### 1. 卡掉线故障

**症状：**

- GPU 突然从 `nvidia-smi` 输出中消失
- XID 79：GPU 已从总线上掉落
- 驱动报告 GPU "未找到"
- DCGM 健康检查显示 GPU 不可用

**常见原因：**

- PCIe 链路不稳定
- 电源供应问题
- 硬件故障
- 驱动崩溃

**检测信号：**

- 内核日志中的 XID 79 错误
- DCGM Exporter：`DCGM_FI_DEV_XID_ERRORS` 指标
- `nvidia-smi` 返回码非零
- `/dev/nvidia*` 中缺少 GPU 设备文件

### 2. 链路故障

**症状：**

- NVLink 带宽降级
- XID 74：NVLink 训练错误
- 多 GPU 通信性能下降
- 训练吞吐量无明显原因下降

**常见原因：**

- NVLink 物理连接问题
- Fabric Manager 配置错误
- GPU 拓扑变化
- 环境因素（温度、振动）

**检测信号：**

- 内核日志中的 XID 74
- DCGM Exporter：`DCGM_FI_DEV_NVLINK_BANDWIDTH_*` 低于阈值
- `nvidia-smi nvlink --status` 显示非活跃链路
- NVLink 错误计数器持续增长

### 3. 内存故障

**症状：**

- XID 92, 95：GPU 内存错误
- ECC 错误累积
- 训练损失发散或出现 NaN 值
- GPU 重新映射事件
- 应用程序崩溃并出现 CUDA 内存错误

**常见原因：**

- 硬件内存缺陷
- 过热
- 电压不稳定
- 宇宙射线导致的比特翻转（罕见但可能）

**检测信号：**

- XID 92, 95 错误
- DCGM Exporter：`DCGM_FI_DEV_ECC_*` 计数器
- `nvidia-smi --query-gpu=ecc.errors.corrected.volatile.total` 持续增加
- NVML 中的 GPU 重新映射事件

### 4. 驱动故障

**症状：**

- NVML API 调用失败或超时
- 容器运行时报告设备插件错误
- Pod 卡在 `ContainerCreating` 状态，设备分配失败
- `nvidia-smi` 挂起或返回错误

**常见原因：**

- 驱动版本不兼容
- 内核模块崩溃
- CUDA 库不匹配
- 设备插件重启

**检测信号：**

- 设备插件日志显示分配错误
- NVML API 返回码指示失败
- kubelet 日志中的容器运行时错误
- GPU Operator Pod 处于 `CrashLoopBackOff` 状态

## 故障检测方法

### 1. DCGM Exporter（推荐）

NVIDIA Data Center GPU Manager (DCGM) 提供全面的 GPU 遥测，是 Kubernetes 中 GPU 监控的行业标准。

**部署：**

```yaml
# 通过 NVIDIA GPU Operator 部署
apiVersion: v1
kind: Pod
metadata:
  name: dcgm-exporter
  namespace: gpu-operator
spec:
  containers:

  - name: dcgm-exporter
    image: nvcr.io/nvidia/k8s/dcgm-exporter:3.3.8-3.6.0-ubuntu22.04
    securityContext:
      privileged: true
    volumeMounts:

    - name: pod-gpu-resources
      mountPath: /var/lib/kubelet/pod-resources
```

**用于故障检测的关键指标：**

| 指标 | 描述 | 故障类型 |
| -------- | ------------- | ------------ |
| `DCGM_FI_DEV_XID_ERRORS` | XID 错误码 | 全部 |
| `DCGM_FI_DEV_GPU_TEMP` | GPU 温度 | 过热 |
| `DCGM_FI_DEV_POWER_VIOLATION` | 电源限流事件 | 电源 |
| `DCGM_FI_DEV_THERMAL_VIOLATION` | 热限流 | 散热 |
| `DCGM_FI_DEV_ECC_DBE_VOL_TOTAL` | 不可纠正的 ECC 错误 | 内存 |
| `DCGM_FI_DEV_NVLINK_BANDWIDTH_*` | NVLink 带宽 | 链路 |
| `DCGM_FI_DEV_NVLINK_*_REPLAY_ERROR_COUNT_*` | NVLink 错误 | 链路 |

**Prometheus 告警示例：**

```yaml
groups:

- name: gpu_health
  rules:

  - alert: GPUXidError
    expr: increase(DCGM_FI_DEV_XID_ERRORS[5m]) > 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "在 {{ $labels.gpu }} 上检测到 GPU XID 错误"
      description: "XID 错误表明严重的 GPU 硬件问题"

  - alert: NVLinkDegraded
    expr: |
      DCGM_FI_DEV_NVLINK_BANDWIDTH_TOTAL <
      (DCGM_FI_DEV_NVLINK_BANDWIDTH_TOTAL offset 1h) * 0.8
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "{{ $labels.gpu }} 上的 NVLink 带宽降级"
```

**参考：**

- [DCGM Exporter GitHub](https://github.com/NVIDIA/dcgm-exporter)
- [NVIDIA 数据中心监控博客](https://developer.nvidia.com/blog/making-gpu-clusters-more-efficient-with-nvidia-data-center-monitoring/)

### 2. Node Problem Detector (NPD)

Kubernetes Node Problem Detector 将监控扩展到包括自定义 GPU 健康检查，并与 Kubernetes 节点条件集成。

**自定义 GPU 插件示例：**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: node-problem-detector-config
  namespace: kube-system
data:
  gpu-health-check.sh: |
    #!/bin/bash
    # 检查内核日志中的 XID 错误
    if dmesg | grep -q "Xid.*79\ | Xid.*74\ | Xid.*92\ | Xid.*95"; then
      echo "GPU_XID_ERROR"
      exit 1
    fi

    # 检查 nvidia-smi 可用性
    if ! nvidia-smi > /dev/null 2>&1; then
      echo "GPU_UNAVAILABLE"
      exit 1
    fi

    # 检查 NVLink 状态
    if nvidia-smi nvlink --status | grep -q "Down"; then
      echo "NVLINK_DOWN"
      exit 1
    fi

    exit 0

  gpu-problem-detector.json: |
    {
      "plugin": "custom",
      "pluginConfig": {
        "invoke_interval": "30s",
        "timeout": "10s",
        "max_output_length": 256,
        "concurrency": 1
      },
      "source": "gpu-health-check",
      "conditions": [
        {
          "type": "GPUHealthy",
          "reason": "GPUIsHealthy",
          "message": "GPU 功能正常"
        }
      ],
      "rules": [
        {
          "type": "permanent",
          "condition": "GPUHealthy",
          "reason": "GPUXIDError",
          "pattern": "GPU_XID_ERROR",
          "message": "检测到 GPU XID 错误"
        },
        {
          "type": "permanent",
          "condition": "GPUHealthy",
          "reason": "GPUUnavailable",
          "pattern": "GPU_UNAVAILABLE",
          "message": "通过 nvidia-smi 无法访问 GPU"
        },
        {
          "type": "temporary",
          "condition": "GPUHealthy",
          "reason": "NVLinkDown",
          "pattern": "NVLINK_DOWN",
          "message": "NVLink 连接降级"
        }
      ]
    }
```

**参考：**

- [Node Problem Detector GitHub](https://github.com/kubernetes/node-problem-detector)
- [Azure AKS Node Problem Detector](https://learn.microsoft.com/en-us/azure/aks/node-problem-detector)

### 3. NVIDIA GPU 调试指南

NVIDIA 提供了用于系统故障诊断的官方调试流程：

**XID 错误解释：**

- **XID 79**：GPU 已从总线上掉落（卡掉线）
- **XID 74**：NVLink 训练/初始化错误
- **XID 92, 95**：不可纠正的内存错误
- **XID 13, 31, 43**：驱动或内核错误

**诊断流程：**

1. 从日志或 DCGM 识别 XID 错误码

2. 查阅 NVIDIA 文档了解特定 XID 含义

3. 验证驱动版本兼容性

4. 检查 GPU 温度和电源状态

5. 使用压力工具隔离测试 GPU

6. 如果持续存在，考虑硬件更换

**参考：**

- [NVIDIA GPU 调试指南](https://docs.nvidia.com/deploy/gpu-debug-guidelines/index.html)
- [常见 GPU 错误讨论](https://www.cnblogs.com/jisongxie/p/10405302.html)

### 4. Kubernetes 原生可观测性

**GPU 遥测集成：**

- [将 GPU 遥测集成到 Kubernetes](https://cloud-atlas.readthedocs.io/zh-cn/latest/kubernetes/gpu/intergrate_gpu_telemetry_into_k8s.html)
- [DCGM Exporter 源码分析](https://www.jianshu.com/p/f38e58864496)

**DRA 驱动集成：**

- NVIDIA DRA 驱动与 DCGM 指标相关问题：[k8s-dra-driver-gpu #166](https://github.com/NVIDIA/k8s-dra-driver-gpu/issues/166)

## 故障语义与 Kubernetes 集成

为了实现自动化修复，GPU 故障必须转换为 Kubernetes 原生语义，以便调度器、控制器和 Operator 可以执行相应操作。

### 三层语义模型

#### 1. NodeCondition（节点级）

当整个节点 GPU 子系统受损且应避免在其上放置新工作负载时使用。

**标准条件（与 Azure AKS 对齐）：**

```yaml
apiVersion: v1
kind: Node
metadata:
  name: gpu-node-01
status:
  conditions:

  - type: GPUHealthy
    status: "False"
    reason: GPUXIDError
    message: "GPU 0 上检测到 XID 79"
    lastTransitionTime: "2025-12-17T10:00:00Z"

  - type: NVLinkHealthy
    status: "False"
    reason: NVLinkStatusInactive
    message: "NVLink 0-1 连接断开"
    lastTransitionTime: "2025-12-17T10:05:00Z"
```

**条件类型：**

- `GPUHealthy`：GPU 子系统整体健康
- `NVLinkHealthy`：NVLink 结构状态
- `GPUDriverHealthy`：NVIDIA 驱动和 NVML 可用性
- `GPUMemoryHealthy`：ECC 错误状态在阈值内

**触发的操作：**

- 调度器过滤：阻止新的 Pod 放置
- 应用污点：`gpu.health/unhealthy=true:NoSchedule`
- 向运维团队生成告警
- 触发自动化修复控制器

#### 2. DeviceCondition（GPU 设备级）

用于隔离特定故障 GPU，同时保持同节点上健康 GPU 可供调度。

**建议的 CRD（用于细粒度设备追踪）：**

```yaml
apiVersion: gpu.nvidia.com/v1
kind: GPUDevice
metadata:
  name: gpu-node-01-gpu-0
  labels:
    node: gpu-node-01
    gpu.index: "0"
spec:
  uuid: GPU-12345678-1234-1234-1234-123456789abc
  model: NVIDIA A100-SXM4-80GB
status:
  health:
    status: Unhealthy
    reason: XIDError
    severity: Critical
    confidence: 0.95
    lastCheck: "2025-12-17T10:00:00Z"
  faults:

  - type: CardDropout
    xid: 79
    timestamp: "2025-12-17T10:00:00Z"
    count: 1
  allocatable: false
```

**优势：**

- 多 GPU 节点可以继续使用健康 GPU 提供服务
- 减少硬件故障的"爆炸半径"
- 支持维护期间的节点部分容量
- 支持 MIG（多实例 GPU）场景

#### 3. WorkloadCondition（任务/工作负载级）

将硬件故障映射到训练/推理任务语义，实现应用感知的恢复。

**训练任务集成：**

```yaml
apiVersion: batch.volcano.sh/v1alpha1
kind: Job
metadata:
  name: training-job-01
status:
  conditions:

  - type: GPUFaultDetected
    status: "True"
    reason: XIDErrorOnPod
    message: "Pod training-job-01-worker-3 上的 GPU 故障"
    lastTransitionTime: "2025-12-17T10:00:00Z"
  state:
    phase: Requeued
    lastTransitionTime: "2025-12-17T10:01:00Z"
    reason: GPUFaultRemediation
```

**与训练框架的集成：**

- **Volcano**：支持 GPU 故障时任务重新入队
- **Kubeflow Training Operator**：自动替换工作节点
- **PyTorch Elastic**：利用弹性训练实现韧性
- **检查点保存**：与检查点/恢复系统协调

### 故障知识图谱

将低层信号系统性映射到可操作洞察：

```text
信号层：

  - XID 错误（内核日志）
  - DCGM 指标（Prometheus）
  - NVML 健康检查
  - NVLink 状态
  - PCIe AER 日志
  - Fabric Manager 事件

↓ 信号聚合与关联 ↓

故障属性：

  - type: CardDropout | LinkFailure | MemoryError | DriverFailure
  - scope: device | node | fabric-domain | rack
  - severity: S0（严重）| S1（高）| S2（中）| S3（低）
  - confidence: 0.0-1.0（多信号融合）
  - duration: transient | persistent
  - impact: workload-disrupting | performance-degrading

↓ 修复决策 ↓

操作提示：

  - cordon: 阻止新调度
  - isolate-device: 从可分配池中移除设备
  - reset-driver: 重启 nvidia-fabricmanager / device-plugin
  - reset-gpu: nvidia-smi -r（高风险）
  - drain: 驱逐 Pod 并重启节点
  - replace: 需要硬件更换
```

**多信号关联示例：**

```text
IF (XID_79_count > 0 AND nvidia_smi_returns_error) THEN
  fault.type = CardDropout
  fault.severity = S0
  fault.confidence = 0.99
  action.hints = [cordon, drain, replace]

IF (nvlink_bandwidth < 0.8 * baseline AND XID_74_count > 3) THEN
  fault.type = LinkFailure
  fault.severity = S1
  fault.confidence = 0.85
  action.hints = [isolate-device, reset-fabricmanager]

IF (ecc_uncorrectable_errors > threshold) THEN
  fault.type = MemoryError
  fault.severity = S0
  fault.confidence = 0.90
  action.hints = [cordon, isolate-device, replace]
```

## 渐进式修复策略

核心原则：**在确保集群健康的同时，最小化对运行工作负载的干扰**。从低影响到高影响逐步推进，并在每一步进行验证。

### 修复操作层级

#### 级别 0：检测与告警（无操作）

- **操作**：记录事件、发出 Prometheus 告警、更新 NodeCondition
- **影响**：对工作负载无影响
- **用例**：监控、早期预警
- **回滚**：不适用

#### 级别 1：调度预防（Cordon/Taint）

- **操作**：
  - 应用污点：`kubectl taint node gpu-node-01 gpu.health/suspect=true:NoSchedule`
  - 将节点标记为不可调度
  - 发出 Kubernetes Event
- **影响**：不调度新 Pod，现有 Pod 继续运行
- **用例**：疑似但未确认的问题、瞬时故障
- **验证**：监控 5-10 分钟，检查问题是否持续
- **回滚**：移除污点，标记为可调度

#### 级别 2：软恢复（服务重启）

- **操作**：
  - 重启设备插件：`kubectl delete pod -n gpu-operator nvidia-device-plugin-xxx`
  - 重启 fabric manager：`systemctl restart nvidia-fabricmanager`
  - 重启 DCGM：`systemctl restart nvidia-dcgm`
- **影响**：短暂的 GPU 分配暂停（30-60s），运行中的容器不受影响
- **用例**：驱动/插件问题、fabric manager 错误
- **验证**：检查 nvidia-smi，验证设备插件就绪状态
- **回滚**：不适用（服务重启是幂等的）

#### 级别 3：设备隔离

- **操作**：
  - 更新设备插件配置以排除故障 GPU
  - 减少节点的可分配 GPU 数量
  - 将 GPUDevice CR 标记为 `allocatable: false`
- **影响**：节点容量减少，健康 GPU 上的运行工作负载继续
- **用例**：多 GPU 节点上的单 GPU 故障
- **验证**：确认健康 GPU 仍可分配
- **回滚**：在插件配置中重新启用设备

#### 级别 4：硬恢复（GPU 重置）

- **操作**：
  - `nvidia-smi -r -i <gpu_id>`（需要特殊权限）
  - PCI 设备重新扫描：`echo 1 > /sys/bus/pci/rescan`
- **影响**：所有使用受影响 GPU 的容器被杀死
- **用例**：持续性 XID 错误、GPU 挂起
- **风险**：高——可能影响多个工作负载
- **验证**：nvidia-smi 显示 GPU 健康，运行 GPU 压力测试
- **回滚**：如果重置失败，执行 drain 和重启

#### 级别 5：节点 Drain 与重启

- **操作**：
  - 遵守 PodDisruptionBudgets
  - `kubectl drain gpu-node-01 --ignore-daemonsets --delete-emptydir-data`
  - 重启节点
  - 等待节点就绪
- **影响**：节点上所有 Pod 被驱逐并重新调度
- **用例**：驱动故障、多 GPU 故障、持续性问题
- **验证**：节点恢复正常，所有 GPU 检查通过
- **回滚**：需要手动干预

#### 级别 6：节点更换

- **操作**：
  - 标记节点退役
  - Drain 节点
  - 触发 Cluster Autoscaler 或 MCO 进行更换
- **影响**：节点从集群中移除
- **用例**：需要 RMA 的硬件故障
- **验证**：新节点健康并加入集群
- **回滚**：不适用（需要硬件维修）

### 修复策略引擎

**关键要求：**

1. **迟滞与冷却**

   - 不要对单个瞬时信号做出反应
   - 要求故障持续超过持续时间阈值（例如 5 分钟）
   - 在修复尝试之间设置冷却期（例如 30 分钟）
   - 跟踪修复历史以发现重复问题

2. **爆炸半径控制**

   - 限制每个节点池的并发修复数量（例如最大 10%）
   - 限制每个机架/可用区的并发修复数量
   - 尊重 fabric 域（不要干扰整个 NVSwitch 域）
   - 对破坏性操作实施速率限制

3. **工作负载感知**

   - 对训练（长时运行）和推理（无状态）采用不同策略
   - 尊重任务优先级和抢占策略
   - 在驱逐前与检查点系统协调
   - 严格遵循 PodDisruptionBudgets

4. **验证与回滚**

   - 操作后验证是认为修复成功的必要条件
   - 使用相同的检测信号来验证修复
   - 如果验证失败，自动回滚
   - 对重复失败使用指数退避

**策略示例（伪代码）：**

```python
def remediate_gpu_fault(node, fault):
    # 检查爆炸半径预算
    if current_remediations_in_rack(node.rack) > MAX_RACK_REMEDIATIONS:
        return DEFER

    # 检查故障置信度和持续时间
    if fault.confidence < 0.7 or fault.duration < 5min:
        return MONITOR  # 级别 0

    # 检查近期修复历史
    if node.last_remediation_time < 30min_ago:
        return DEFER  # 冷却期

    # 渐进式操作选择
    if fault.type == "CardDropout" and fault.severity == "S0":
        # 严重故障，快速升级
        if not node.is_cordoned:
            action = CORDON  # 级别 1
        elif not node.drain_attempted:
            action = DRAIN_AND_REBOOT  # 级别 5
        else:
            action = REPLACE  # 级别 6

    elif fault.type == "LinkFailure" and fault.severity == "S1":
        # 先尝试软恢复
        if not node.fabricmanager_restarted:
            action = RESTART_FABRICMANAGER  # 级别 2
        elif node.healthy_gpus > 0:
            action = ISOLATE_DEVICE  # 级别 3
        else:
            action = DRAIN_AND_REBOOT  # 级别 5

    elif fault.type == "DriverFailure":
        if not node.device_plugin_restarted:
            action = RESTART_DEVICE_PLUGIN  # 级别 2
        else:
            action = DRAIN_AND_REBOOT  # 级别 5

    # 执行操作并验证
    execute_action(action, node)

    # 安排操作后验证
    schedule_verification(node, action, delay=2min)
```

### 与现有控制器的集成

**Draino（Drain 自动化）：**

```yaml
# Draino 监控 NodeConditions 并自动执行 drain
apiVersion: v1
kind: ConfigMap
metadata:
  name: draino-config
data:
  config.yaml: |
    conditions:

    - type: GPUHealthy
      status: "False"
      min-duration: 5m

    - type: NVLinkHealthy
      status: "False"
      min-duration: 10m
    drain-buffer: 10m
    eviction-headroom: 5
    max-concurrent-drains: 3
```

**Node Readiness Controller：**

```yaml
# kubernetes-sigs/node-readiness-controller
apiVersion: v1
kind: ConfigMap
metadata:
  name: node-readiness-config
data:
  config.yaml: |
    maintenance_triggers:

    - node_condition:
        type: GPUHealthy
        status: "False"
      action: cordon_and_drain
      grace_period: 5m
```

**Cluster Autoscaler：**

- 为自动缩放器注释不健康节点以进行更换
- 修复期间设置 `cluster-autoscaler.kubernetes.io/scale-down-disabled: "true"`

- 与自动缩放器协调以防止过早缩容

## 故障感知调度

扩展 Kubernetes 调度，将 GPU 健康作为一等维度考虑，防止工作负载放置在有风险的硬件上。

### 调度器插件集成

**GPU 健康评分调度器插件：**

```go
// 概念实现
type GPUHealthScore struct {
    NodeScore float64  // 0.0（不健康）到 1.0（健康）
    Factors   []HealthFactor
}

type HealthFactor struct {
    Name   string
    Weight float64
    Value  float64
}

func (p *GPUHealthPlugin) Score(ctx context.Context, state *framework.CycleState,
    pod *v1.Pod, nodeName string) (int64, *framework.Status) {

    node := p.getNode(nodeName)

    // 从多个信号计算健康评分
    score := 1.0

    // 因素 1：近期 XID 错误（权重：0.4）
    xidErrors := p.getXIDErrorCount(node, last5min)
    score *= (1.0 - 0.4 * min(xidErrors / 10.0, 1.0))

    // 因素 2：NVLink 健康（权重：0.3）
    nvlinkHealth := p.getNVLinkBandwidth(node) / p.getBaselineNVLinkBandwidth(node)
    score *= (0.7 + 0.3 * nvlinkHealth)

    // 因素 3：ECC 错误（权重：0.2）
    eccErrors := p.getECCUncorrectableErrors(node, last1hour)
    score *= (1.0 - 0.2 * min(eccErrors / 100.0, 1.0))

    // 因素 4：温度（权重：0.1）
    temp := p.getGPUTemperature(node)
    if temp > 85 {
        score *= (1.0 - 0.1 * (temp - 85) / 15.0)
    }

    // 转换为调度器评分（0-100）
    return int64(score * 100), nil
}
```

**最小健康阈值过滤插件：**

```go
func (p *GPUHealthPlugin) Filter(ctx context.Context, state *framework.CycleState,
    pod *v1.Pod, nodeInfo *framework.NodeInfo) *framework.Status {

    // 检查 Pod 是否需要高 GPU 健康评分
    minHealthAnnotation := pod.Annotations["gpu.health/min-score"]
    if minHealthAnnotation == "" {
        return nil  // 无要求
    }

    minHealth, _ := strconv.ParseFloat(minHealthAnnotation, 64)
    nodeHealth := p.getNodeHealthScore(nodeInfo.Node())

    if nodeHealth < minHealth {
        return framework.NewStatus(framework.Unschedulable,
            fmt.Sprintf("节点 GPU 健康评分 %.2f 低于要求的 %.2f",
                nodeHealth, minHealth))
    }

    return nil
}
```

### Kueue/Volcano 集成

**带健康要求的 Kueue 队列：**

```yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: ClusterQueue
metadata:
  name: training-high-priority
spec:
  cohort: default
  resourceGroups:

  - coveredResources: ["gpu"]
    flavors:

    - name: a100-healthy
      resources:

      - name: gpu
        nominalQuota: 32
      # 健康节点的自定义标签选择器
      nodeSelector:
        gpu.health/score: "0.9"  # 仅使用健康评分 >0.9 的节点

    - name: a100-standard
      resources:

      - name: gpu
        nominalQuota: 64
      nodeSelector:
        gpu.health/score: "0.7"  # 标准工作负载的较低要求
```

**Volcano Gang 调度与健康集成：**

```yaml
apiVersion: batch.volcano.sh/v1alpha1
kind: Job
metadata:
  name: distributed-training
spec:
  minAvailable: 4
  schedulerName: volcano
  plugins:
    healthaware: []  # 自定义插件
  tasks:

  - replicas: 4
    template:
      metadata:
        annotations:
          gpu.health/min-score: "0.8"  # 要求 0.8+ 健康评分
      spec:
        containers:

        - name: worker
          resources:
            limits:
              nvidia.com/gpu: 8
```

### DRA 健康集成

使用动态资源分配（DRA），GPU 健康可以影响声明分配和设备选择：

```yaml
apiVersion: resource.k8s.io/v1alpha3
kind: ResourceClaim
metadata:
  name: training-gpus
spec:
  devices:
    requests:

    - name: gpus
      deviceClassName: gpu.nvidia.com
      count: 8
      selectors:

      - cel:
          expression: |
            device.health.score > 0.9 &&
            device.health.xidErrors == 0 &&
            device.health.nvlinkStatus == "active"
```

**DRA 驱动健康扩展：**

```go
// 扩展 NVIDIA DRA 驱动以暴露健康属性
type GPUDevice struct {
    UUID  string
    Model string
    Health struct {
        Score        float64
        XIDErrors    int64
        NVLinkStatus string
        ECCErrors    int64
        LastCheck    time.Time
    }
}

// 驱动在发现期间填充健康信息
func (d *driver) GetDevices() []*GPUDevice {
    devices := d.discoverGPUs()
    for _, device := range devices {
        device.Health = d.checkDeviceHealth(device.UUID)
    }
    return devices
}
```

## 作业级归因与计费

了解哪些工作负载触发了 GPU 故障对以下方面至关重要：

- 公平的资源核算和成本分摊
- 识别有问题的应用程序
- 容量规划和硬件更新决策
- SLA 执行

### DCGM HPC 任务映射

DCGM Exporter 支持使用 HPC 任务映射功能将 GPU 指标映射到特定任务/Pod。

**GPU Operator 配置（v25.10.1+）：**

```yaml
apiVersion: nvidia.com/v1
kind: ClusterPolicy
metadata:
  name: cluster-policy
spec:
  dcgmExporter:
    enabled: true
    hpcJobMapping:
      enabled: true
      directory: /var/lib/kubelet/pod-resources
      volumeMount:
        hostPath: /var/lib/kubelet/pod-resources
        mountPath: /var/lib/kubelet/pod-resources
```

**任务映射文件格式：**

```text
# 由 job-mapping 控制器生成
# 格式：<GPU UUID>,<任务 ID>,<任务名称>,<任务用户>,<任务开始时间>
GPU-12345678-1234-1234-1234-123456789abc,job-12345,training-job-01,team-ml,1702800000
GPU-87654321-4321-4321-4321-cba987654321,job-12346,inference-serve,team-ai,1702800060
```

**带有任务标签的增强 DCGM 指标：**

```text
# 无任务映射
DCGM_FI_DEV_GPU_UTIL{gpu="0",UUID="GPU-12345...",modelName="A100-SXM4-80GB"} 95.0

# 有任务映射
DCGM_FI_DEV_GPU_UTIL{
  gpu="0",
  UUID="GPU-12345...",
  modelName="A100-SXM4-80GB",
  job_id="job-12345",
  job_name="training-job-01",
  namespace="ml-team",
  pod="training-job-01-worker-0"
} 95.0
```

### Kubernetes 任务映射控制器

DCGM 的 HPC 任务映射是为 HPC 调度器（Slurm、PBS）设计的。对于 Kubernetes，我们需要一个控制器来生成映射文件：

**控制器职责：**

1. 监控 Pod 创建/删除事件

2. 查询 kubelet pod-resources API 获取 GPU 分配

3. 生成 DCGM 兼容的映射文件

4. 在 Pod GPU 分配变化时更新映射

5. Pod 终止时清理映射

**实现示例：**

```go
type JobMappingController struct {
    kubeclient kubernetes.Interface
    podResourcesClient podresourcesapi.PodResourcesListerClient
    mappingDir string
}

func (c *JobMappingController) reconcilePod(pod *v1.Pod) error {
    // 获取此 Pod 的 GPU 分配
    resp, err := c.podResourcesClient.List(context.TODO(), &podresourcesapi.ListPodResourcesRequest{})
    if err != nil {
        return err
    }

    var mappings []string
    for _, podRes := range resp.PodResources {
        if podRes.Namespace != pod.Namespace || podRes.Name != pod.Name {
            continue
        }

        for _, container := range podRes.Containers {
            for _, device := range container.Devices {
                if device.ResourceName == "nvidia.com/gpu" {
                    for _, deviceID := range device.DeviceIds {
                        uuid := c.getGPUUUID(deviceID)
                        jobID := string(pod.UID)
                        jobName := fmt.Sprintf("%s/%s", pod.Namespace, pod.Name)
                        jobUser := pod.Labels["workload.user"]
                        startTime := pod.CreationTimestamp.Unix()

                        mapping := fmt.Sprintf("%s,%s,%s,%s,%d",
                            uuid, jobID, jobName, jobUser, startTime)
                        mappings = append(mappings, mapping)
                    }
                }
            }
        }
    }

    // 写入映射文件
    mappingFile := filepath.Join(c.mappingDir, fmt.Sprintf("pod-%s.txt", pod.UID))
    return os.WriteFile(mappingFile, []byte(strings.Join(mappings, "\n")), 0644)
}
```

### 故障归因查询

通过任务感知的指标，我们可以将故障归因到特定工作负载：

**查询：哪个任务触发了 XID 错误？**

```promql
increase(DCGM_FI_DEV_XID_ERRORS{job_name!=""}[1h]) > 0
```

**查询：故障发生时的 GPU 利用率：**

```promql
DCGM_FI_DEV_GPU_UTIL{job_name="training-job-01"}
  and on(gpu, UUID)
  increase(DCGM_FI_DEV_XID_ERRORS[5m]) > 0
```

**查询：GPU 停机时间的成本归因：**

```promql
sum by (namespace, job_name) (
  rate(DCGM_FI_DEV_GPU_UTIL[1h]) *
  (1 - (increase(DCGM_FI_DEV_XID_ERRORS[1h]) > bool 0))
) * GPU_HOURLY_COST
```

### 成本分摊与 SLA 执行

**公平计费调整：**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: gpu-billing-policy
data:
  policy.yaml: |
    # 当 GPU 因硬件（而非用户导致）故障而不健康时，
    # 不向用户收取费用
    fault_types:
      hardware_failure:
        charge_adjustment: 0.0  # 不收费
        xids: [79, 74]  # 卡掉线、NVLink 故障

      user_oom:
        charge_adjustment: 1.0  # 全额收费
        xids: [13, 31]  # 内存不足、驱动错误

      memory_error:
        charge_adjustment: 0.5  # 共同责任
        xids: [92, 95]  # ECC 错误（可能是工作负载压力所致）
```

## 生产实现

### 参考架构

```text
┌─────────────────────────────────────────────────────────────────┐
│                        Kubernetes 集群                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    检测层                                   │  │
│  ├──────────────────────────────────────────────────────────┤  │
│  │  • DCGM Exporter（GPU 指标）                              │  │
│  │  • Node Problem Detector（系统检查）                      │  │
│  │  • 内核日志监控（XID 错误）                                  │  │
│  │  • NVML 健康检查                                           │  │
│  └──────────────────────────────────────────────────────────┘  │
│                            ↓                                     │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                 关联与语义                                  │  │
│  ├──────────────────────────────────────────────────────────┤  │
│  │  • Prometheus 规则（告警聚合）                              │  │
│  │  • 故障分类（XID→故障类型）                                  │  │
│  │  • 置信度评分（多信号融合）                                │  │
│  │  • NodeCondition/DeviceCondition 更新                     │  │
│  └──────────────────────────────────────────────────────────┘  │
│                            ↓                                     │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                 修复控制器                                  │  │
│  ├──────────────────────────────────────────────────────────┤  │
│  │  • 策略引擎（爆炸半径、工作负载感知）                         │  │
│  │  • 操作执行器（Cordon/Drain/Reset/Replace）               │  │
│  │  • 验证循环（操作后健康检查）                               │  │
│  │  • 集成：Draino、Node Readiness Controller                │  │
│  └──────────────────────────────────────────────────────────┘  │
│                            ↓                                     │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                 故障感知调度                                │  │
│  ├──────────────────────────────────────────────────────────┤  │
│  │  • 调度器插件（健康评分）                                   │  │
│  │  • Kueue/Volcano（队列级健康要求）                          │  │
│  │  • DRA 驱动（设备级健康属性）                              │  │
│  └──────────────────────────────────────────────────────────┘  │
│                            ↓                                     │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │               任务归因与计费                                │  │
│  ├──────────────────────────────────────────────────────────┤  │
│  │  • 任务映射控制器（Pod→GPU 映射）                           │  │
│  │  • DCGM HPC 任务映射（指标丰富化）                          │  │
│  │  • 计费调整（故障感知的成本分摊）                           │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### 部署检查清单

**阶段 1：检测（MVP）**
- [ ] 部署 NVIDIA GPU Operator 及 DCGM Exporter
- [ ] 配置 Prometheus 以抓取 DCGM 指标
- [ ] 创建关键 XID 错误的 Prometheus 告警
- [ ] 部署带有 GPU 健康检查的 Node Problem Detector
- [ ] 设置监控仪表板（Grafana）

**阶段 2：基本修复**
- [ ] 部署 Draino 以自动执行节点 drain
- [ ] 配置 Node Readiness Controller
- [ ] 为 GPU 重置/重启实现手动操作手册
- [ ] 设置事件响应流程
- [ ] 定义每机架/AZ 的爆炸半径限制

**阶段 3：高级修复**
- [ ] 实现设备级隔离（GPUDevice CRD）
- [ ] 部署带有渐进式策略的修复控制器
- [ ] 为所有修复操作添加验证循环
- [ ] 实现冷却和速率限制
- [ ] 添加工作负载感知策略（训练 vs 推理）

**阶段 4：故障感知调度**
- [ ] 实现 GPU 健康评分调度器插件
- [ ] 将健康要求集成到 Kueue/Volcano
- [ ] 使用健康属性扩展 DRA 驱动
- [ ] 为关键工作负载添加准入策略
- [ ] 实现基于健康的自动扩缩策略

**阶段 5：归因与优化**
- [ ] 部署任务映射控制器
- [ ] 启用 DCGM HPC 任务映射
- [ ] 实现故障归因查询
- [ ] 设置成本分摊调整策略
- [ ] 创建容量规划报告

### 最佳实践

**1. 保守起步**
- 仅从检测和告警开始（无自动化）
- 逐步启用自动化并设置严格速率限制
- 在启用修复前验证检测准确性
- 关键决策保持人工参与

**2. 度量和迭代**
- 跟踪故障检测的误报率
- 衡量 MTTR（平均修复时间）改进
- 监控修复操作的"附带损害"
- 收集工作负载所有者的反馈

**3. 安全与合规**
- 审计所有自动化修复操作
- 破坏性操作（GPU 重置、重启）需要审批
- 使用 RBAC 限制修复控制器权限
- 通过 Webhook/Slack 实施变更通知

**4. 集成点**
- 与 Cluster Autoscaler 协调进行节点更换
- 与事件管理系统集成（PagerDuty、OpsGenie）
- 向容量规划工具提供数据
- 与硬件维护计划同步

## 参考

### 行业实践

**字节跳动 Volcano：**

- [GPU 故障检测与自愈](https://mp.weixin.qq.com/s/vc4y2wnZlyou3cfqnQP2-w)
- GPU 健康监控和自动恢复的综合方法

**Microsoft Azure AKS：**

- [Node Problem Detector](https://learn.microsoft.com/en-us/azure/aks/node-problem-detector)
- 标准化的 GPU 条件类型：`GPUMissing`、`NVLinkStatusInactive`、`XIDErrors`、`IBLinkFlapping`

**NVIDIA 资源：**

- [DCGM Exporter](https://github.com/NVIDIA/dcgm-exporter)
- [GPU 调试指南](https://docs.nvidia.com/deploy/gpu-debug-guidelines/index.html)
- [GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/)
- [数据中心监控博客](https://developer.nvidia.com/blog/making-gpu-clusters-more-efficient-with-nvidia-data-center-monitoring/)
- [k8s-dra-driver-gpu](https://github.com/NVIDIA/k8s-dra-driver-gpu)

### 社区资源

**Kubernetes：**

- [Node Problem Detector](https://github.com/kubernetes/node-problem-detector)
- [Node Readiness Controller](https://github.com/kubernetes-sigs/node-readiness-controller)
- [动态资源分配 (DRA)](https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/)
- [WG Batch](https://github.com/kubernetes/community/blob/master/wg-batch/README.md)
- [WG Device Management](https://github.com/kubernetes/community/blob/master/wg-device-management/README.md)

**自动化：**

- [Draino](https://github.com/planetlabs/draino) - 自动节点 drain

**案例研究：**

- [GPU 故障排查故事](https://mp.weixin.qq.com/s/ehcX_43lmpaYaQS_x49BcA)
- [Kubernetes 上的 GPU 故障排查](https://nolebase.ayaka.io/zh-CN/%E7%AC%94%E8%AE%B0/%F0%9F%A7%B1%20%E5%9F%BA%E7%A1%80%E8%AE%BE%E6%96%BD/%F0%9F%9A%A2%20Kubernetes/%E5%9C%A8%20Kubernetes%20%E9%9B%86%E7%BE%A4%E4%B8%8A%E7%9A%84%20GPU%20%E6%8E%89%E5%8D%A1%E5%92%8C%E6%95%85%E9%9A%9C%E6%8E%92%E6%9F%A5.html)
- [GPU 消失故障排查](https://www.cnblogs.com/jisongxie/p/10405302.html)
- [Kubernetes GPU 遥测集成](https://cloud-atlas.readthedocs.io/zh-cn/latest/kubernetes/gpu/intergrate_gpu_telemetry_into_k8s.html)
- [DCGM Exporter 源码分析](https://www.jianshu.com/p/f38e58864496)

### 相关文档

- [NVIDIA GPU Operator](./nvidia-gpu-operator.md)
- [动态资源分配 (DRA)](./dra.md)
- [可观测性指南](../observability/README.md)
- [训练容错](../training/README.md)

---

**警告：** 本指南中的部分内容参考了 AI 生成的摘要和社区讨论。请始终在您的特定环境中验证方法，并查阅官方 NVIDIA 文档以获取硬件特定的指导。
