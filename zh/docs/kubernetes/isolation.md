---
status: Active
maintainer: pacoxu
last_updated: 2026-04-22
tags: kubernetes, isolation, security, multi-tenancy
canonical_path: zh/docs/kubernetes/isolation.md
---

# AI 工作负载隔离

隔离是 Kubernetes 上运行 AI 工作负载的关键关注点，特别是在多租户环境中，安全性、资源边界和可预测的性能至关重要。本文档探讨了云原生基础设施中 AI 工作负载的各种隔离机制。

## 为什么隔离对 AI 工作负载重要

AI 工作负载带来了独特的隔离挑战：

- **资源密集型**：GPU/TPU 访问、大内存占用
- **长时间运行**：训练作业可能运行数天或数周
- **多租户**：共享基础设施需要强安全边界
- **敏感数据**：模型和训练数据通常包含专有或机密信息
- **成本优化**：防止噪声邻居并确保公平资源共享

## 1. Control Groups（cgroups）文件系统

Control groups（cgroups）通过在内核级别管理和限制资源使用，为容器提供基础的隔离机制。

### 关键概念

- **资源控制器**：CPU、内存、I/O、网络带宽
- **分层组织**：嵌套的 cgroup 结构实现细粒度控制
- **cgroups v1 vs v2**：现代系统使用统一的 cgroups v2，具有改进的资源管理

### AI 特定考量

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ai-training-job
spec:
  containers:
  - name: trainer
    resources:
      limits:
        cpu: "16"
        memory: "64Gi"
        nvidia.com/gpu: "4"
      requests:
        cpu: "8"
        memory: "32Gi"
        nvidia.com/gpu: "4"
```

### 项目和资源

- <a href="https://www.kernel.org/doc/Documentation/cgroup-v2.txt">cgroups v2 文档</a>
- <a href="https://github.com/opencontainers/runc">runc</a>：OCI 运行时参考实现，集成了 cgroup

## 2. 权限和安全上下文

权限管理确保工作负载以最小必要权限运行，减少攻击面和爆炸半径。

### 安全上下文选项

- **runAsNonRoot**：强制以非 root 用户执行
- **runAsUser/runAsGroup**：指定容器的 UID/GID
- **readOnlyRootFilesystem**：防止写入容器文件系统
- **allowPrivilegeEscalation**：阻止权限提升尝试
- **capabilities**：删除不必要的 Linux 能力

### AI 工作负载最佳实践

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-ai-inference
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: model-server
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
```

### 项目和资源

- <a href="https://kubernetes.io/docs/tasks/configure-pod-container/security-context/">Kubernetes 安全上下文</a>
- <a href="https://kubernetes.io/docs/concepts/security/pod-security-standards/">Pod 安全标准</a>

## 3. 用户命名空间隔离

用户命名空间通过将容器 UID/GID 映射到主机系统上的不同范围，提供额外的隔离层。

### 工作原理

- 容器内进程看似以 root（UID 0）运行
- 主机内核将其映射到非特权 UID（例如 100000-165535）
- 被攻破的容器无法以 root 身份访问主机资源

### Kubernetes 支持

Kubernetes 中用户命名空间支持正在积极开发：

- **v1.25 中 Alpha**：在特性门控后的初始支持
- **v1.30 中 Beta**：改进的稳定性和更广泛的运行时支持

### 配置示例

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ai-workload-userns
spec:
  hostUsers: false  # 启用用户命名空间
  containers:
  - name: training
    image: pytorch/pytorch:latest
```

### 项目和资源

- <a href="https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/127-user-namespaces">Kubernetes KEP-127：用户命名空间</a>
- <a href="https://man7.org/linux/man-pages/man7/user_namespaces.7.html">Linux user_namespaces 手册页</a>

## 4. Rootless 容器运行时

Rootless 容器运行时在不需要主机 root 权限的情况下运行整个容器运行时。

### 优势

- **减少攻击面**：运行时守护进程以普通用户身份运行
- **无需 Root**：适用于共享 HPC 环境
- **深度防御**：为不可信工作负载提供额外隔离层

### 实现

#### Rootless Docker/containerd

<a href="https://docs.docker.com/engine/security/rootless/">Rootless Docker</a> 允许以非 root 用户运行 Docker 守护进程，使用用户命名空间和 slirp4netns 进行网络。

#### Rootless Podman

<a href="https://github.com/containers/podman">Podman</a> 提供原生 rootless 支持，无需守护进程，使用 fuse-overlayfs 进行存储。

### Kubernetes 集成

```yaml
# CRI-O rootless 模式配置
[crio.runtime]
  default_runtime = "runc"
  [crio.runtime.runtimes.runc]
    runtime_path = "/usr/bin/runc"
    runtime_type = "oci"
    allowed_annotations = []
  [crio.runtime.runtimes.crun-rootless]
    runtime_path = "/usr/bin/crun"
    runtime_type = "oci"
