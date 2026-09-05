# Amazon EKS 多模型 GPU 共享与 SLO 驱动弹性推理优化方案

企业在 Amazon EKS 上部署多个生成式 AI 推理服务时，常采用“一模型一卡”。对于小模型、低频模型或流量错峰模型，这会造成 GPU 算力和显存长期闲置；但直接把多个模型放到同一 GPU，又可能引入延迟抖动、显存超配和故障域扩大。

本方案以 NVIDIA GPU Time-Slicing 为基础，将单张物理 GPU 暴露为多个 Kubernetes 可调度副本，使多个推理 Pod 共享同一 GPU 的计算时间；再结合显存预算、共享/独占 GPU 分池、队列与延迟驱动的扩缩容、模型加载加速和端到端可观测性，在满足业务 SLO 的前提下提高 GPU 利用率。

> 核心原则：Time-Slicing 不是“免费扩容”。它增加的是调度密度，不增加 GPU 物理算力，也不提供显存隔离。

> **本方案的 GPU 与方式**：压测和实施示例使用 EC2 G5 的 NVIDIA A10G，采用 **Time-Slicing**；A10G 不支持 MIG，本方案也没有使用 MIG。MIG 是仅适用于部分其他 GPU 的硬件分区能力，不能当作本方案的隔离能力。

## 适用场景

| 优先评估共享池 | 优先使用独占 GPU |
|---|---|
| 中小模型、低频、间歇或流量错峰 | 长期高并发、GPU 持续高利用率 |
| 可接受一定延迟波动 | TTFT、P95/P99 要求严格 |
| 模型权重和 KV Cache 有明确显存余量 | 显存接近上限或上下文不可预测 |
| 内部助手、低频 API、开发测试 | 核心业务、强隔离或高故障敏感业务 |

## 方案用途与收益

- 将多个低频模型放入共享 GPU 池，减少“一模型一卡”造成的空闲；
- 按业务 SLO 把关键模型保留在独占 GPU 池；
- 用显存预算和真实流量决定哪些模型可以配对共享；
- 用 Queue、TTFT、P95/P99 和 KV Cache 压力驱动 Pod 扩容；
- Pod Pending 时由 Karpenter 增加 GPU 节点；
- 通过 ECR、模型缓存和预热缩短扩容到首个成功请求的时间。

真正的收益不是“每张卡能启动更多 Pod”，而是：共享后仍满足 SLO，并降低 GPU 空闲率、GPU 小时或单位输出 Token 成本。

## 实施架构

```text
Web / Mobile / Internal / Batch Clients
                  │
                  ▼
           ALB / NLB Ingress
        TLS、认证、配额、限流
                  │
                  ▼
          Model Router + Queue
       按优先级、SLO、容量路由
          ┌───────┴────────┐
          ▼                ▼
 Shared GPU Pool      Dedicated GPU Pool
 NVIDIA Time-Slicing  严格 SLO / 强隔离模型
 低频、可降级模型      高优先级、长上下文模型

KEDA / HPA：根据队列和延迟扩缩 Pod
Karpenter：为 Pending Pod 增加 GPU 节点
Amazon ECR：推理镜像
S3 / Mountpoint / FSx：模型权重与缓存
DCGM Exporter + Prometheus/Grafana + CloudWatch：观测与告警
```

## Time-Slicing 的技术边界（也是本方案的适用条件）

配置 `replicas: 2` 后，一张物理 GPU 会向 Kubernetes 报告两个 `nvidia.com/gpu` 可调度副本，使两个各申请一个 GPU 的 Pod 能被调度到同一张卡。

它不代表：

- 每个 Pod 获得固定 50% 算力；
- 每个 Pod 获得独立显存；
- 一个 Pod 的 CUDA/OOM/GPU 故障不会影响同卡其他 Pod；
- 物理 GPU 吞吐翻倍。

共享准入必须满足：

```text
模型权重显存 + 峰值 KV Cache + 推理运行时开销 + 安全余量
< 物理 GPU 总显存
```

## 原方案实测结果

隔离测试使用一个 3B 指令模型和单张 EC2 G5 GPU，对比独占、共享但邻居空闲、共享双活跃三种情况：

| 场景 | 单请求平均延迟 | 单流吞吐 | 10 并发聚合吞吐 |
|---|---:|---:|---:|
| 单模型独占 | 1.83 s | 约 66.5 tok/s | 481.98 tok/s |
| 两模型共享、邻居空闲 | 1.96 s | 约 67.5 tok/s | 499.10 tok/s |
| 两模型共享、双活跃：模型 A | 4.71 s | 约 31 tok/s | 292.07 tok/s |
| 两模型共享、双活跃：模型 B | 3.90 s | 约 31 tok/s | 220.02 tok/s |
| 两模型共享、双活跃合计 | — | 约 62 tok/s | 512.09 tok/s |

结论：邻居空闲时，活跃模型接近独占基线；双模型同时满载时，整卡聚合吞吐相近，但单请求延迟约升至基线的 2.1–2.6 倍，单流吞吐约降至一半。这说明 Time-Slicing 可以提高低频模型的资源密度，但不能用整卡吞吐代替 TTFT、P95/P99 和队列时间做上线判断。

以上结果只对应原测试模型、GPU、软件版本和负载形状，不是通用性能承诺。

