# Amazon EKS GPU Time-Slicing 端到端部署指南

本文把仓库中的示例 Manifest 串成一条可执行路径：在已有 Amazon EKS 集群中准备 G5/A10G GPU 节点，启用 NVIDIA Time-Slicing，部署两个共享 GPU 的 vLLM 服务，并按需接入 KEDA 与 Karpenter。

> 本方案的共享池使用 G5/A10G Time-Slicing。它增加 Kubernetes 可调度名额，但不增加物理算力，也不隔离显存。中国区固定隔离池可使用 P4d/A100；MIG 不适用于 A10G。

## 1. 部署结果

完成后应得到：

- GPU 节点标签：`workload.example.com/gpu-pool=shared`；
- 一张物理 GPU 向 Kubernetes 上报两个 `nvidia.com/gpu` 名额；
- 两个各申请一个 GPU 名额的推理 Pod 可以落到同一张 A10G；
- KEDA 根据队列指标扩 Pod；
- 新 Pod 因 GPU 不足而 Pending 时，Karpenter 扩 G5 节点。

## 2. 前置条件

- 已有 Amazon EKS 集群和 `kubectl` 访问权限；
- 已安装 Helm 3；
- 至少一台使用 EKS GPU 优化 AMI 的 G5 节点，或已安装 Karpenter；
- 推理镜像已推送到 Amazon ECR，并能在端口 8000 提供 `/health`；
- 模型权重、启动参数和 ECR Digest 已确定；
- 如启用弹性，集群中已有 Prometheus 指标源。

确认连接：

```bash
aws sts get-caller-identity
kubectl cluster-info
kubectl get nodes -o wide
```

## 3. 获取代码并设置变量

```bash
git clone https://github.com/zjinsong/amazon-eks-multi-model-gpu-sharing.git
cd amazon-eks-multi-model-gpu-sharing

export AWS_REGION=<region>
export CLUSTER_NAME=<eks-cluster-name>
export GPU_NODE=<existing-g5-node-name>
export ECR_IMAGE=<account-id>.dkr.ecr.<region>.amazonaws.com.cn/<repo>@sha256:<digest>
```

不要使用可变的 `:latest` 标签；生产部署应固定到镜像 Digest。

## 4. 准备共享 GPU 节点

如果使用现有 G5 节点，添加共享池标签：

```bash
kubectl label node "${GPU_NODE}" workload.example.com/gpu-pool=shared --overwrite
kubectl taint node "${GPU_NODE}" nvidia.com/gpu=true:NoSchedule --overwrite
```

确认节点能识别 NVIDIA GPU：

```bash
kubectl get node "${GPU_NODE}" -o jsonpath='{.status.capacity.nvidia\.com/gpu}{"\n"}'
```

如果结果为空，先检查 GPU 优化 AMI、NVIDIA 驱动、Container Toolkit 和 Device Plugin，不能继续部署推理 Pod。

## 5. 安装 NVIDIA GPU Operator

以下步骤使用 GPU Operator 管理 Device Plugin。已有 GPU Operator 的集群跳过安装，只确认 Namespace 和 ClusterPolicy 名称。

```bash
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm repo update

helm upgrade --install gpu-operator nvidia/gpu-operator \
  --namespace gpu-operator \
  --create-namespace \
  --wait

kubectl get pods -n gpu-operator
kubectl get clusterpolicy
```

所有必需 Pod 应进入 `Running` 或 `Completed`，再继续启用 Time-Slicing。

## 6. 启用 Time-Slicing

仓库配置从每张 GPU 两个逻辑副本开始：

```bash
kubectl apply -f manifests/time-slicing-config.yaml

kubectl patch clusterpolicy cluster-policy --type merge -p \
  '{"spec":{"devicePlugin":{"config":{"name":"time-slicing-config","default":"any"}}}}'
```

如果 ClusterPolicy 不是 `cluster-policy`，先用 `kubectl get clusterpolicy` 查询实际名称并替换。

等待 Device Plugin 更新：

```bash
kubectl rollout status daemonset/nvidia-device-plugin-daemonset \
  -n gpu-operator --timeout=5m

kubectl get node "${GPU_NODE}" \
  -o jsonpath='{.status.allocatable.nvidia\.com/gpu}{"\n"}'
```

预期结果为 `2`。如果仍是 `1`：

```bash
kubectl get configmap time-slicing-config -n gpu-operator -o yaml
kubectl get clusterpolicy cluster-policy -o yaml
kubectl logs -n gpu-operator daemonset/nvidia-device-plugin-daemonset \
  --all-containers --tail=200
```

仅创建 ConfigMap 不代表 Time-Slicing 已经生效；必须让 Device Plugin 引用该 ConfigMap。

## 7. 配置并部署推理服务

编辑 `manifests/shared-inference.yaml`，至少替换：

- `REPLACE_ME_WITH_AN_IMMUTABLE_ECR_DIGEST`；
- 模型名称或模型路径；
- vLLM 启动参数；
- `--gpu-memory-utilization`；
- Readiness 路径和端口。

本仓库实测示例在 A10G 24 GB 上为每个 vLLM Pod 设置 `0.40`。这表示约 9.6 GB 的 vLLM 显存规划目标，不是显存硬隔离，也不能保证进程绝不超过该数值。

```bash
sed -i "s|REPLACE_ME_WITH_AN_IMMUTABLE_ECR_DIGEST|${ECR_IMAGE}|g" \
  manifests/shared-inference.yaml

kubectl apply -f manifests/priority-and-pdb.yaml
kubectl apply -f manifests/shared-inference.yaml
kubectl rollout status deployment/example-shared-inference --timeout=15m
```

若要部署两个不同模型，复制 `shared-inference.yaml` 中的 Deployment 和 Service，为第二个模型修改：

