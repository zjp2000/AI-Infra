---
status: Active
maintainer: pacoxu
last_updated: 2026-05-09
tags: kubernetes, dra, drivers, ecosystem, feature-matrix
canonical_path: zh/docs/kubernetes/dra-driver-feature-matrix.md
source_urls: |-
  https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/
  https://kubernetes.io/blog/2026/05/07/kubernetes-v1-36-dra-136-updates/
  https://github.com/kubernetes-sigs/dra-driver-nvidia-gpu
  https://github.com/ROCm/k8s-gpu-dra-driver
  https://github.com/kubernetes-sigs/dra-driver-cpu
  https://github.com/kubernetes-sigs/dra-driver-google-tpu
  https://github.com/Project-HAMi/k8s-dra-driver
  https://github.com/Project-HAMi/enflame-dra-driver
  https://github.com/kubernetes-sigs/cni-dra-driver
  https://github.com/kubernetes-sigs/dranet
  https://github.com/k8snetworkplumbingwg/dra-driver-sriov
  https://github.com/IBM/power-dra-driver
  https://github.com/ibm-aiu/dra-driver-spyre
  https://github.com/RBLN-SW/rbln-k8s-dra-driver
  https://github.com/kubevirt/dra-pci-driver
  https://github.com/DiamondLightSource/dra-usbip-driver
  https://github.com/ffromani/dra-driver-memory
  https://github.com/ffromani/dra-driver-cpumem
  https://github.com/fabiendupont/dra-driver-hugepages
  https://github.com/fabiendupont/k8s-dra-driver-time-share
  https://github.com/justin-oleary/cxl-dra-driver
  https://github.com/abhinavdahiya/k8s-dra-driver-trace-collector
  https://github.com/mewz-project/waiot-dra-driver
---

# 公共 Kubernetes DRA 驱动功能矩阵

本文档将**公开的 GitHub DRA 实现**映射到一组**高信号 DRA 功能**上。目标不是重复说明每个项目是"一个 DRA 驱动"，而是回答一个更实际的问题：

**哪些驱动在公开层面显示了它们已经使用了特定 DRA 功能的证据，哪些仍在进行中，哪些还没有可见的 issue/PR 信号？**

## 范围

截至 **2026 年 5 月 9 日**，本矩阵仅审计用户**已确认的公开实现**列表。更具实验性/描述稀疏的仓库在末尾单独列出，不纳入主矩阵评分。

## 追踪的功能

本文档关注 Beta/GA 功能以及一个较旧但在运营上重要的 Beta 功能：

- `PL`：优先级列表（v1.36，Stable）
- `ER`：扩展资源支持（v1.36，Beta）
- `PD`：可分区设备（v1.36，Beta）
- `DT`：设备污点和容忍度（v1.36，Beta）
- `BC`：设备绑定条件（v1.36，Beta）
- `RHS`：资源健康状态（v1.36，Beta）
- `CC`：可消耗容量（v1.34+，Beta，但对共享很重要）

Alpha 功能（如工作负载级声明、节点可分配资源、资源池可见性、设备元数据和确定性设备选择）不包含在主评分表中。

## 阅读指南

- `Supported`：在代码、README、demo、测试或明确合并的 PR 中有公开的仓库信号。
- `WIP`：有明确的公开 issue 或 PR，或仅有较弱的局部信号而无明确的落地示例。
- `No visible issue/PR`：在上述追踪的功能中，未在该仓库中发现强的公开信号。

重要说明：

- 对于某些调度端功能（如 `PL`），驱动可能在**理论上兼容**而无需太多驱动特定代码。此处的"无可见信号"仅表示**未找到明确的公开仓库证据**，而非"无法使用"。

## 矩阵：已确认且有明确信号的驱动