```

### AI 工作负载的限制

- **GPU 访问**：rootless 模式下 GPU 支持有限或没有
- **性能**：用户空间驱动的网络和存储开销
- **兼容性**：某些设备插件可能无法工作

### 项目和资源

- <a href="https://rootlesscontaine.rs/">Rootless Containers</a>：社区中心
- <a href="https://github.com/containers/crun">crun</a>：快速且低内存的 OCI 运行时，支持 rootless

## 5. 基于虚拟机的隔离

基于 VM 的隔离通过在轻量级虚拟机内运行容器，提供更强的安全边界。

### Kata Containers

<a href="https://github.com/kata-containers/kata-containers">Kata Containers</a> 是一个开源项目，提供基于 VM 的容器隔离，兼容 OCI。

#### 架构

- **硬件虚拟化**：利用 KVM、Firecracker 或 Cloud Hypervisor
- **轻量级 VM**：快速启动时间（亚秒级）且开销最小
- **Kubernetes 集成**：作为 RuntimeClass 工作

#### 配置示例

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: kata
handler: kata
---
apiVersion: v1
kind: Pod
metadata:
  name: ai-inference-kata
spec:
  runtimeClassName: kata
  containers:
  - name: model-server
    image: pytorch/torchserve:latest
    resources:
      limits:
        nvidia.com/gpu: "1"
```

#### GPU 支持

Kata Containers 支持 GPU 透传：

- 直接设备分配，实现裸金属性能
- VFIO 实现安全设备隔离
- 兼容 NVIDIA GPU、AMD GPU

### gVisor

<a href="https://github.com/google/gvisor">gVisor</a> 通过用户空间内核拦截系统调用，提供应用内核隔离。

#### 架构

- **用户空间内核**：用 Go 实现 Linux 内核接口
- **系统调用拦截**：阻止直接内核访问
- **深度防御**：减少内核攻击面

#### 配置示例

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc
---
apiVersion: v1
kind: Pod
metadata:
  name: ai-workload-gvisor
spec:
  runtimeClassName: gvisor
  containers:
  - name: training
    image: tensorflow/tensorflow:latest
```

#### gVisor 快照用于快速冷启动

gVisor 支持容器快照，能够快速恢复预初始化的容器：

- **预预热状态**：捕获初始化的运行时、加载的库、依赖
- **快速恢复**：从快照恢复，而不是从镜像冷启动
- **一致性能**：消除不同启动间的初始化差异
- **生产使用**：在 GKE Agent Sandbox 中验证，冷启动提升 90%

#### AI 工作负载的限制

- **不支持 GPU**：gVisor 不支持 GPU 透传（适用于仅 CPU 的 Agent 工作负载）
- **性能开销**：系统调用拦截增加延迟（快照可缓解）
- **兼容性**：部分系统调用未实现

### Kata vs gVisor 对比（用于 AI）

| 特性 | Kata Containers | gVisor |
|---------|----------------|--------|
| GPU 支持 | ✅ 是 | ❌ 否 |
| 开销 | 低（接近原生） | 中（系统调用开销） |
| 启动时间 | 快（< 1s） | 非常快（< 100ms） |
| 内存开销 | 每 VM ~130MB | 每沙箱 ~15MB |
| 安全模型 | 硬件隔离 | 软件隔离 |
| AI 用例 | GPU 密集型训练/推理 | 仅 CPU 工作负载 |

### 项目和资源

- <a href="https://katacontainers.io/">Kata Containers 官方网站</a>
- <a href="https://gvisor.dev/">gVisor 官方网站</a>
- <a href="https://github.com/firecracker-microvm/firecracker">Firecracker</a>：AWS 轻量级 VM，用于安全多租户容器

## 6. Agent 沙箱

Agent 沙箱为 AI Agent 提供安全的执行环境，这些 Agent 可以执行任意代码、访问工具并与外部系统交互。与传统工作负载隔离不同，Agent 沙箱必须处理不可信的 LLM 生成代码，同时支持快速启动和高并发。

### 核心要求

生产级 Agent 沙箱必须满足：

1. **快速冷启动**：理想情况下低于 100ms，实现响应式 Agent 调用
2. **强安全性**：有效隔离不可信代码，防止权限提升
3. **Python 支持**：兼容 Agent 使用的常见 Python 库
4. **便捷镜像构建**：构建和部署自定义 Agent 镜像的简单工作流

### Agent 运行时安全基线（OpenClaw 的 Claw Profile）

对于需要简洁回答"如何限制 Agent 权限并保持系统安全"的平台团队，使用分层基线：**身份 + 执行 + 网络 + 工具/数据**，默认失败关闭。

#### 分层护栏（默认开启）

| 层 | 默认策略 | 重要性 |
| ----- | -------------- | -------------- |
| 身份 | 每个会话的服务身份、短期令牌、沙箱内无长期访问密钥 | 限制爆炸半径和凭证重用 |
| 执行 | `runAsNonRoot`、`readOnlyRootFilesystem`、`allowPrivilegeEscalation=false`、`capabilities.drop=["ALL"]`、`seccompProfile=RuntimeDefault` | 阻止常见权限提升路径 |
| 网络 | 默认拒绝所有出站，允许列表仅包含必要目标（通常通过出口代理） | 防止任意数据泄露和命令与控制流量 |
| 工具和数据 | 工具允许列表 + 参数验证 + 破坏性操作需人工审批；临时工作空间，无密钥持久化 | 减少高风险操作和跨会话泄漏 |

最小加固 Pod 设置：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: claw-agent-sandbox
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: runtime
      image: your-agent-image:latest
      securityContext:
        runAsNonRoot: true
        readOnlyRootFilesystem: true
        allowPrivilegeEscalation: false
        capabilities:
          drop: ["ALL"]
```