- Deployment、Service 和 `app` 标签名称；
- ECR 镜像或模型路径；
- vLLM 参数；
- 显存预算。

两个 Pod 都应保留：

```yaml
resources:
  limits:
    nvidia.com/gpu: "1"
nodeSelector:
  workload.example.com/gpu-pool: shared
```

## 8. 验证两 Pod 共享一张物理 GPU

```bash
kubectl get pods -o wide
kubectl describe node "${GPU_NODE}" | sed -n '/Capacity:/,/System Info:/p'
```

进入任一推理 Pod 检查 GPU：

```bash
POD=$(kubectl get pod -l app=example-shared-inference \
  -o jsonpath='{.items[0].metadata.name}')
kubectl exec "${POD}" -- nvidia-smi
```

至少验证以下三种负载：

1. 单模型独占基线；
2. 两模型共享但邻居空闲；
3. 两模型同时活跃。

同时观察 TTFT、P95/P99、Waiting Requests、KV Cache、GPU 显存和 CUDA OOM。不能只看整卡 Tokens/s。

## 9. 配置 KEDA 扩 Pod

先安装 KEDA：

```bash
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
helm upgrade --install keda kedacore/keda \
  --namespace keda \
  --create-namespace \
  --wait
```

编辑 `manifests/keda-scaledobject.yaml`：

- 将 `serverAddress` 替换为集群可访问的 Prometheus 地址；
- 将查询中的 `inference_waiting_requests` 替换为实际指标；
- 根据压测结果设置 `threshold`、`maxReplicaCount` 和 `cooldownPeriod`。

```bash
kubectl apply -f manifests/keda-scaledobject.yaml
kubectl get scaledobject
kubectl get hpa
```

KEDA 应基于队列、并发、TTFT/P99 等提前扩容。GPU OOM 更适合作为保护和告警信号，不应等 OOM 后才触发扩容。

## 10. 配置 Karpenter 扩 GPU 节点

此步骤假设 Karpenter 已按目标 EKS 版本安装，并已创建具备子网、安全组、实例角色和 GPU AMI 配置的 `EC2NodeClass`。

编辑 `manifests/karpenter-nodepool.yaml`：

- 把 `REPLACE_ME_GPU_NODE_CLASS` 替换为实际 EC2NodeClass；
- 增加目标 Region 可用的 G5 实例约束；
- 检查 On-Demand/Spot 策略、配额、子网和安全组。

例如将实例族限定为 G5：

```yaml
- key: karpenter.k8s.aws/instance-family
  operator: In
  values: ["g5"]
```

然后执行：

```bash
kubectl apply -f manifests/karpenter-nodepool.yaml
kubectl get nodepool
kubectl get ec2nodeclass
```

验证闭环：

```bash
kubectl get pods -w
kubectl get nodes -w
```

预期过程：

```text
队列/延迟达到阈值
  → KEDA/HPA 创建新 Pod
  → 没有可用 GPU，Pod Pending
  → Karpenter 创建 G5 节点
  → Device Plugin Ready
  → Pod 调度、加载模型并通过 Readiness
```

## 11. 接入 Model Router

Model Router 位于 ALB/NLB Ingress 与两类 GPU Service 之间。至少按以下信息选择后端：

- 模型名称和版本；
- 业务优先级；
- TTFT/P99 SLO；
- 当前队列长度；
- 共享池是否仍有显存安全余量。

低频、错峰且可容忍延迟波动的请求进入共享池；严格 SLO、高负载或强隔离请求进入独占池。中国区可在 P4d/A100 上使用 Dedicated GPU 或 MIG 固定分区。

## 12. 上线检查

```bash
kubectl get pods -A
kubectl get node "${GPU_NODE}" \
  -o jsonpath='{.status.allocatable.nvidia\.com/gpu}{"\n"}'
kubectl get scaledobject,hpa
kubectl get nodepool,ec2nodeclass
```

上线前还应完成 [生产检查清单](production-checklist.md)，并验证：

- 双活跃压测仍满足 TTFT/P95/P99；
- 显存保留安全余量；
- OOM、CUDA/Xid 和节点故障演练；
- KEDA 到 Karpenter 的扩容耗时满足等待预算；
- 共享池可回退到独占池。

## 13. 清理

```bash
kubectl delete -f manifests/keda-scaledobject.yaml --ignore-not-found
kubectl delete -f manifests/shared-inference.yaml --ignore-not-found
kubectl delete -f manifests/priority-and-pdb.yaml --ignore-not-found
kubectl delete -f manifests/karpenter-nodepool.yaml --ignore-not-found
```

不要在仍有其他 GPU 工作负载时删除 GPU Operator。若需恢复每卡一个调度名额，应先从 ClusterPolicy 移除 Time-Slicing 配置并滚动更新 Device Plugin，再删除 ConfigMap。

## 14. 常见问题

### Pod 一直 Pending

检查 `nodeSelector`、Taint/Toleration、节点 `allocatable nvidia.com/gpu`、Karpenter NodePool 与 EC2NodeClass。

### 配置 replicas: 2 后节点仍显示 1

ConfigMap 尚未被 Device Plugin 引用，或 DaemonSet 尚未完成更新。检查 ClusterPolicy 和 Device Plugin 日志。

### 某个 Pod OOM

Time-Slicing 没有显存隔离。降低并发、上下文长度、`max_tokens` 或 `gpu-memory-utilization`，减少同卡模型数量，或者把该模型迁移到独占/MIG 固定池。

### KEDA 扩容但性能没有改善

确认新增 Pod 是否落到另一张 GPU。如果它仍落在同一张已饱和 GPU 上，只会增加算力和显存竞争。应限制每卡共享密度，并让 Pending Pod 触发 Karpenter 扩节点。