| Repo | 资源类型 | 已支持 | WIP | 无可见 issue/PR | 示例证据 |
| --- | --- | --- | --- | --- | --- |
| [`kubernetes-sigs/dra-driver-nvidia-gpu`](https://github.com/kubernetes-sigs/dra-driver-nvidia-gpu) | NVIDIA GPU | `ER`, `PD`, `DT`, `RHS` | `BC` | `PL`, `CC` | `demo/specs/extended-resources/*`；`cmd/gpu-kubelet-plugin/partitions.go`；[PR #983](https://github.com/kubernetes-sigs/dra-driver-nvidia-gpu/pull/983)；[PR #689](https://github.com/kubernetes-sigs/dra-driver-nvidia-gpu/pull/689)；[PR #855](https://github.com/kubernetes-sigs/dra-driver-nvidia-gpu/pull/855) |
| `ROCm/k8s-gpu-dra-driver` | AMD GPU | `PD` | 无 | `PL`, `ER`, `DT`, `BC`, `RHS`, `CC` | PR `#33` 提及了不可分区 GPU |
| `kubernetes-sigs/dra-driver-cpu` | CPU | `CC` | 无 | `PL`, `ER`, `PD`, `DT`, `BC`, `RHS` | `README.md`；`pkg/driver/dra_hooks.go`；PR `#16` |
| `Project-HAMi/k8s-dra-driver` | HAMi GPU | `ER`, `PD`, `CC` | 无 | `PL`, `DT`, `BC`, `RHS` | `templates/deviceclass-hami-gpu.yaml`；`cmd/hami-kubelet-plugin/partitions.go`；`cmd/hami-kubelet-plugin/hami_core.go` |
| `kubernetes-sigs/cni-dra-driver` | CNI 网络 DRA | `CC` | 无 | `PL`, `ER`, `PD`, `DT`, `BC`, `RHS` | `pkg/discovery/resources.go`；`docs/demo/readme.md` |
| `k8snetworkplumbingwg/dra-driver-sriov` | SR-IOV 网络 | `ER` | 无 | `PL`, `PD`, `DT`, `BC`, `RHS`, `CC` | `demo/extended-resource/*`；PR `#75` |
| `RBLN-SW/rbln-k8s-dra-driver` | Rebellions NPU | `ER` | 无 | `PL`, `PD`, `DT`, `BC`, `RHS`, `CC` | `templates/deviceclass-npu.yaml` |
| `DiamondLightSource/dra-usbip-driver` | USB/IP 远程 USB | `PL` | 无 | `ER`, `PD`, `DT`, `BC`, `RHS`, `CC` | `docs/tutorials/quick-start.md` 包含 `firstAvailable` |
| `ffromani/dra-driver-memory` | memory / hugepages | `CC` | 无 | `PL`, `ER`, `PD`, `DT`, `BC`, `RHS` | `pkg/sysinfo/rslice.go`；`discover_amd64_test.go` |
| `IBM/power-dra-driver` | IBM Power 功能 | 无 | `PD` | `PL`, `ER`, `DT`, `BC`, `RHS`, `CC` | 已关闭的 issue `#26 Support DRAPartitionableDevices`，但尚无强 demo/example 信号 |

## 矩阵：已确认但在追踪功能中无高信号匹配的驱动

这些仓库可能仍然是有用的 DRA 实现，但未在上述追踪的 Beta/GA 功能集中找到强的公开证据。

| Repo | 资源类型 | 当前判断 |
| --- | --- | --- |
| [`kubernetes-sigs/dra-driver-google-tpu`](https://github.com/kubernetes-sigs/dra-driver-google-tpu) | Google TPU | 本轮未发现 `PL/ER/PD/DT/BC/RHS/CC` 的强公开信号。 |
| `Project-HAMi/enflame-dra-driver` | Enflame | 在追踪功能集中公开功能信号稀疏。 |
| `kubernetes-sigs/dranet` | network DRA | 强拓扑/网络定位，但本轮未在追踪功能集中找到高置信度的明确匹配。 |
| `ibm-aiu/dra-driver-spyre` | IBM AIU Spyre | 在追踪功能集中公开功能信号稀疏。 |
| `kubevirt/dra-pci-driver` | PCI passthrough | 尚未发现追踪功能集中的高信号公开证据。 |
| `ffromani/dra-driver-cpumem` | combined CPU/memory/hugepages | 可能与 `CC` 功能相关，但本轮未找到强明确信号。 |
| `fabiendupont/dra-driver-hugepages` | hugepages | 在追踪功能集中公开功能信号稀疏。 |
| `fabiendupont/k8s-dra-driver-time-share` | CPU time-sharing | 概念上接近 `CC`，但本轮未发现强的 `allowMultipleAllocations` 风格信号。 |
| `justin-oleary/cxl-dra-driver` | CXL pooled memory | 在追踪功能集中公开功能信号稀疏。 |
| `abhinavdahiya/k8s-dra-driver-trace-collector` | trace-processing capacity | DRA 生命周期信号存在，但本轮未找到与追踪的 Beta/GA 功能集的强明确匹配。 |
| `mewz-project/waiot-dra-driver` | Waiot | 在追踪功能集中公开功能信号稀疏。 |

## 值得指出的模式

### 1. `ER` 是当今最清晰的迁移信号

在公开仓库中，最清晰且最可复现的信号是**扩展资源支持**：

- `nvidia-gpu`
- `sriov`
- `HAMi`
- `RBLN`

这与让现有 `vendor.com/device: N` 工作负载逐步迁移到 DRA 的运维需求一致。

### 2. `PD` 集中在 GPU 类驱动中

最强的公开 `PD` 信号仍在 GPU 导向的仓库中：

- `nvidia-gpu`
- `ROCm`
- `HAMi`

`IBM/power-dra-driver` 有一个关于可分区设备的公开功能 issue，但其公开落地信号弱于 NVIDIA/HAMi/ROCm。

### 3. `CC` 在 CPU/内存/共享导向的驱动中最强

最清晰的公开 `CC` 信号来自：

- `dra-driver-cpu`
- `dra-driver-memory`
- `HAMi`
- `cni-dra-driver`

这与 DRA 共享语义在当今最自然的应用场景一致：CPU 池、内存切片、带宽类资源和细粒度多租户共享。

### 4. `DT`、`BC` 和 `RHS` 在公开仓库中仍然稀疏

唯一在所有三个运维功能上都有强公开信号的仓库是 `kubernetes-sigs/dra-driver-nvidia-gpu`：

- `DT`：已合并 PR `#983`
- `RHS`：已合并 PR `#689`
- `BC`：开放 PR `#855`

这并**不**意味着其他驱动无法采用这些功能，仅说明公开仓库证据仍然分布不均。

## 未纳入评分的实验性/描述稀疏仓库

以下仓库未包含在评分矩阵中，因为用户已将其区分为可能实验性/描述稀疏的仓库，它们的公开元数据需要更深入的手动审计：

- `CoHDI/composable-dra-driver`
- `converged-computing/fluxbind-dra-driver`
- `lengrongfu/dra-device-driver`
- `xrwang8/ascend-dra-driver`
- `johnahull/dra-driver-nvme`
- `TheRealSibasishBehera/kubevirt-dra-driver`
- `pravk03/simple-cpu-dra-driver`
- `Tal-or/dra-cpu-driver`
- `gke-labs/dra-drivers`
- `ndbaker1/runtime-spec-dra-driver`
- `rotatingvio/cni-dra-driver`
- `toVersus/fake-dra-driver`
- `nojnhuh/dra-noop-driver`
- `nojnhuh/dra-driver-sandbox`
- `13567436138/k8s-dra-driver`

## 方法说明

本文档有意保持保守：

- 优先使用**公开仓库证据**而非推断。
- 将 **README/demo/测试/代码匹配**视为强于泛泛的 issue 正文匹配。
- 将**无明确示例/PR 的已关闭 issue**视为弱于已落地的 demo 或已合并的实现 PR。
- 在扫描过程中遇到了 GitHub Search API 速率限制，因此本矩阵应视为**高信号公开快照**，而非数学上的穷举证明。

如果您后续想扩展本文档，下一步有用的工作包括：

1. 为每个仓库添加 `last updated` 列。
2. 添加单独的 Alpha 功能附录。
3. 在稀疏仓库增加示例或设计文档后重新检查。