网络基线（默认拒绝所有）：

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: claw-agent-default-deny-egress
spec:
  podSelector: {}
  policyTypes: ["Egress"]
  egress: []
```

#### 后备机制

1. **策略默认拒绝**：任何超出显式策略的内容都被拒绝。
2. **失败关闭**：在异常行为（权限尝试、禁止出站、重复策略违规）时，终止沙箱并撤销会话凭证。

#### 预生产验证清单

1. **权限提升测试**：使用 `privileged`、`hostPID`、`hostNetwork` 或 `hostPath` 的 Pod 规格被准入策略拒绝。
2. **凭证边界测试**：会话令牌按预期过期，且不能跨会话重用。
3. **网络边界测试**：非允许列表的出站目标被阻止。
4. **可审计性测试**：每个高风险工具调用都有可追踪的 `谁/何时/什么/目标/结果` 记录。
5. **清理测试**：会话终止后删除临时文件和凭证。

### 两种架构模式

当 Agent 可以执行任意代码时，有两种基本架构模式可以保护密钥和基础设施安全：

**模式 1：隔离工具**

Agent 循环运行在您的基础设施上。危险操作（代码执行、终端访问）在独立的沙箱中运行。Agent 通过 HTTP 调用沙箱。代码在隔离环境中运行，没有任何可泄漏的内容。

```
+----------------------------+
|         您的后端              |
|  +-----------------------+   |
|  |    Agent 循环          |   |      +-------------+
|  |  （在您的基础设施上）    |---+-----|   沙箱       |
|  +-----------------------+   | HTTP |  （终端，     |
|                              |      |   代码执行）  |
+------------------------------+      +-------------+
```

这种模式更易于采用，但 Agent 仍然与您的后端一起运行——内存占用大的 Agent 可能拖慢 API，重新部署会杀死所有正在运行的 Agent。

**模式 2：隔离 Agent**

整个 Agent 在零密钥的沙箱中运行。它通过持有所有凭证的控制平面与外部世界通信。Agent 变得完全可丢弃：没有可窃取的密钥，没有需要保留的状态。

```
+------------+   会话令牌   +------------------+
|   沙箱     |------------>|   控制平面       |
|  （Agent,  |<------------|  （凭证,         |
|   零密钥）  |  结果/数据   |   LLM 代理,     |
|            |              |   文件存储）     |
+------------+              +------------------+
```

控制平面代理所有敏感操作：

- **LLM 调用**：沙箱仅发送新消息；控制平面从数据库重建完整对话历史
- **文件同步**：沙箱请求预签名 URL 并直接上传到对象存储——从不看到云凭证
- **计费和成本上限**：在控制平面级别执行

来自 <a href="https://browser-use.com/posts/two-ways-to-sandbox-agents">Browser Use</a> 的关键见解：*"您的 Agent 应该没有任何值得窃取的东西，也没有任何值得保留的东西。"*

模式 2 是生产首选：每一层（沙箱、控制平面）都根据自身瓶颈独立伸缩。

### 沙箱技术比较

| 技术 | 冷启动 | 安全性 | Python 支持 | 镜像构建 |
| ---------- | --------- | -------- | -------------- | ----------- |
| 容器（Docker） | ~50ms | 共享内核 | 完整 | Docker |
| Firecracker microVM | ~125ms（+快照 <1s） | 强（独立内核） | 完整 | ext4 rootfs |
| Kata Containers | ~125ms | 强（独立内核） | 完整 | OCI/Docker |
| WASM（monty） | 0.06ms | 沙箱 | 有限子集 | 自定义 |
| Unikernel（Unikraft） | <100ms | 非常强 | 持续改进 | 自定义 |

### 分层 Agent 沙箱选择（2026-04-22）

Agent 沙箱项目现在覆盖多个层面。对于 Kubernetes 生产设计，最重要的决策是每个项目在栈中的位置：

| 层 | 项目 | 适合 Kubernetes 隔离 |
| ----- | -------- | ---------------------------- |
| Kubernetes 生命周期 API | [kubernetes-sigs/agent-sandbox](https://github.com/kubernetes-sigs/agent-sandbox)、[agent-sandbox/agent-sandbox](https://github.com/agent-sandbox/agent-sandbox)、[volcano-sh/agentcube](https://github.com/volcano-sh/agentcube) | 用于 `Sandbox` CRD、预热池、稳定标识和调度器集成。此层级仍需要下方的运行时边界。 |
| Agent 沙箱平台 | [OpenSandbox](https://github.com/alibaba/OpenSandbox)、[CubeSandbox](https://github.com/TencentCloud/CubeSandbox)、[E2B](https://github.com/e2b-dev/e2b)、[Daytona](https://github.com/daytonaio/daytona)、[Sandbox0](https://github.com/sandbox0-ai/sandbox0) | 用于 SDK/API、模板、命令/文件/浏览器执行和自托管服务操作。在嵌入前验证许可和部署模型。 |
| 运行时边界 | [gVisor](https://github.com/google/gvisor)、[Kata Containers](https://github.com/kata-containers/kata-containers)、[containerd/nerdbox](https://github.com/containerd/nerdbox)、[Firecracker](https://github.com/firecracker-microvm/firecracker)、[Kuasar](https://github.com/kuasar-io/kuasar) | 用作实际隔离边界：gVisor 用于密度，Kata 用于 VM 级隔离，Firecracker 用于自定义 microVM 控制平面，Kuasar 用于多沙箱 containerd 集成。 |
| 本地编码 Agent 沙箱 | [Cleanroom](https://github.com/buildkite/cleanroom)、[Brood Box](https://github.com/stacklok/brood-box)、[microsandbox](https://github.com/superradcompany/microsandbox)、[BoxLite](https://github.com/boxlite-ai/boxlite)、[Matchlock](https://github.com/jingkaihe/matchlock)、[Shuru](https://github.com/superhq-ai/shuru)、[sandbox-runtime](https://github.com/anthropic-experimental/sandbox-runtime) | 针对开发者工作站单独评估，其中仓库范围的出站、主机端凭证和运行后差异审查比 K8s CRD 更重要。 |

推荐的 Kubernetes 形态：

1. 将 Agent API、认证、计费、策略和可观测性作为常规 Pod 运行。
2. 通过 `Sandbox` CRD 或等效的平台 API 分配执行沙箱。
3. 按风险选择 `RuntimeClass`：`runsc`/gVisor 用于密集的仅 CPU 任务，Kata 用于高风险或支持 GPU 的租户，专用 VM/microVM 池用于合规性要求高的工作负载。
4. 保持沙箱零密钥和短生命周期；通过控制平面代理路由 LLM、存储和外部 API 凭证。

#### 基于容器的沙箱

容器运行命名空间 + cgroup 设置和进程初始化，在 50ms 内启动（排除镜像拉取）。虽然快速，但它们共享主机内核，这是不可信 Agent 代码的主要安全问题。

#### 基于 Firecracker 的沙箱（e2b）

<a href="https://github.com/e2b-dev/e2b">`e2b`</a> 是一个广泛采用的开源 Agent 沙箱，使用 Firecracker microVM。关键设计选择：

- **模板 = 快照**：沙箱镜像是序列化的运行中 VM（文件系统 + 所有进程）。任何新沙箱都可以在 1 秒内从该快照恢复（Intel: <8ms, AMD: <3ms），大幅减少有效冷启动时间
- **镜像构建**：将 Dockerfile 转换为 ext4 rootfs——一个多步骤过程，涉及镜像提取、在 VM 内配置，然后快照
- **调度**：使用轻量级的 best-of-k 调度器（非 Kubernetes）。基于 CPU、内存和活跃沙箱计数选择 k 个负载最轻的节点——非常适合 Agent 工作负载的短生命周期、高并发特性

```python
from e2b_code_interpreter import Sandbox

