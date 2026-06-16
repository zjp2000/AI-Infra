---
status: Active
maintainer: pacoxu
last_updated: 2025-10-29
tags: kubernetes, learning-path, docker, cloud-native
canonical_path: zh/docs/kubernetes/learning-plan.md
---

# 📘 Kubernetes 学习计划

本文档提供了一条结构化的 Kubernetes 学习路径。有关特定的 AI 基础设施相关主题，请参见：

- [Pod 启动速度优化](./pod-startup-speed.md)
- [工作负载隔离](./isolation.md)
- [Pod 生命周期](./pod-lifecycle.md)
- [调度优化](./scheduling-optimization.md)
- [动态资源分配（DRA）](./dra.md)

## 🔰 阶段 0 – 基础：Docker 和容器基础

在深入学习 Kubernetes 之前，理解容器的工作原理至关重要。

### ✅ 关键主题

- **Docker 基础**
  - 镜像、容器、卷
- **Docker 网络**
  - Bridge、Host、Overlay
- **容器 registry**
  - DockerHub、Harbor
- **容器编排（Docker 原生）**
  - `docker-compose`
  - Swarm 和 SwarmKit：集群管理、基于 Raft 的领导者选举

---

## ⚙️ 阶段 1 – Kubernetes 0.1 时代：K8s 入门

了解 Kubernetes 的早期架构和生态系统。

### ✅ 关键主题

- **基础网络**
  - CNI 插件：Calico、Flannel、Macvlan
  - Ingress、NodePort、外部负载均衡器（例如 F5）
- **存储**
  - 驱动：ScaleIO、Portworx
- **RBAC 和资源管理**
  - 基于角色的访问控制（RBAC）
  - ResourceQuota 和 LimitRange
  - 水平 Pod 自动伸缩（HPA）
- **可观测性**
  - Prometheus（监控）
  - EFK 栈（Elasticsearch + Fluentd + Kibana）
- **运行时**
  - Containerd

---

## 🚀 阶段 2 – Kubernetes 0.2 时代：工具与生态扩展

深入探索增强 Kubernetes 可用性和运维能力的工具与组件。

### ✅ 关键主题

- **CoreDNS** – 服务发现的 DNS
- **Knative** – Kubernetes 上的无服务器工作负载
- **启动工具**
  - `kubeadm`、`kind`
- **高级网络**
  - VLAN 集成
  - Istio 服务网格
  - Keepalived（高可用虚拟 IP）
- **集群工具**
  - Velero（备份/恢复）
  - KubeFed v2（联邦）
  - VPA（垂直 Pod 自动伸缩）
  - Thanos（Prometheus 长期存储）

---

## 🌐 阶段 3 – Kubernetes 0.3 时代：云原生生态

学习如何管理生产级、可扩展的多集群 Kubernetes 环境。此阶段包含更多生态项目，部分可能处于沙箱级别。

### ✅ 关键主题

- **集群部署**
  - Kubespray、Kops
- **下一代网络**
  - Cilium（eBPF）
  - Submariner（跨集群）
  - Whereabouts（IPAM）
  - MetalLB（裸金属负载均衡）
- **现代存储**
  - HwameiStor、本地持久卷
- **镜像管理和联邦**
  - Harbor（容器 registry）
  - Karmada（多集群编排）
- **认证和授权**
  - Dex、Pinniped（身份和访问管理）
- **可观测性和服务网格增强**
  - Clusterpedia（集群状态查询）
  - Merbridge（基于 eBPF 的 Istio 加速）
  - OpenTelemetry（追踪、指标、日志）

---

## 🛠 推荐资源

- [Kubernetes 官方文档](https://kubernetes.io/)
- [Awesome Kubernetes GitHub 列表](https://github.com/ramitsurana/awesome-kubernetes)
- 动手实践工具：
  - [Play with Kubernetes](https://labs.play-with-k8s.com/)
  - `kind`、`minikube`、`k3s`

---

Happy learning! 🚀

如需了解更多关于 Kubernetes 的内容，请参考 https://kubernetes.io/。以下是我个人的学习笔记。
