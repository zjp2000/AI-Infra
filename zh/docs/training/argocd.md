---
status: Active
maintainer: pacoxu
last_updated: 2025-10-29
tags: training, argocd, gitops, continuous-deployment
canonical_path: zh/docs/training/argocd.md
---

# ArgoCD：AI 工作负载管理

## 概述

<a href="https://github.com/argoproj/argo-cd">`ArgoCD`</a> 是一个 CNCF Graduated 项目，为 Kubernetes 提供声明式 GitOps 持续交付。在 AI 基础设施的背景下，ArgoCD 使团队能够使用 Git 作为单一事实来源来管理、版本化和部署 ML 训练任务、推理服务和数据管道。

## 为什么选择 ArgoCD 用于 AI 工作负载？

### 关键优势

1. **GitOps 工作流**
   - ML 任务配置的版本控制
   - 所有变更的审计跟踪
   - 轻松回滚到以前的版本
   - 通过 pull request 进行协作审查

2. **声明式配置**
   - 将训练任务、模型和基础架构定义为代码
   - 跨环境的可复现部署
   - 一致的配置管理

3. **多环境支持**
   - 管理开发、预发布和生产环境
   - 新模型的渐进式发布
   - 环境特定的配置

4. **自动化和自愈**
   - 与 Git 仓库自动同步
   - 漂移检测的自愈
   - 自动化部署管道

## ArgoCD 在 Kubeflow 生态系统中的位置

ArgoCD 通过为 ML 工作负载提供持续部署能力来补充 Kubeflow。如 <a href="https://mp.weixin.qq.com/s/qFmMduuQtgOdjwQLaT95dw">Kubeflow 生态系统架构</a>（中文资源）中所示，并在 <a href="https://argo-cd.readthedocs.io/">官方 ArgoCD 文档</a> 中引用，ArgoCD 与各种 Kubeflow 组件集成：