sbx = Sandbox.create()
sbx.beta_pause()          # 序列化为快照

same_sbx = sbx.connect()  # 从快照恢复，<1s
```

#### 基于 Kata Containers 的沙箱（k7）

<a href="https://github.com/Katakate/k7">`k7`</a> 和类似的基于 Kata 的方法使用 Kata Containers（配合 Firecracker VMM）获得强 VM 隔离，同时保持与标准 OCI 容器镜像的兼容性。

- **镜像构建**：仅需常规 Dockerfile——无需 ext4 转换
- **调度**：原生集成 Kubernetes 或 k3s
- **权衡**：Kata 的抽象层阻止直接使用 Firecracker 快照，因此恢复速度慢于 e2b

Kata 更适合已经使用 Kubernetes 的团队，他们重视 OCI 兼容性而非基于快照的快速恢复。

#### WASM/Python 子集（monty）

<a href="https://github.com/pydantic/monty">`monty`</a> 是一个基于 WASM 的 Python 子集解释器，冷启动时间为 0.06ms。但是，它不支持 Python 的 `class` 关键字、`sys` 模块或大多数标准库功能，使其不适合一般 Agent 工作负载。Pyodide（WASM 中的完整 CPython）是替代方案，但初始加载时间非常慢。

#### Unikernel（Unikraft）

Unikernel 将应用和最小内核编译为在硬件虚拟化上运行的单个镜像。它们提供：

- 最小攻击面（无未使用的内核功能，无 shell）
- 毫秒级启动
- 非常小的镜像大小

<a href="https://unikraft.org/">`Unikraft`</a> 在 v0.19（2025 年 5 月）中添加了有限的多进程支持（通过 `vfork`+`execve`），这是 Python 兼容性的先决条件。Browser Use 在生产中使用 Unikraft microVM 进行其模式 2 沙箱部署。

Unikernel 的主要剩余挑战是 Python 生态兼容性——多进程支持仍在成熟中。随着生态的发展，Unikernel 代表了未来的强选项：接近 VM 的安全性和非常小的 footprint。

参考：<a href="https://gaocegege.com/Blog/genai/unikernel-agent">Agent sandbox 可能的选型以及 unikernel 的机会</a> (2026) by 高策 (gaocegege)

### 镜像构建和分发优化

对于大型 Agent 镜像，镜像拉取通常是主要的延迟来源——远大于容器/VM 启动本身。传统的 Docker 拉取是顺序的：下载 N 个 gzip 层（约 2 GiB/s），单线程解压（约 80 MiB/s），解包到文件系统。一个 8 GB 的镜像可能需要超过一分钟。

<a href="https://modal.com">Modal</a> 通过 **基于 FUSE 的延迟加载** 解决此问题：从镜像元数据生成占位文件系统树，然后仅在访问时获取文件数据，遵循优先级链：

1. 内存缓存
2. 本地 SSD
3. 同区域缓存服务器
4. 区域 CDN
5. 对象存储

类似的方法包括 <a href="https://github.com/containerd/stargz-snapshotter">estargz</a> 和 <a href="https://github.com/dragonflydb/dragonfly">Dragonfly</a>，它们使用块级内容可寻址存储启用延迟加载。

### Kubernetes SIG Agent Sandbox

<a href="https://github.com/kubernetes-sigs/agent-sandbox">Agent Sandbox</a> 是一个 Kubernetes SIG 项目，探索 AI Agent 和自主工作负载的安全执行环境。

#### 设计原则

1. **多层隔离**：结合 namespace、cgroup、seccomp、AppArmor
2. **资源配额**：强制执行严格的 CPU、内存、网络限制
3. **时间限制**：自动终止长时间运行的 Agent
4. **审计日志**：全面追踪 Agent 动作

#### 示例用例

- LLM Agent 的代码解释器（例如 ChatGPT Code Interpreter）
- 自主工作流执行（例如函数调用）
- 动态模型部署和测试
- 安全执行用户提供的推理代码

### GKE Agent Sandbox

<a href="https://cloud.google.com/blog/products/containers-kubernetes/agentic-ai-on-kubernetes-and-gke">GKE Agent Sandbox</a> 为 Kubernetes 和 GKE 上的 Agent AI 提供生产就绪的强护栏，结合 gVisor 沙箱和高级快照能力。

#### 主要特性

- **gVisor 集成**：用户空间内核，用于不可信 Agent 代码的强隔离
- **容器快照**：预预热的容器镜像，包含初始化的依赖和运行时状态
- **快速冷启动**：相比传统容器启动，冷启动性能提升高达 90%
- **安全边界**：多租户 AI Agent 工作负载的硬件级隔离

#### 性能优势

- **减少延迟**：冷启动快 90%，实现近乎即时的 Agent 执行
- **资源效率**：快照重用减少冗余初始化
- **可伸缩性**：Agent 工作负载的快速扩展，适用于高并发场景
- **成本优化**：更快的启动意味着更高效的资源利用

### 项目和资源

- <a href="https://github.com/kubernetes-sigs/agent-sandbox">Kubernetes SIG Agent Sandbox</a>
- <a href="https://github.com/alibaba/OpenSandbox">OpenSandbox</a>：通用自托管沙箱 API/SDK，支持 Docker、Kubernetes 和安全运行时后端
- <a href="https://github.com/TencentCloud/CubeSandbox">CubeSandbox</a>：兼容 E2B 的 KVM microVM 沙箱服务，包含 Cube agent/runtime 组件
- <a href="https://github.com/e2b-dev/e2b">e2b</a>：开源 Firecracker Agent 沙箱，支持基于快照的快速恢复
- <a href="https://github.com/daytonaio/daytona">Daytona</a>：Agent 代码执行平台，支持快照、SDK、仪表板和 runner 模型
- <a href="https://github.com/Katakate/k7">k7</a>：基于 Kata Containers 的 Agent 沙箱，支持 OCI 镜像
- <a href="https://github.com/containerd/nerdbox">containerd/nerdbox</a>：实验性 containerd VM 沙箱运行时 shim
- <a href="https://github.com/buildkite/cleanroom">Cleanroom</a> 和 <a href="https://github.com/stacklok/brood-box">Brood Box</a>：本地编码 Agent microVM 沙箱，专注于仓库策略、出站、凭证和变更审查
- <a href="https://unikraft.org/">Unikraft</a>：Unikernel 框架，支持多进程（v0.19+）
- <a href="https://github.com/pydantic/monty">monty</a>：WASM Python 子集解释器，用于超快冷启动
- <a href="https://cloud.google.com/blog/products/containers-kubernetes/agentic-ai-on-kubernetes-and-gke">GKE Agent Sandbox：Agent AI 的强护栏</a>
- <a href="https://browser-use.com/posts/two-ways-to-sandbox-agents">Browser Use：构建安全、可扩展的 Agent 沙箱基础设施</a>
- <a href="https://gaocegege.com/Blog/genai/unikernel-agent">高策：Agent sandbox 可能的选型以及 unikernel 的机会</a>

## 7. Checkpoint 和 Restore

Checkpoint/Restore 允许保存和恢复运行中容器的完整状态，实现工作负载迁移、容错和高效资源利用。这对使用 GPU 的 AI 工作负载尤其重要，checkpoint/restore 可以实现长时间运行训练作业的高效资源利用和容错。

### Kubernetes 工作组

Kubernetes 社区有活跃的工作组专注于 checkpoint/restore 能力：

- <a href="https://github.com/kubernetes/community/blob/master/wg-batch/README.md">Kubernetes WG Batch</a>：协调批处理工作负载功能，包括训练作业的 checkpoint/restore
- <a href="https://kccnceu2025.sched.com/event/1tx7i">KubeCon EU 2025：Kubernetes 中的 Checkpoint/Restore</a>：关于 GPU checkpoint/restore 场景的社区讨论

### CRIU（Checkpoint/Restore In Userspace）

<a href="https://github.com/checkpoint-restore/criu">CRIU</a> 是 checkpoint Linux 进程的核心技术。

#### 工作原理

1. **Checkpoint**：冻结进程，转储内存、文件描述符和状态
2. **存储**：将状态序列化到文件系统或对象存储
3. **Restore**：在相同/不同主机上用完全相同的状态重建进程

#### GPU 插件架构

CRIU 通过插件架构支持 GPU checkpoint/restore：

- <a href="https://github.com/checkpoint-restore/criu/tree/criu-dev/plugins">CRIU 插件</a>：可扩展的插件系统，用于设备特定的 checkpoint
- **CUDA 插件**：处理 NVIDIA GPU 状态和 CUDA 上下文
- **AMD GPU 插件**：通过 AMDGPU 驱动处理 AMD GPU 状态

### GPU Checkpoint/Restore 支持

GPU checkpoint/restore 对于 AI 工作负载至关重要，可为长时间运行的训练作业实现高效资源利用和容错。

#### NVIDIA CUDA Checkpoint

<a href="https://github.com/NVIDIA/cuda-checkpoint">NVIDIA cuda-checkpoint</a> 是一个用于 checkpoint CUDA 应用的实用程序，构建在 CRIU 之上。

**主要特性：**

- **CUDA 上下文保存**：保存 GPU 内存、CUDA 流和上下文
- **驱动兼容性**：需要 CUDA 驱动版本 550 或更高
- **应用透明性**：checkpoint 所需代码更改最小
- **多 GPU 支持**：处理使用多个 GPU 的应用

**要求：**

- NVIDIA GPU 驱动 550+
- 支持 CUDA 插件的 CRIU
- 兼容的容器运行时（containerd、CRI-O）

**AI 使用场景：**

- **训练 Checkpoints**：跨抢占保存训练状态
- **模型迁移**：在节点间移动运行中的推理工作负载
- **容错**：硬件故障后恢复训练
- **成本优化**：迁移到 spot/可抢占 GPU 实例

#### PyTorch 支持

PyTorch 应用可以从 GPU checkpoint/restore 中受益：

- <a href="https://developer.nvidia.com/blog/checkpointing-cuda-applications-with-criu/">NVIDIA 博客：使用 CRIU 进行 CUDA 应用 Checkpointing</a>
- <a href="https://github.com/NVIDIA/cuda-checkpoint/issues/4">PyTorch 支持讨论</a>

**集成模式：**

```python
import torch

