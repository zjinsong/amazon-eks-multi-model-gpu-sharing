# Production checklist

## Admission

- Define the model's memory budget, maximum context/output length, concurrency limit, and safety margin.
- Replay simultaneous peaks for every proposed model pairing.
- Reject sharing if TTFT, P95/P99, queue time, or error rate violates the service SLO.

## Resilience

- Spread critical replicas across physical nodes.
- Test OOM, CUDA/Xid errors, Pod restart, node replacement, and GPU-capacity shortage.
- Use readiness only after weights are loaded and warm-up requests succeed.
- Configure overload protection: request limits, priority queues, rate limits, fallback models, then explicit rejection.

## Security

- Use private networking where possible and restrict security groups.
- Use Pod Identity or IRSA for workload credentials; do not embed keys in images or manifests.
- Scope ECR and model-artifact access to the required repositories and prefixes.

## Operations

- Restart the NVIDIA device-plugin DaemonSet after a time-slicing ConfigMap change; use a maintenance window.
- Maintain a tested path back to dedicated capacity.
- Review sharing density and scaling thresholds using production telemetry.