![Kubeflow Ecosystem](https://github.com/user-attachments/assets/bc87980f-37d8-452f-ae1d-94cadc53a8f1)

### 集成点

**Kubeflow 组件：**

- Jupyter Notebook 部署
- Training Operator（PyTorch、TensorFlow、XGBoost）
- Kubeflow Pipelines
- Tensorboard 可视化
- Katib 超参数调优任务
- KServe 模型服务部署
- MPI Operator 分布式训练
- Kubeflow Metadata 跟踪
- 周期任务调度

**基础设施组件：**

- Istio 服务网格配置
- Knative serverless 函数
- cert-manager 证书
- MinIO 对象存储
- Dex 身份管理

## ML 工作流的 ArgoCD 架构

### 组件

1. **Git 仓库**：所有配置的事实来源
   - 训练任务定义
   - 模型服务配置
   - 管道定义
   - 基础设施清单

2. **ArgoCD Application**：定义部署策略
   - 源仓库和分支
   - 目标 Kubernetes 命名空间
   - 同步策略（手动或自动）
   - 健康检查和状态

3. **Kubernetes 集群**：目标部署环境
   - 训练任务（PyTorchJob、TFJob）
   - 推理服务（KServe）
   - ML 管道（Kubeflow Pipelines）

## 训练任务的 GitOps 工作流

### 仓库结构

```text
ml-training-repo/
├── base/
│   ├── kustomization.yaml
│   └── pytorch-job-template.yaml
├── overlays/
│   ├── dev/
│   │   ├── kustomization.yaml
│   │   └── patches.yaml
│   ├── staging/
│   │   ├── kustomization.yaml
│   │   └── patches.yaml
│   └── prod/
│       ├── kustomization.yaml
│       └── patches.yaml
└── argocd/
    ├── training-dev.yaml
    ├── training-staging.yaml
    └── training-prod.yaml
```

### 示例：Git 中的 PyTorch 训练任务

```yaml
# base/pytorch-job-template.yaml
apiVersion: kubeflow.org/v1
kind: PyTorchJob
metadata:
  name: pytorch-distributed-training
spec:
  pytorchReplicaSpecs:
    Master:
      replicas: 1
      restartPolicy: OnFailure
      template:
        spec:
          containers:
          - name: pytorch
            image: pytorch/pytorch:2.0.0-cuda11.7-cudnn8-runtime
            command:
              - python
              - train.py
            resources:
              limits:
                nvidia.com/gpu: 1
    Worker:
      replicas: 3
      restartPolicy: OnFailure
      template:
        spec:
          containers:
          - name: pytorch
            image: pytorch/pytorch:2.0.0-cuda11.7-cudnn8-runtime
            command:
              - python
              - train.py
            resources:
              limits:
                nvidia.com/gpu: 1
```

### ArgoCD Application 定义

```yaml
# argocd/training-prod.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ml-training-prod
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/ml-training-repo
    targetRevision: main
    path: overlays/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: ml-training
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

## 使用 ArgoCD 管理训练任务

### 部署工作流

1. **定义训练任务**：在 Git 中创建 PyTorchJob 或 TFJob 清单
2. **提交到仓库**：推送更改到特性分支
3. **代码审查**：团队通过 pull request 审查任务配置
4. **合并到主分支**：批准的更改合并到 main 分支
5. **ArgoCD 同步**：ArgoCD 检测更改并同步到集群
6. **任务执行**：Kubernetes 创建 pod 并运行训练
7. **监控**：在 ArgoCD UI 和 Kubeflow 仪表板中跟踪任务状态

### 渐进式发布策略

```yaml
# 使用 ArgoCD sync waves 进行渐进式发布
apiVersion: kubeflow.org/v1
kind: PyTorchJob
metadata:
  name: llm-training-v2
  annotations:
    argocd.argoproj.io/sync-wave: "1"  # 在基础设施之后部署
spec:
  # 任务配置
```

### 处理任务更新

#### 场景：更新训练超参数

```bash
# 1. 克隆仓库
git clone https://github.com/your-org/ml-training-repo
cd ml-training-repo

# 2. 创建特性分支
git checkout -b update-learning-rate

# 3. 更新任务配置
vim overlays/prod/patches.yaml
# 将学习率从 0.001 改为 0.0005

# 4. 提交并推送
git add overlays/prod/patches.yaml
git commit -m "Update learning rate to 0.0005"
git push origin update-learning-rate

# 5. 创建 pull request，审查后合并

# 6. ArgoCD 自动同步更改
```

## 与 CI/CD 管道的集成

### GitHub Actions + ArgoCD

```yaml
# .github/workflows/deploy-training.yaml
name: Deploy Training Job
on:
  push:
    branches: [main]
    paths:
      - 'overlays/prod/**'

jobs:
  validate-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Validate Kubernetes manifests
        run: |
          kubectl --dry-run=client apply -f overlays/prod/

      - name: Trigger ArgoCD Sync
        env:
          ARGOCD_SERVER: ${{ secrets.ARGOCD_SERVER }}
          ARGOCD_TOKEN: ${{ secrets.ARGOCD_TOKEN }}
        run: |
          argocd app sync ml-training-prod --grpc-web
```

### 预同步钩子用于数据准备

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: prepare-training-data
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
      - name: data-prep
        image: python:3.11
        command:
          - python
          - prepare_data.py
      restartPolicy: Never
```

## 多集群训练管理

### 场景：跨区域训练

ArgoCD 支持将训练任务部署到多个 Kubernetes 集群：

```yaml
# argocd/training-us-west.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ml-training-us-west
spec:
  destination:
    server: https://us-west-k8s.example.com
    namespace: ml-training
  source:
    repoURL: https://github.com/your-org/ml-training-repo
    path: overlays/us-west

---
# argocd/training-eu-central.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ml-training-eu-central
spec:
  destination:
    server: https://eu-central-k8s.example.com
    namespace: ml-training
  source:
    repoURL: https://github.com/your-org/ml-training-repo
    path: overlays/eu-central
```

## 监控和可观测性

### ArgoCD Application 健康状态

ArgoCD 为训练任务提供健康状态：

- **Healthy**：所有 pod 成功运行
- **Progressing**：任务正在部署或运行中
- **Degraded**：部分 pod 失败或任务遇到错误
- **Suspended**：任务暂停或等待资源
- **Missing**：在集群中未找到任务定义

### 自定义健康检查

```yaml
# PyTorchJob 的自定义健康检查
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: pytorch-training
spec:
  # ... 其他配置
  ignoreDifferences:
  - group: kubeflow.org
    kind: PyTorchJob
    jsonPointers:
    - /status
  # 自定义健康评估
  health:
    status: Progressing
    message: "Training job is running"
```

### 与 Prometheus 的集成

监控 ML 工作负载部署的 ArgoCD 指标：

```promql
# 不同步的 ML 训练应用数量
argocd_app_info{namespace="ml-training",sync_status="OutOfSync"}

# 部署频率
rate(argocd_app_sync_total{namespace="ml-training"}[1h])

# 失败的同步
argocd_app_sync_total{namespace="ml-training",phase="Failed"}
```

## AI 工作负载的最佳实践

### 1. 将配置与代码分离

- 将训练脚本存储在单独的仓库中
- 使用容器镜像来管理代码
- 在 ArgoCD 跟踪的仓库中管理任务配置

### 2. 环境特定配置

```text
overlays/
├── dev/          # 小数据集，单 GPU，频繁更新
├── staging/      # 中等数据集，4 GPU，预生产验证
└── prod/         # 完整数据集，16+ GPU，稳定发布
```

### 3. 版本控制策略

- 对训练任务定义使用语义化版本
- 为生产部署标记发布版本
- 为实验维护单独的分支

### 4. Secrets 管理

```yaml
# 使用 sealed secrets 或外部 secret operator
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: training-secrets
spec:
  encryptedData:
    api-key: AgBhQ3J...  # 加密的 API 密钥
```

### 5. 资源配额和限制

定义资源约束以防止失控任务：

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: ml-training-quota
spec:
  hard:
    requests.nvidia.com/gpu: "32"
    requests.cpu: "256"
    requests.memory: "1024Gi"
```

## 故障排除

### 常见问题

#### 1. Application 不同步

```bash
# 检查差异
argocd app diff ml-training-prod

# 强制同步
argocd app sync ml-training-prod --force
```

#### 2. 训练任务卡在 Pending 状态

```bash
# 检查 ArgoCD 日志
argocd app logs ml-training-prod

# 检查 Kubernetes 事件
kubectl describe pytorchjob pytorch-training -n ml-training
```

#### 3. 同步钩子失败

```bash
# 查看钩子日志
argocd app logs ml-training-prod --kind Job
```

## 与其他 CD 工具的比较

### ArgoCD vs Flux CD

- **ArgoCD**：Web UI、多租户、用于多集群的 ApplicationSet
- **Flux CD**：更轻量、原生 Kubernetes、HelmRelease CRD

### ArgoCD vs Jenkins

- **ArgoCD**：GitOps、声明式、Kubernetes 原生
- **Jenkins**：命令式、通用 CI/CD，需要 K8s 插件

### ArgoCD vs Kubeflow Pipelines

- **ArgoCD**：部署和生命周期管理
- **Kubeflow Pipelines**：ML 工作流编排和实验管理
- **一起使用**：ArgoCD 部署 Pipelines，Pipelines 编排训练

## 学习资源

### 官方文档

- <a href="https://argo-cd.readthedocs.io/">ArgoCD Documentation</a>
- <a href="https://github.com/argoproj/argo-cd">ArgoCD GitHub Repository</a>
- <a href="https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/">ArgoCD Best Practices</a>

### 社区资源

- <a href="https://mp.weixin.qq.com/s/qFmMduuQtgOdjwQLaT95dw">Kubeflow Ecosystem with ArgoCD</a>（中文）
- <a href="https://www.cncf.io/blog/2023/01/12/argo-cd-best-practices/">CNCF ArgoCD Blog</a>
- <a href="https://argoproj.github.io/community/">ArgoCD Community</a>

### 教程

- <a href="https://argo-cd.readthedocs.io/en/stable/getting_started/">Getting Started with ArgoCD</a>
- <a href="https://codefresh.io/learn/argo-cd/">ArgoCD Tutorial by Codefresh</a>

## Argo 生态中的相关项目

- <a href="https://github.com/argoproj/argo-workflows">`Argo Workflows`</a>：Kubernetes 原生工作流引擎（CNCF Graduated）
- <a href="https://github.com/argoproj/argo-rollouts">`Argo Rollouts`</a>：Kubernetes 的渐进式交付
- <a href="https://github.com/argoproj/argo-events">`Argo Events`</a>：事件驱动的工作流自动化

## 路线图和未来发展

### 即将推出的功能

- 增强的 ApplicationSet，支持动态应用生成
- 改进的 Helm charts 和 Kustomize 支持
- 与云原生安全工具的更好集成
- 高级部署策略（蓝绿、金丝雀）

## 贡献

欢迎对 ArgoCD 做出贡献！详见 <a href="https://github.com/argoproj/argo-cd/blob/master/CONTRIBUTING.md">贡献指南</a>。

---

**注意：** 部分内容可能根据引用来源生成或总结。在生产环境中使用前，请以官方文档为准验证技术细节。