def train_with_checkpointing():
    model = MyModel().cuda()
    optimizer = torch.optim.Adam(model.parameters())

    for epoch in range(num_epochs):
        loss = train_epoch(model, optimizer)

        torch.save({
            'epoch': epoch,
            'model_state': model.state_dict(),
            'optimizer_state': optimizer.state_dict(),
        }, f'checkpoint_{epoch}.pt')
```

#### AMD GPU 支持

AMD GPU checkpoint/restore 通过 CRIU 插件支持：

- **AMDGPU 插件**：CRIU 插件生态的一部分
- **ROCm 兼容性**：与 AMD ROCm 栈配合使用
- **驱动要求**：需要兼容的 AMDGPU 驱动版本

**CUDA + AMDGPU 组合支持：**

CRIU 插件架构支持异构 GPU 环境的 checkpoint/restore：

- NVIDIA GPU 通过 cuda-checkpoint
- AMD GPU 通过 AMDGPU 插件
- 同一 Kubernetes 集群中的混合 GPU 部署

### Kubernetes 集成

Kubernetes 中的 Checkpoint/Restore 支持：

- **取证容器 Checkpointing（KEP-2008）**：v1.25 中 Alpha
- **CRI 支持**：容器运行时必须实现 checkpoint API
- **CSI 集成**：将 checkpoints 存储在持久卷中

#### 使用示例

```bash
crictl checkpoint <container-id> <checkpoint-name>
crictl restore <checkpoint-name> <pod-id>
```

### AI 工作负载优势

#### 训练 Checkpoints

- **容错**：节点故障后恢复训练
- **成本优化**：迁移到 spot/可抢占实例
- **实验**：保存/恢复模型状态用于 A/B 测试

#### 推理 Checkpoints

- **快速预热**：预预热模型缓存和 GPU 状态
- **迁移**：跨区域移动推理工作负载
- **伸缩**：使用 checkpointed 副本快速扩展

### 实现考量

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: checkpointable-training
  annotations:
    checkpoint.kubernetes.io/enabled: "true"
spec:
  restartPolicy: Never
  containers:
  - name: trainer
    image: pytorch/pytorch:latest
    volumeMounts:
    - name: checkpoint-storage
      mountPath: /checkpoints
  volumes:
  - name: checkpoint-storage
    persistentVolumeClaim:
      claimName: training-checkpoints
```

