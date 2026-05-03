---
name: GKE Autopilot Request Bloat
description: Over-specified pod resource requests in GKE Autopilot -- billed at the request level rather than node level, making right-sizing the primary cost lever.
scope: gcp
color: "#326CE5"
emoji: ⎈
---

## Problem

In GKE Standard clusters, oversized pod requests waste node headroom but
at least share a node with other pods — the waste is diluted. In
**GKE Autopilot**, every pod is billed directly at its requested CPU, memory,
and ephemeral-storage level. There are no nodes to share. There is no
dilution.

A pod requesting `cpu: 2` and `memory: 4Gi` when it actually uses 0.2 CPU
and 512 Mi is billed at 10x its actual consumption. Multiply across 50
microservices, each copied from a default deployment template with
`cpu: 1, memory: 2Gi`, and the cluster costs 5-10x more than necessary.

The problem is amplified by three Autopilot behaviors:

1. **Requests are rounded up** to the nearest resource class (micro / small /
   medium / large). A request of `cpu: 0.3` rounds up to `cpu: 0.5`, billing
   67% more than requested.
2. **No limits cap the bill**: In Standard clusters, CPU limits prevent
   bursting. In Autopilot, the billing is based on requests, not actual
   usage — but GKE may auto-adjust requests via built-in VPA.
3. **Copied-from-production templates**: Devs frequently copy deployment
   YAML from a production workload to a development workload. The production
   requests (sized for 10k RPS) land on the dev pod (serving 0 RPS).

## Symptoms

- Autopilot cluster cost growing with pod count, not with actual workload
- GKE Cost Allocation shows high cost for low-traffic namespaces
- Many pods with `cpu: 1+` requests and < 0.1 actual CPU usage
- Dev / staging pods using the same resource spec as production
- `kubectl top pods` showing < 10% of requested CPU used across namespaces

## Detection

```bash
# Compare requests vs actual usage for all pods
kubectl top pods --all-namespaces | sort -k4 -rn > usage.txt
kubectl get pods --all-namespaces -o json | jq -r '
  .items[] |
  .metadata.namespace + "/" + .metadata.name + "\t" +
  (.spec.containers[0].resources.requests.cpu // "none") + "\t" +
  (.spec.containers[0].resources.requests.memory // "none")
' > requests.txt
```

Use the GKE Cost Allocation feature for billing-level data:

```sql
-- Cost per namespace from GKE cost allocation BigQuery export
SELECT
  namespace,
  SUM(cpu_request_cores * cpu_price_per_core_hour) AS cpu_cost_usd,
  SUM(memory_request_gib * memory_price_per_gib_hour) AS memory_cost_usd,
  SUM(cpu_request_cores * cpu_price_per_core_hour +
      memory_request_gib * memory_price_per_gib_hour) AS total_cost_usd
FROM `MY_PROJECT.MY_DATASET.gke_cluster_resource_usage_*`
WHERE DATE(start_time) >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY namespace
ORDER BY total_cost_usd DESC;
```

Cross-reference with Prometheus / Cloud Monitoring actual usage:

```bash
# p95 actual CPU usage per pod over 14 days (via Cloud Monitoring)
gcloud monitoring time-series list \
  --filter='metric.type="kubernetes.io/container/cpu/core_usage_time" resource.labels.cluster_name="MY-CLUSTER"' \
  --interval-start-time=$(date -d '14 days ago' -u +%Y-%m-%dT%H:%M:%SZ) \
  --interval-end-time=$(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --aggregation-per-series-aligner=ALIGN_RATE \
  --aggregation-cross-series-reducer=REDUCE_PERCENTILE_95
```

## Fix

### 1. Right-size requests from observed p95 usage

```yaml
# Before (over-sized from production template)
resources:
  requests:
    cpu: "1000m"
    memory: "2Gi"

# After (sized from 14-day p95 + 1.3x memory safety margin)
resources:
  requests:
    cpu: "100m"     # p95 was 60m; round to Autopilot class boundary
    memory: "256Mi" # p95 was 180Mi × 1.3 = 234Mi; round up to 256Mi
```

**Autopilot resource class boundaries (round up to nearest):**

| CPU | Memory |
|---|---|
| 0.25 vCPU | 0.5 GB |
| 0.5 vCPU | 1 GB |
| 1 vCPU | 2 GB |
| 2 vCPU | 4 GB |
| 4 vCPU | 8 GB |

Requests below a class boundary round up. Setting `cpu: 300m` bills
the same as `cpu: 500m`. Adjust to class boundaries explicitly.

### 2. Use environment-specific Kustomize overlays

```yaml
# base/deployment.yaml (production sizing)
resources:
  requests:
    cpu: "1000m"
    memory: "2Gi"

# overlays/dev/patch-resources.yaml
- op: replace
  path: /spec/template/spec/containers/0/resources/requests/cpu
  value: "100m"
- op: replace
  path: /spec/template/spec/containers/0/resources/requests/memory
  value: "256Mi"
```

Enforce the dev overlay in CI for any deployment targeting a
`env=dev` or `env=staging` namespace.

### 3. Use Autopilot Spot for non-critical workloads

For dev/staging pods where interruption is acceptable, Spot pods
cost ~60-70% less than on-demand Autopilot:

```yaml
spec:
  nodeSelector:
    cloud.google.com/gke-spot: "true"
  tolerations:
    - key: cloud.google.com/gke-spot
      operator: Equal
      value: "true"
      effect: NoSchedule
```

Combined with right-sized requests, this typically achieves 80-90%
cost reduction vs production-sized on-demand Autopilot pods.

### 4. Set namespace resource quotas to enforce right-sizing

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: my-service-dev
spec:
  hard:
    requests.cpu: "2"       # Total for the namespace
    requests.memory: "4Gi"
    limits.cpu: "4"
    limits.memory: "8Gi"
```

Quotas force developers to be intentional about resource allocation
rather than copy-pasting from production.

## Anti-patterns

- **"Autopilot handles rightsizing automatically"**: Autopilot's built-in
  VPA adjusts requests based on usage, but it can only adjust at pod
  recreation time. For long-running pods that are never recreated, the
  original request persists.
- **Single Kubernetes YAML for all environments**: The production pod spec
  should never be the dev pod spec. Use overlays or separate manifests.
- **Monitoring only node metrics**: In Autopilot, node metrics are not
  visible. Monitor pod request vs actual usage via GKE Cost Allocation
  and Cloud Monitoring container metrics.

## References

- [GKE Autopilot resource requests](https://cloud.google.com/kubernetes-engine/docs/concepts/autopilot-resource-requests)
- [GKE Cost Allocation](https://cloud.google.com/kubernetes-engine/docs/how-to/cost-allocations)
- [Autopilot pricing](https://cloud.google.com/kubernetes-engine/pricing#autopilot_mode)
- Agent: `kubernetes/kubernetes-finops-engineer.md`
- Agent: `kubernetes/kubernetes-workload-optimizer.md`
