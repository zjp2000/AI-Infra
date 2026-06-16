---
status: Active
maintainer: pacoxu
last_updated: 2025-10-29
tags: inference, model-lifecycle, cold-start, offloading, optimization
canonical_path: zh/docs/inference/model-lifecycle.md
---

# 模型冷启动、休眠模式与生命周期管理

本文档涵盖 LLM 推理系统中的模型生命周期管理，重点关注冷启动优化、休眠模式、模型卸载和快速模型驱动。这些技术对于高效资源利用、成本降低和在生产推理系统中保持低延迟至关重要。

## 目录

- [概述](#overview)
- [模型冷启动优化](#model-cold-start-optimization)
- [休眠模式与唤醒](#sleep-mode-and-wake-up)
- [模型卸载](#model-offloading)
- [快速模型驱动](#fast-model-actuation)
- [KV 缓存持久化](#kv-cache-persistence)
- [生产实现](#production-implementations)
- [最佳实践](#best-practices)
- [参考资料](#references)

---

## 概述

模型生命周期管理解决了跨操作状态高效管理 LLM 资源的挑战：

- **冷启动**：从存储加载和初始化模型
- **活跃服务**：模型正在处理推理请求
- **休眠模式**：临时暂停非活跃模型以释放资源
- **唤醒**：在需要时快速重新激活休眠模型
- **卸载**：在存储层之间移动模型（GPU、CPU、磁盘）
- **关闭**：从内存中完全卸载模型

![LLM 模型生命周期](https://github.com/user-attachments/assets/f45727ca-526e-4e78-8afc-b1b085a24fc1)

*图 1：LLM 模型生命周期状态图——模型从冷启动到关闭经历各种状态，休眠模式和卸载作为中间优化状态。*

**关键挑战：**

- 模型加载可能需要数秒到数分钟，取决于模型大小
- GPU 内存有限且昂贵
- 多个模型竞争同一资源
- 用户期望即使对不常访问的模型也有低延迟
- 低效的生命周期管理导致资源浪费

**优化生命周期管理的优势：**

- 通过更好的资源利用降低基础设施成本
- 同一硬件上支持更多模型（超订）
- 通过智能预加载和缓存降低延迟
- 更快的模型切换改善用户体验
- 非活跃模型的休眠模式实现能效

---

## 模型冷启动优化

**关于 GPU Pod 冷启动策略的全面介绍，包括预预热池、更快的序列化格式和延迟加载模式，请参见 [GPU Pod 冷启动优化](../kubernetes/gpu-pod-cold-start.md)。**

模型冷启动指从存储加载模型、初始化并为推理做好准备所需的时间。这包括加载模型权重、初始化 GPU 内存、编译内核和预热缓存。本节重点关注模型级优化，而 GPU Pod 冷启动指南涵盖容器和编排级策略。

### 冷启动组件

总冷启动时间包括几个阶段：

1. **模型权重加载**（通常 5-30 秒）
   - 从磁盘/对象存储读取模型权重
   - 将权重传输到 GPU 内存
   - 取决于模型大小和存储 I/O 速度

2. **模型初始化**（通常 2-10 秒）
   - 初始化模型架构
   - 分配 GPU 内存缓冲区
   - 设置注意力机制和缓存

3. **内核编译**（通常 1-5 秒）
   - 为特定 GPU 编译 CUDA 内核
   - 可能使用缓存的内核编译结果
   - 首次编译成本较高

4. **缓存预热**（通常 0-2 秒）
   - 填充内部缓存
   - 运行初始推理遍
   - 优化内存布局

### 优化策略

#### 存储层优化

根据访问模式选择适当的存储层：

![LLM 推理的模型存储层](https://github.com/user-attachments/assets/8d2df23c-4dff-4100-952b-f6c95eb2e3b3)

*图 2：LLM 推理的模型存储层——从 GPU VRAM（最快）到远程对象存储（最慢）的层次化存储架构，层间具有传输机制。*

- **VRAM（GPU 内存）**：最快，用于热模型
- **RAM（主机内存）**：快速，用于温模型
- **SSD（本地存储）**：中等速度，用于缓存模型
- **远程存储**：最慢，用于冷模型

#### 模型格式优化

使用优化的模型格式减少加载时间：

- **Safetensors**：比基于 pickle 的格式更快更安全
- **内存映射文件**：避免将整个模型复制到内存
- **预编译内核**：缓存已编译的 CUDA 内核
- **量化模型**：更小尺寸、更快加载、更低内存使用

#### 预测性预加载

预测模型使用模式并预加载模型：

- **使用模式分析**：跟踪模型访问模式
- **基于时间的预加载**：在预期使用期之前加载模型
- **请求触发的预加载**：在第一个请求到达时开始加载
- **协同过滤**：基于用户模式预测模型使用

---

## 休眠模式与唤醒

休眠模式允许推理系统临时暂停非活跃模型，释放 GPU 资源同时保持快速重新激活能力。

### vLLM 休眠模式

[vLLM](https://github.com/vllm-project/vllm) 于 2025 年 10 月引入休眠模式，支持多模型服务的高效资源管理。休眠模式实现**比传统冷启动快 18-200 倍**的模型切换。

**关键特性：**

- **自动休眠**：模型在非活跃后自动进入休眠模式
- **GPU 内存释放**：释放 GPU 内存同时保留模型状态
- **快速唤醒**：从休眠状态快速恢复模型
- **KV 缓存保留**：可选择在休眠期间保留 KV 缓存
- **可配置超时**：为每个模型自定义休眠超时

**架构：**

休眠模式通过状态机运行：

1. **活跃状态**：模型正在处理请求
2. **空闲检测**：监控非活跃期（例如 5 分钟）
3. **休眠准备**：序列化模型状态和 KV 缓存
4. **休眠状态**：释放 GPU 内存，保留 CPU 状态
5. **唤醒触发**：新请求到达时恢复模型
6. **活跃状态**：恢复推理处理

**两个操作级别：**

#### 级别 1：CPU 内存卸载

- 模型权重移至 CPU RAM
- 唤醒时间：0.1-0.8 秒
- 适用于内存充足的环境
- 最快唤醒性能

#### 级别 2：完全丢弃权重

- 模型权重完全丢弃
- 内存占用：仅 MB 级别
- 适用于成本敏感型部署
- 唤醒时需要完全重新加载模型

**休眠模式为何快速：**

传统模型切换需要：❌ 重启 Python 进程、❌ 重新初始化 CUDA 上下文、❌ 重新编译 GPU 内核、❌ 重新捕获 CUDA 图

休眠模式保留进程状态：✅ 保留内存分配器、✅ 保留 CUDA 图、✅ 缓存 JIT 编译内核、✅ 保持完整进程状态

**性能基准：**

*A100 大型模型测试：*
- **Qwen3-235B（TP=4）**：冷启动 97.7s → 休眠唤醒 5.4s → **18 倍更快**
- **Qwen3-Coder-30B**：冷启动 47.4s → 休眠唤醒 2.9s → **17 倍更快**

*A4000 小型模型测试：*
- **Qwen3-0.6B + Phi-3-vision**：唤醒时间 0.1-0.8s → **58-203 倍更快**

*多切换场景：*
- 5 次模型切换：357s → 125s，**节省 65%（232 秒）**

**配置示例：**

```python
# vLLM 休眠模式配置
engine_config = {
    "sleep_mode_enabled": True,
    "sleep_mode_level": "level1",  # 或 "level2"
    "sleep_timeout_seconds": 300,  # 5 分钟
    "preserve_kv_cache": True,
    "max_sleeping_models": 10
}
```

**级别选择指南：**

- **选择级别 1**：当您有足够的 CPU RAM 且需要最快唤醒速度时（0.1-0.8s）
- **选择级别 2**：当内存成本至关重要且可以容忍稍长唤醒时间时

有关全面的模型切换文档，包括阿里 Aegaeon 的 token 级调度，请参见 [模型切换](./model-switching.md)。

### 唤醒优化

快速唤醒对保持低延迟至关重要：

- **部分状态恢复**：仅恢复基本模型组件
- **渐进式加载**：在完全恢复前开始推理
- **内存锁定**：在主机内存中保留模型权重
- **状态压缩**：压缩休眠状态以减少恢复时间

---

## 模型卸载

模型卸载根据访问模式和资源可用性在不同存储层之间移动模型。这实现了 GPU 资源的超订和成本效益的多模型服务。

### 卸载策略

#### 完全模型卸载

在存储层间移动整个模型：

- **GPU 到 CPU 内存**：通过 PCIe 快速传输（亚秒级）
- **CPU 内存到 SSD**：中等速度，适用于长期存储
- **SSD 到远程存储**：慢但容量无限

#### 逐层卸载

单独卸载模型层：

- 将频繁访问的层保留在 GPU 上
- 将不常访问的层移至 CPU 或磁盘
- 基于使用动态调整层放置

#### 自适应卸载

根据系统状态自动调整卸载：

- 监控 GPU 内存压力
- 跟踪模型访问模式
- 优化延迟与成本的权衡
- 实施驱逐策略（LRU、LFU、使用预测）

### 卸载的硬件加速

现代硬件特性加速模型卸载：

- **PCIe Gen 4/5**：16-32 GB/s 双向 CPU-GPU 传输
- **NVLink**：300-900 GB/s 多 GPU 系统
- **CXL（Compute Express Link）**：远程内存的内存语义访问
- **高速 RDMA**：节点间快速传输

---

## 快速模型驱动

快速模型驱动指最小化推理系统中不同模型或模型配置之间切换时间的技术。

### llm-d Fast Model Actuation

[llm-d fast-model-actuation](https://github.com/llm-d-incubation/llm-d-fast-model-actuation) 项目为 Kubernetes 环境中的快速模型切换提供机制。

**关键特性：**

- **预加载模型池**：维护可立即使用的模型池
- **模型路由**：智能路由到可用模型实例
- **资源管理**：高效分配 GPU 资源
- **Kubernetes 集成**：原生支持 K8s 调度

### 模型切换技术

#### 内存内切换

在 GPU 内存中保留多个模型并在它们之间切换：

- **多模型批处理**：在同一批次中运行不同模型
- **时间分片切换**：按时间间隔交替模型
- **基于优先级的切换**：基于请求优先级切换

#### LoRA 适配器切换

使用低秩适配（LoRA）进行快速模型切换：

- 基础模型保留在内存中
- 交换轻量级 LoRA 适配器（通常 10-100MB）
- 亚秒级切换时间
- 每个基础模型支持数百个适配器

---

## KV 缓存持久化

跨模型生命周期转换保留 KV 缓存对于保持用户体验和减少重新计算成本至关重要。

### KV 缓存卸载

[kvcached 项目](https://github.com/ovg-project/kvcached) 提供了在模型转换期间持久化 KV 缓存的机制。

**关键特性：**

- **KV 缓存序列化**：将 KV 缓存保存到持久存储
- **快速恢复**：模型唤醒时快速恢复 KV 缓存
- **压缩**：减少缓存数据的存储需求
- **分布式存储**：跨多个实例共享 KV 缓存

**与休眠模式集成：**

```python
# 概念性休眠模式与 KV 缓存持久化
async def sleep_model(model_id):
    kv_cache = capture_kv_cache(model_id)
    cache_id = await store_kv_cache(kv_cache)
    release_gpu_memory(model_id)
    save_wake_metadata(model_id, cache_id)

async def wake_model(model_id):
    await load_model_to_gpu(model_id)
    metadata = load_wake_metadata(model_id)
    kv_cache = await restore_kv_cache(metadata.cache_id)
    inject_kv_cache(model_id, kv_cache)
```

有关 KV 缓存管理的更多信息，请参见 [LLM 推理中的缓存](./caching.md)。

---

## 生产实现

### AIBrix 多模型管理

[AIBrix](https://github.com/vllm-project/aibrix) 提供生产就绪的多模型生命周期管理：

- **动态模型加载**：按需加载模型
- **LoRA 多路复用**：LoRA 适配器间高效切换
- **资源优化**：智能 GPU 内存管理
- **网关集成**：具有模型亲和性的请求路由

### KServe Model Server

[KServe](https://github.com/kserve/kserve) 提供模型生命周期能力：

- **Scale-to-zero**：自动缩放到零副本
- **快速扩展**：首次请求时快速加载模型
- **Knative 集成**：无服务器模型服务
- **多框架支持**：与各种推理引擎配合使用

---

## 最佳实践

### 何时使用每种技术

**使用休眠模式：**

- 模型有可预测的空闲期（>5 分钟）
- 每 GPU 支持多个模型
- 成本优化优先于绝对最低延迟
- 用户可以容忍 1-3 秒唤醒延迟

**使用模型卸载：**

- GPU 内存严重受限
- 支持非常大的模型目录
- 访问模式高度可变
- 基础设施成本是主要关注点

**使用快速模型驱动：**

- 需要频繁模型切换
- 需要亚秒级切换延迟
- 使用 LoRA 或基于适配器的模型
- 在 Kubernetes 等编排环境中运行

**使用 KV 缓存持久化：**

- 用户进行长对话
- 模型经历休眠/唤醒周期
- 上下文保留对用户体验至关重要
- 重新计算导致服务成本高

### 监控和可观测性

关键跟踪指标：

- **冷启动延迟**：从请求到首 token 的时间
- **唤醒延迟**：从休眠到活跃状态的时间
- **休眠转换时间**：进入休眠模式的时间
- **GPU 内存利用率**：生命周期转换前后
- **模型切换率**：模型变更频率
- **KV 缓存命中率**：恢复与重新计算的缓存百分比

---

## 参考资料

- [vLLM 休眠模式博客](https://blog.vllm.ai/2025/10/26/sleep-mode.html)
- [kvcached 项目](https://github.com/ovg-project/kvcached)
- [llm-d Fast Model Actuation](https://github.com/llm-d-incubation/llm-d-fast-model-actuation)
- [vLLM GitHub 仓库](https://github.com/vllm-project/vllm)
- [AIBrix 介绍](./aibrix.md)
- [LLM 推理中的缓存](./caching.md)
- [KServe Model Server](https://github.com/kserve/kserve)
- [Kubernetes Serving WG](https://github.com/kubernetes/community/blob/master/wg-serving/README.md)

**部分内容基于所提供的 GitHub 参考和博客文章整理。实施前请核实技术细节和当前项目状态。**