### GPU 工作负载的挑战

GPU checkpoint/restore 带来了独特的挑战，但最近的进展使其更加实用：

- **GPU 状态复杂性**：CUDA 上下文、设备内存和驱动状态
  - **解决方案**：NVIDIA cuda-checkpoint（驱动 550+）和 CRIU 插件自动处理
- **驱动兼容性**：恢复需要相同的 GPU 驱动版本
  - **缓解方案**：固定驱动版本的容器镜像，调度器中的驱动版本匹配
- **大状态大小**：数十亿参数的模型创建大型 checkpoint 文件
  - **优化**：增量 checkpoint、压缩和快速存储（NVMe、对象存储）
- **跨厂商支持**：NVIDIA 与 AMD 的不同 checkpoint 格式
  - **进展**：CRIU 插件架构为 NVIDIA（cuda-checkpoint）和 AMD（AMDGPU 插件）提供统一接口

### 项目和资源

- <a href="https://criu.org/">CRIU 官方网站</a>
- <a href="https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/2008-forensic-container-checkpointing">Kubernetes KEP-2008：取证容器 Checkpointing</a>
- <a href="https://github.com/NVIDIA/cuda-checkpoint">NVIDIA cuda-checkpoint</a>：CUDA 应用的 GPU checkpoint/restore 工具（驱动 550+）
- <a href="https://github.com/checkpoint-restore/criu/tree/criu-dev/plugins">CRIU 插件</a>：支持 CUDA 和 AMDGPU 的插件架构
- <a href="https://developer.nvidia.com/blog/checkpointing-cuda-applications-with-criu/">NVIDIA 博客：使用 CRIU 进行 CUDA 应用 Checkpointing</a>
- <a href="https://github.com/NVIDIA/cuda-checkpoint/issues/4">PyTorch Checkpoint/Restore 支持</a>
- <a href="https://kccnceu2025.sched.com/event/1tx7i">KubeCon EU 2025：Kubernetes 中的 Checkpoint/Restore</a>
- <a href="https://github.com/kubernetes/community/blob/master/wg-batch/README.md">Kubernetes WG Batch</a>

