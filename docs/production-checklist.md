# 生产检查清单

## 共享准入

- 明确模型权重、最大上下文、最大输出、并发、峰值 KV Cache、运行时开销和安全余量；
- 对每组候选模型回放同时峰值；
- TTFT、P95/P99、队列时间或错误率不满足 SLO 时，拒绝共享；
- 核心业务、强隔离或长上下文模型优先使用独占 GPU / MIG。

## 可靠性

- 关键副本跨物理节点分布；
- 演练 OOM、CUDA/Xid、Pod 重启、节点替换和 GPU 容量不足；
- 只有模型加载并完成预热后，Readiness 才通过；
- 配置请求长度、并发、队列、限流、备用模型和主动拒绝策略；
- 保留并验证从共享池回退到独占池的路径。

## 安全

- 优先使用私有网络并限制 Security Group；
- 使用 EKS Pod Identity 或 IRSA，不在镜像和 Manifest 中嵌入密钥；
- 将 ECR 和模型存储权限收敛到必要 Repository 和 Prefix；
- 不把客户数据、账户 ID、Endpoint 或内部资源名提交到公共仓库。

## 运维

- Time-Slicing ConfigMap 变更后，在维护窗口滚动重启 NVIDIA Device Plugin；
- 验证节点报告的逻辑 GPU 数量与配置一致；
- 持续观察显存、KV Cache、TTFT、P95/P99、队列、OOM 和 Xid；
- 定期用生产遥测调整模型配对、共享密度和弹性阈值；
- 复盘 GPU 小时、空闲率、单位 Token 成本和 SLO 达标率。