## 实施步骤

### 1. 做业务画像和独占基线

至少采集 1–2 周：

- QPS、并发和峰值重叠；
- 输入/输出 Token 与上下文长度；
- TTFT、ITL、P95/P99 和队列时间；
- GPU 利用率、显存和 KV Cache；
- GPU 小时、空闲率和单位 Token 成本。

先确认模型是低频或错峰，而不是持续高负载。

### 2. 计算显存预算并选择模型配对

对每个模型计算常驻权重、峰值 KV Cache、运行时和安全余量。只有总量能稳定低于物理显存，且峰值不会高度重叠的模型，才进入共享 PoC。

原测试中，独占模式使用 `--gpu-memory-utilization=0.90`；两模型共享时每个模型使用 `0.40`，合计约 80%，为运行时波动保留余量。该值只是原测试样例，不能直接复制到其他模型。

### 3. 建立共享和独占 GPU 池

- 共享池：低频、可降级、可容忍延迟抖动的模型；
- 独占 GPU 池：严格 SLO、长上下文、高优先级或强隔离模型；
- 使用 Node Label、Taint/Toleration 和 Node Affinity 防止混调；
- 关键副本跨物理节点分布，避免单卡故障同时影响所有副本。

### 4. 启用 Time-Slicing

仓库示例从每张物理 GPU 两个逻辑副本开始：

```bash
kubectl apply -f manifests/time-slicing-config.yaml
```

将该 ConfigMap 配置到 NVIDIA GPU Operator 或 Device Plugin 后，在维护窗口滚动重启 Device Plugin。随后验证节点报告的可调度 GPU 数量是否从 1 变为 2。

具体挂载 ConfigMap 的方式随 GPU Operator / Device Plugin 安装方式和版本而变化，不能只创建 ConfigMap 就认为已生效。

### 5. 部署共享推理服务

先替换 `manifests/shared-inference.yaml` 中的：

- 不可变 ECR 镜像 Digest；
- 模型路径和启动参数；
- `--gpu-memory-utilization`；
- Namespace、Service、健康检查和安全配置。

然后执行：

```bash
kubectl apply -f manifests/priority-and-pdb.yaml
kubectl apply -f manifests/shared-inference.yaml
```

Readiness 必须等模型加载和预热成功后再通过，不能只判断容器进程已启动。

### 6. 配置两级弹性

安装并配置 KEDA/Prometheus 后，替换示例中的 Prometheus Endpoint 和指标名：

```bash
kubectl apply -f manifests/keda-scaledobject.yaml
```

KEDA/HPA 根据 Waiting Requests、Queue Time 和延迟扩 Pod。新增 Pod 因 GPU 不足而 Pending 时，再由 Karpenter 创建 GPU 节点：

```bash
kubectl apply -f manifests/karpenter-nodepool.yaml
```

应用 Karpenter 示例前，必须先创建并替换实际的 `EC2NodeClass`，同时按目标 Region、实例类型、配额和容量策略检查配置。

### 7. 做全活跃压测和故障演练

必须覆盖：

- 单活跃、部分活跃、全部模型同时活跃；
- 短、中、长上下文和突发流量；
- OOM、CUDA/Xid、Pod 重启、节点丢失；
- Pod Pending、节点创建、镜像拉取、权重加载和预热；
- 共享池回退到独占池。

### 8. 按 SLO 验收

只有同时满足以下条件才上线：

- TTFT、P95/P99、错误率和队列时间满足业务 SLO；
- 峰值显存仍保留安全余量，无不可接受 OOM；
- 扩容能在业务等待预算内完成；
- 单位 Token 成本或 GPU 空闲率达到预期改善；
- 已验证回退到独占 GPU 池的路径。

## 最小观测指标

| 维度 | 指标 |
|---|---|
| GPU | 利用率、显存、温度、功耗、ECC/Xid |
| Kubernetes | Pending、调度失败、Pod 重启、节点创建时间 |
| 推理服务 | QPS、Running/Waiting、队列时间、KV Cache、Tokens/s |
| 用户体验 | TTFT、ITL、P50/P95/P99、超时和错误率 |
| 成本 | GPU 小时、空闲率、单位输出 Token 成本、SLO 达标率 |

## 仓库文件

- `manifests/time-slicing-config.yaml`：每张 GPU 两个逻辑副本
- `manifests/shared-inference.yaml`：共享池推理服务示例
- `manifests/priority-and-pdb.yaml`：优先级和中断保护
- `manifests/keda-scaledobject.yaml`：队列驱动 Pod 扩缩容
- `manifests/karpenter-nodepool.yaml`：共享 GPU 节点池示例
- `docs/production-checklist.md`：生产检查清单

这些 Manifest 是参考模板，包含占位符。必须按目标 EKS、Karpenter、KEDA、GPU Operator、AMI 和推理引擎版本检查后再应用。

## 参考资料

- [在 Amazon EKS 上管理 NVIDIA GPU](https://docs.aws.amazon.com/eks/latest/userguide/device-management-nvidia.html)
- [在 Amazon EKS 上自动扩缩 GPU 模型推理](https://docs.aws.amazon.com/eks/latest/userguide/ml-inference-autoscaling.html)
- [NVIDIA GPU Operator：Time-Slicing](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-sharing.html)