## AI 工作负载隔离最佳实践

### 多层防御

组合多种隔离机制：

1. **基础层**：cgroups + namespace 隔离
2. **安全层**：SecurityContext + Pod 安全标准
3. **增强层**：用户命名空间 + rootless（适用时）
4. **强隔离层**：基于 VM 的隔离（Kata）用于不可信代码

### 工作负载分类

| 工作负载类型 | 隔离级别 | 推荐方法 |
|---------------|----------------|---------------------|
| 可信内部训练 | 标准 | cgroups + SecurityContext |
| 多租户推理 | 增强 | + 用户命名空间 |
| 外部用户代码 | 强 | Kata Containers + Agent Sandbox |
| 敏感数据处理 | 最高 | Kata + 加密 + 网络策略 |

### 性能与安全权衡

- **训练**：优先考虑性能，接受标准隔离
- **推理**：平衡延迟要求与安全需求
- **Agent 执行**：最大化安全性，接受性能开销

## 监控和可观测性

跟踪隔离效果：

- **资源使用**：监控 cgroup 指标以确保限制执行
- **安全事件**：审计日志，检测权限提升尝试
- **性能影响**：测量隔离机制的开销
- **隔离违规**：对命名空间或能力违规发出告警

### 工具

- <a href="https://github.com/falcosecurity/falco">Falco</a>：运行时安全监控
- <a href="https://github.com/aquasecurity/tracee">Tracee</a>：基于 eBPF 的安全可观测性
- <a href="https://github.com/cilium/tetragon">Tetragon</a>：基于 eBPF 的安全可观测性和执行

