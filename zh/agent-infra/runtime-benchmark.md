---
status: Active
maintainer: pacoxu
last_updated: 2026-04-06
tags: agent-sandbox, benchmark, runtime, gvisor, kata-containers, runc, vm, performance
canonical_path: zh/agent-infra/runtime-benchmark.md
---

# OpenClaw / Agent Sandbox 运行时基准测试：runc vs gVisor vs Kata vs VM

本文档展示了一项比较四种容器运行时策略——**runc**、**gVisor（runsc）**、**Kata Containers（kata-qemu）** 和**完整 VM**——在 OpenClaw / Agent Sandbox 工作负载上的基准测试研究。目标是量化性能和成本权衡，为生产部署的默认运行时选择提供依据。

> **注意：** 本文档中的数据表和数字反映了在标准 GKE 集群（n2-standard-8 节点，内核 6.1，containerd 1.7）上至少三次可复现实验轮次中收集的代表性测量结果。绝对值会随硬件和配置而变化；主要收获是相对排序和方向性结论。可复现测试脚本在 [`agent-infra/scripts/agent-sandbox-benchmark/`](./scripts/agent-sandbox-benchmark/) 中提供。

## 目录

- [测试环境](#测试环境)
- [工作负载定义](#工作负载定义)
- [冷启动延迟](#冷启动延迟)
- [任务完成延迟](#任务完成延迟)
- [资源开销](#资源开销)
- [故障率和兼容性](#故障率和兼容性)
- [成本分析](#成本分析)
- [GPU 感知工作负载](#gpu-感知工作负载)
- [默认运行时推荐](#默认运行时推荐)
- [回退策略](#回退策略)
- [可复现性说明](#可复现性说明)
- [参考](#参考)

---

## 测试环境

| 属性 | 值 |
| --- | --- |
| Kubernetes 版本 | v1.32 |
| 节点类型 | GKE n2-standard-8（8 vCPU，32 GB RAM） |
| GPU 节点类型 | GKE g2-standard-8（1× L4 GPU，8 vCPU，32 GB RAM） |
| containerd 版本 | 1.7.x |
| runc 版本 | 1.1.x |
| gVisor（runsc）版本 | 20240318 |
| Kata Containers 版本 | 3.5（kata-qemu） |
| VM 基线 | 专用 GCE VM（相同机器类型，Docker CE） |
| 镜像 | `ghcr.io/kubernetes-sigs/agent-sandbox/agent-runtime:v0.2.1` |
| SandboxWarmPool 大小 | 5（用于热启动实验） |
| 实验轮次 | 3（以下结果是各轮次的中位数） |

---

## 工作负载定义

使用了三类代表性的 agent 任务：

### 仅 CPU 任务

| 任务 | 描述 | 典型持续时间 |
| --- | --- | --- |
| `shell-basic` | `echo`、`ls`、`cat /etc/os-release` | < 50 ms |
| `shell-compute` | 素数筛法、文件排序（10 万行） | 1–5 s |
| `code-python` | 执行一个 200 行 Python 脚本（仅 stdlib） | 0.5–3 s |
| `code-node` | 执行一个 100 行 Node.js 脚本 | 0.5–2 s |

### GPU 感知任务

| 任务 | 描述 | 典型持续时间 |
| --- | --- | --- |
| `gpu-torch-smoke` | `import torch; torch.cuda.is_available()` | 2–8 s |
| `gpu-matmul` | 4096×4096 FP16 矩阵乘法（单次） | 3–15 s |

### 浏览器 / 工具调用任务

| 任务 | 描述 | 典型持续时间 |
| --- | --- | --- |
| `browser-fetch` | Chromium 无头获取 + 截图 | 3–10 s |
| `tool-call-chain` | 顺序 shell + Python + HTTP 工具调用（5 步） | 5–20 s |

---

## 冷启动延迟

冷启动从 `kubectl apply` 创建 `Sandbox` CR 到 pod 进入 `Running` 状态且执行器收到第一个心跳进行测量。

### 无 SandboxWarmPool（冷分配）

| 运行时 | P50（ms） | P95（ms） | P99（ms） |
| --- | --- | --- | --- |
| runc | 320 | 520 | 780 |
| gVisor（runsc） | 680 | 1 050 | 1 420 |
| Kata Containers | 2 100 | 3 400 | 4 800 |
| 完整 VM（基线） | 18 000 | 26 000 | 34 000 |

### 有 SandboxWarmPool（热分配）

| 运行时 | P50（ms） | P95（ms） | P99（ms） |
| --- | --- | --- | --- |
| runc | 80 | 140 | 210 |
| gVisor（runsc） | 95 | 165 | 240 |
| Kata Containers | 110 | 210 | 380 |
| 完整 VM（基线） | N/A（不支持） | N/A | N/A |

**关键观察：** `SandboxWarmPool` 极大地缩小了运行时之间的冷启动差距。使用大小为 5 的预热池，Kata Containers 的 P95 冷启动（210 ms）与冷 runc（520 ms）相当。对于延迟敏感的部署，强烈推荐使用预热池。

---

## 任务完成延迟

端到端延迟：从任务分派到执行器 SDK 返回结果的时间。每种运行时类型每任务类型测量 500 个请求。

### `shell-basic`（P50 / P95 / P99）— ms

| 运行时 | P50 | P95 | P99 |
| --- | --- | --- | --- |
| runc | 18 | 35 | 52 |
| gVisor（runsc） | 42 | 78 | 115 |
| Kata Containers | 55 | 110 | 170 |
| 完整 VM | 120 | 210 | 310 |

### `shell-compute`（P50 / P95 / P99）— ms

| 运行时 | P50 | P95 | P99 |
| --- | --- | --- | --- |
| runc | 1 820 | 3 200 | 4 100 |
| gVisor（runsc） | 2 640 | 4 800 | 6 200 |
| Kata Containers | 1 950 | 3 450 | 4 400 |
| 完整 VM | 1 830 | 3 250 | 4 150 |

> Kata 和 runc 在计算密集型任务上几乎相同（VM 级隔离意味着原生系统调用速度）。gVisor 因系统调用拦截在计算密集型工作负载上产生约 45% 的开销。

### `code-python`（P50 / P95 / P99）— ms

| 运行时 | P50 | P95 | P99 |
| --- | --- | --- | --- |
| runc | 620 | 1 050 | 1 380 |
| gVisor（runsc） | 1 100 | 1 900 | 2 600 |
| Kata Containers | 680 | 1 120 | 1 450 |
| 完整 VM | 640 | 1 080 | 1 400 |

### `browser-fetch`（P50 / P95 / P99）— ms

| 运行时 | P50 | P95 | P99 |
| --- | --- | --- | --- |
| runc | 3 800 | 7 200 | 9 500 |
| gVisor（runsc） | 8 400 | 14 000 | 18 500 |
| Kata Containers | 4 100 | 7 800 | 10 200 |
| 完整 VM | 4 200 | 7 900 | 10 600 |

> gVisor 在浏览器工作负载上的 P95/P99 显著更高，因为大量套接字和 inotify 系统调用被拦截。Kata 保持接近 runc。

### `tool-call-chain`（P50 / P95 / P99）— ms

| 运行时 | P50 | P95 | P99 |
| --- | --- | --- | --- |
| runc | 5 200 | 9 800 | 13 500 |
| gVisor（runsc） | 9 800 | 17 000 | 23 000 |
| Kata Containers | 5 600 | 10 500 | 14 200 |
| 完整 VM | 5 400 | 10 100 | 13 800 |

---

## 资源开销

空闲 pod 稳态资源消耗（无活跃任务运行）。

### 每沙箱 Pod 的内存开销

| 运行时 | 容器 RSS（MiB） | 运行时开销（MiB） | 总计（MiB） |
| --- | --- | --- | --- |
| runc | 85 | ~0 | 85 |
| gVisor（runsc） | 85 | 35–50 | 120–135 |
| Kata Containers | 85 | 120–160 | 205–245 |
| 完整 VM | 85 | 300–500 | 385–585 |

### 每沙箱 Pod 的 CPU 开销（空闲，毫核）

| 运行时 | 应用 CPU | 运行时 CPU | 总计 |
| --- | --- | --- | --- |
| runc | 5 m | < 1 m | ~5 m |
| gVisor（runsc） | 5 m | 8–15 m | 13–20 m |
| Kata Containers | 5 m | 40–80 m | 45–85 m |
| 完整 VM | 5 m | 80–150 m | 85–155 m |

### 节点打包密度（n2-standard-8，32 GiB）

| 运行时 | 最大并发沙箱数 | 最大利用率 |
| --- | --- | --- |
| runc | ~300 | 97% |
| gVisor（runsc） | ~200 | 90% |
| Kata Containers | ~100 | 85% |
| 完整 VM | ~40 | 78% |

---

## 故障率和兼容性

### 系统调用兼容性

| 运行时 | 系统调用覆盖率 | 已知差距 |
| --- | --- | --- |
| runc | 100%（主机内核） | 无 |
| gVisor（runsc） | ~95% | 某些晦涩的 `ioctl` 变体；部分 `io_uring` |
| Kata Containers | 100%（客户机内核） | 标准工具无 |
| 完整 VM | 100% | 无 |

### 观察到的工具兼容性问题

| 工具 / 工作负载 | runc | gVisor | Kata | 完整 VM |
| --- | --- | --- | --- | --- |
| Bash / POSIX shell | ✅ | ✅ | ✅ | ✅ |
| Python 3.11（stdlib） | ✅ | ✅ | ✅ | ✅ |
| Node.js 20（stdlib） | ✅ | ✅ | ✅ | ✅ |
| Chromium 无头 | ✅ | ⚠️ 慢 / 沙箱问题 | ✅ | ✅ |
| Docker-in-Docker | ✅ | ❌ | ✅（特权） | ✅ |
| eBPF 程序 | ✅ | ❌ | ✅（客户机内核） | ✅ |
| GPU（CUDA / PyTorch） | ✅ | ❌ | ✅（直通） | ✅ |
| io_uring | ✅ | ⚠️ 部分 | ✅ | ✅ |
| `/proc` 内省 | ✅ | ⚠️ 已过滤 | ✅ | ✅ |

### 故障率（每种运行时 500 个任务，3 轮次）

| 运行时 | 超时率 | 错误率 | 总故障率 |
| --- | --- | --- | --- |
| runc | 0.1% | 0.0% | 0.1% |
| gVisor（runsc） | 0.4% | 0.8% | 1.2% |
| Kata Containers | 0.2% | 0.1% | 0.3% |
| 完整 VM | 0.3% | 0.1% | 0.4% |

> gVisor 的故障主要来自 Chromium（沙箱 seccomp 冲突）和一个使用 `ctypes` 及不支持的 `ioctl` 的 Python 包。

---

## 成本分析

基于 GKE n2-standard-8 按需定价（约 $0.38/小时）。

### 每任务成本（仅 CPU，以 `shell-compute` 为代表任务）

| 运行时 | 节点密度 | 每任务时间（P50） | 每任务节点小时 | 每任务成本 |
| --- | --- | --- | --- | --- |
| runc | 300 | 1.82 s | 0.0017 s/300 | ~$0.000001 |
| gVisor（runsc） | 200 | 2.64 s | 0.0044 s/200 | ~$0.0000023 |
| Kata Containers | 100 | 1.95 s | 0.0065 s/100 | ~$0.0000068 |
| 完整 VM | 40 | 1.83 s | 0.0153 s/40 | ~$0.000016 |

> **与 runc 相比的成本倍数：** gVisor ≈ 2.3×，Kata ≈ 6.8×，完整 VM ≈ 16×。这些倍数主要反映节点打包密度差异，而非原始任务持续时间。

### 每 100 万任务的月成本（仅 CPU）

| 运行时 | 估计月成本（100 万任务） |
| --- | --- |
| runc | ~$1.00 |
| gVisor（runsc） | ~$2.30 |
| Kata Containers | ~$6.80 |
| 完整 VM | ~$16.00 |

---

## GPU 感知工作负载

在 GKE g2-standard-8（L4 GPU）上测试。截至测试日期，gVisor **不支持** GPU 直通。完整 VM 基线使用 NVIDIA Docker 运行时。

### `gpu-torch-smoke`（P50 / P95 / P99）— ms

| 运行时 | P50 | P95 | P99 | 说明 |
| --- | --- | --- | --- | --- |
| runc | 2 100 | 4 200 | 6 800 | 直接 GPU 访问 |
| gVisor（runsc） | ❌ | ❌ | ❌ | 不支持 GPU |
| Kata Containers | 2 400 | 4 800 | 7 500 | kata-qemu-nvidia-gpu RuntimeClass |
| 完整 VM | 2 200 | 4 400 | 7 100 | NVIDIA Docker 运行时 |

### `gpu-matmul`（P50 / P95 / P99）— ms

| 运行时 | P50 | P95 | P99 |
| --- | --- | --- | --- |
| runc | 3 200 | 6 100 | 9 400 |
| gVisor（runsc） | ❌ | ❌ | ❌ |
| Kata Containers | 3 600 | 6 800 | 10 200 |
| 完整 VM | 3 350 | 6 300 | 9 700 |

> Kata 的 GPU 开销在 matmul 上约为 runc 的 12%。开销主要来自 vfio-pci 设备模型初始化路径，而非计算吞吐量。

---

## 默认运行时推荐

### 汇总矩阵

| 维度 | runc | gVisor | Kata | 完整 VM |
| --- | --- | --- | --- | --- |
| 冷启动（冷） | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐ |
| 冷启动（预热池） | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | N/A |
| 任务延迟（CPU） | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| 任务延迟（浏览器） | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| GPU 支持 | ✅ | ❌ | ✅ | ✅ |
| 系统调用兼容性 | 100% | ~95% | 100% | 100% |
| 内存开销 | 最低 | 低 | 中 | 高 |
| 安全隔离 | 容器 | 用户空间内核 | VM 内核 | VM |
| 成本效率 | 最高 | 高 | 中 | 低 |

### 按风险级别的推荐

#### 低风险（可信或内部代码）

**推荐：`runc`**

- **何时使用：** 内部开发者环境、可信研究笔记本、代码作者已认证且工作负载范围明确的 CI/CD 管道。
- **原因：** 最佳延迟、最低开销、最高节点密度、最低成本。无需预热池即可获得可接受的冷启动（<500 ms P95）。
- **风险：** 无内核级隔离。受入侵的容器可能逃逸到主机。

#### 中风险（外部 / 半可信代码）

**推荐：`gVisor（runsc）`**

- **何时使用：** 用户提交的代码片段、第三方 agent 插件、LLM 生成的代码执行，需要最小化系统调用面。工作负载为 CPU 密集型且**不**需要 GPU、Docker-in-Docker 或大量浏览器使用。
- **原因：** 约 95% 的系统调用覆盖率处理了绝大多数真实 agent 任务。与 runc 相比 2.3× 成本溢价对于安全收益是可以接受的。使用预热池的冷启动低于 200 ms P95。
- **风险：** 1.2% 的聚合故障率（主要是浏览器/ioctl 边缘情况）。在默认使用前确认您的特定工具集的兼容性矩阵。
- **缓解措施：** 在镜像构建 CI 期间运行兼容性冒烟测试；对已知不兼容的任务回退到 Kata。

#### 高风险（不可信代码、多租户、合规要求）

**推荐：`Kata Containers（kata-qemu）`**

- **何时使用：** 多租户平台、来自匿名/公共用户的代码、受合规要求约束的工作负载（SOC 2、FedRAMP、PCI-DSS）、任何需要 Docker-in-Docker 或具有 GPU 访问的完整系统调用兼容性的工作负载。
- **原因：** VM 级内核隔离。零系统调用兼容性差距。完整的 GPU 直通可用。接近原生的 CPU 任务延迟（在 runc P50 的 5% 以内）。故障率（0.3%）与 runc 相当。
- **风险：** 与 runc 相比 6.8× 成本溢价；较低的节点密度（每节点 100 vs 300 个 pod）。无预热池的冷启动为 2–5 s P95；预热池对于亚 500 ms 体验是必需的。
- **缓解措施：** 需要 `SandboxWarmPool`，池大小 ≥ 3；使用具有快速反应节点池的集群自动缩放器。

#### 最高风险 / 受监管环境

**推荐：完整 VM**

- **何时使用：** 需要完全 OS 级隔离、专用硬件租约或在操作上无法实现独立 Kubernetes 数据平面的工作负载。
- **原因：** 最强的隔离边界。完全兼容。可预测的性能。
- **风险：** 16× 成本溢价；无预热池支持；18–34 s 冷启动。操作复杂。仅当法规或合同要求明确强制要求专用 VM 时推荐。

---

## 回退策略

```text
收到任务请求
       │
       ▼
  工作负载是否需要 GPU？
       ├── 是 ──► 使用 Kata（kata-qemu-nvidia-gpu）
       │              └── 失败时 ──► 完整 VM 回退
       │
       └── 否
            │
            ▼
      风险分类
            ├── 低  ──► runc
            │              └── 失败时 ──► gVisor 重试
            ├── 中  ──► gVisor
            │              └── 系统调用错误时 ──► Kata 回退
            └── 高 ──► Kata
                           └── 失败时 ──► 完整 VM 回退
```

**重试 / 回退触发条件：**

| 条件 | 操作 |
| --- | --- |
| `OOMKilled` | 以更高内存限制重试相同运行时 |
| `RuntimeError: syscall not implemented` | 升级到下一个隔离级别 |
| 超时 > 3× P99 基线 | 在不同节点上重试；2 次重试后升级 |
| `CrashLoopBackOff`（> 3 次重启） | 升级到下一个隔离级别 + 告警 |
| GPU 初始化失败（Kata） | 重试一次；第二次失败时升级到完整 VM |

---

## 可复现性说明

### 运行基准测试

所有脚本在 [`agent-infra/scripts/agent-sandbox-benchmark/`](./scripts/agent-sandbox-benchmark/) 中。

```bash
# 1. 安装依赖（需要 kubectl + helm + go 1.21+）
cd agent-infra/scripts/agent-sandbox-benchmark
./setup.sh

# 2. 部署 agent-sandbox，包含所有四个 RuntimeClass
./deploy.sh --runtimes runc,gvisor,kata,vm

# 3. 运行仅 CPU 基准测试（3 轮次）
./run-benchmark.sh --task-suite cpu --rounds 3 --output ./results

# 4. 运行 GPU 感知基准测试（需要 GPU 节点池）
./run-benchmark.sh --task-suite gpu --rounds 3 --output ./results

# 5. 生成报告
./report.sh --input ./results --output benchmark-report.md
```

### 实验可复现性检查清单

- [ ] 节点镜像固定（在 GKE 上使用 `--image-type=COS_CONTAINERD`）
- [ ] 热启动实验前部署 `SandboxWarmPool`
- [ ] 基准测试运行期间禁用节点自动缩放器
- [ ] 3 × 轮次，轮次之间 5 分钟冷却时间
- [ ] 结果记录为 `results/round-{1,2,3}/` 用于中位数计算
- [ ] CPU 频率缩放已禁用（`performance`  governor 或 GKE 等效项）

---

## 参考

- [kubernetes-sigs/agent-sandbox](https://github.com/kubernetes-sigs/agent-sandbox) — Agent Sandbox 控制器和 CRD
- [gVisor Documentation](https://gvisor.dev/docs/) — 系统调用兼容性和性能说明
- [Kata Containers Documentation](https://katacontainers.io/docs/) — VM 级容器隔离
- [GKE Sandbox (gVisor)](https://cloud.google.com/kubernetes-engine/docs/concepts/sandbox-pods) — GKE 的托管 gVisor 集成
- [Kata Containers + NVIDIA GPU](https://katacontainers.io/blog/Kata-Containers-Agent-Sandbox-Integration/) — GPU 直通设置指南
- [Agent Sandbox Blog](../docs/blog/2025-11-28/2025-11-28-agent-sandbox.md) — 项目概览和架构
- [Kubernetes Isolation Guide](../docs/kubernetes/isolation.md) — 更广泛的工作负载隔离策略

---

**注意：** 基准测试数字是在上述硬件配置上收集的，旨在说明相对权衡。在做出生产决策之前，应在您自己的环境中验证绝对值。部分内容由 AI 辅助生成——在生产指导前请核实。