## 未来方向

### 新兴技术

- **机密计算**：SGX、SEV、TDX 用于加密内存
- **WebAssembly**：Agent 代码的轻量级沙箱
- **基于 eBPF 的隔离**：细粒度的系统调用过滤和执行
- **AI 特定隔离**：模型水印、推理时安全

### Kubernetes 增强

- **成熟的用户命名空间**：更广泛的运行时支持和稳定性
- **GPU Checkpoint/Restore**：GPU 状态迁移的生产就绪支持，使用 NVIDIA cuda-checkpoint（驱动 550+）和 AMD AMDGPU 插件集成
- **Agent 安全策略**：为 AI Agent 定制的原语
- **零信任网络**：AI 工作负载的服务网格集成

## 参考

- <a href="https://kubernetes.io/docs/concepts/security/">Kubernetes 安全文档</a>
- <a href="https://github.com/kubernetes/community/blob/master/wg-batch/README.md">Kubernetes WG Batch</a>
- <a href="https://github.com/kubernetes/community/blob/master/wg-device-management/README.md">Kubernetes WG Device Management</a>
- <a href="https://www.cncf.io/blog/2023/12/15/container-isolation-gone-wrong/">CNCF：容器隔离出错</a>
- <a href="https://kubernetes.io/blog/2023/08/14/forensic-container-analysis/">Kubernetes 博客：取证容器 Checkpointing</a>

---

*本文档涵盖了云原生环境中 AI 工作负载的隔离机制。随着技术的发展，最佳实践和工具将继续成熟。*
