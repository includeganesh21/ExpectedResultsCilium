# Tracing eBPF/kube-proxy Without Direct Node Access

Since you don't have direct node access but can create privileged pods, here's how to set up performance tracing in your EKS cluster:

## Method 1: Privileged Debug Pod for Node Inspection

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: node-debugger
  namespace: default
spec:
  hostNetwork: true
  hostPID: true
  containers:
  - name: debugger
    image: quay.io/cilium/debug:v1.12
    command: ["sleep", "infinity"]
    securityContext:
      privileged: true
      capabilities:
        add: ["SYS_ADMIN", "NET_ADMIN"]
  nodeSelector:
    kubernetes.io/hostname: <your-target-node>  # specify node name
```

### Commands to Run Inside Debug Pod:

**For kube-proxy (iptables) tracing:**
```bash
# Monitor iptables changes for a specific service
watch -n 0.1 "iptables -t nat -L KUBE-SERVICES -v -n --line-numbers | grep <your-service-ip>"

# Continuous logging of changes
iptables -t nat -L KUBE-SERVICES -v -n -x > iptables-before.txt
# After making changes
iptables -t nat -L KUBE-SERVICES -v -n -x > iptables-after.txt
diff -u iptables-before.txt iptables-after.txt
```

**For Cilium (eBPF) tracing:**
```bash
# Install bpftrace if not available
apt-get update && apt-get install -y bpftrace

# Trace service updates
bpftrace -e 'kprobe:__cilium_lxc* { @start[tid] = nsecs; }
              kretprobe:__cilium_lxc* { @latency = hist(nsecs - @start[tid]); }'
```

## Method 2: Using Cilium's Hubble for Observability

If you have Hubble enabled:
```bash
# Observe real-time service updates
kubectl exec -it <cilium-pod> -n kube-system -- hubble observe --follow --type trace

# Filter for specific service
kubectl exec -it <cilium-pod> -n kube-system -- hubble observe --follow --to-ip <service-ip>
```

## Method 3: Kubernetes Audit Logs + Metrics

For control plane timing:
```bash
# Check kube-proxy logs
kubectl logs -l k8s-app=kube-proxy -n kube-system --tail=50 -f

# Check Cilium agent logs
kubectl logs -l k8s-app=cilium -n kube-system --tail=50 -f

# Get metrics
kubectl exec -it <cilium-pod> -n kube-system -- cilium metrics list
```

## Scaling Test Procedure (1→44 pods):

1. Prepare your privileged pod on target node
2. In one terminal, start monitoring:
```bash
kubectl exec -it node-debugger -- watch -n 0.1 "iptables -t nat -L KUBE-SERVICES -v -n | grep <service-ip>"
```
3. In another terminal, trigger scaling:
```bash
date +%s.%N; kubectl scale deploy/test --replicas=44; date +%s.%N
```
4. Observe:
   - Time difference between date commands = API processing time
   - Time until iptables stabilize = dataplane propagation time

## Expected Results:

| Metric                | Cilium (eBPF)          | kube-proxy (iptables)   |
|-----------------------|------------------------|-------------------------|
| API to dataplane sync | 50-500ms total         | 5-30 seconds            |
| Update granularity     | Per-endpoint           | Full table rewrite      |
| CPU impact            | Minimal spike          | Significant node CPU    |
| Connection drops      | None                   | Possible during update  |

Would you like me to provide specific bpftrace scripts tailored to your service IPs for more precise measurement?

--- 
# Accessing Cilium vs kube-proxy Metrics via Prometheus

Here's how to collect and compare the key performance metrics through Prometheus in your EKS cluster:

## 1. Setting Up Prometheus Monitoring

First, ensure you have Prometheus installed (if not, use the kube-prometheus-stack):

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install kube-prometheus prometheus-community/kube-prometheus-stack
```

## 2. Key Metrics to Compare

### For Cilium (eBPF):

| Metric | PromQL Query | Description |
|--------|--------------|-------------|
| Endpoint update latency | `rate(cilium_endpoint_regeneration_time_stats_seconds_sum[1m])` | Time to program new endpoints |
| Policy update latency | `rate(cilium_policy_regeneration_time_stats_seconds_sum[1m])` | Time to enforce network policies |
| Service update latency | `rate(cilium_service_update_time_stats_seconds_sum[1m])` | Time to update service mappings |
| Forwarding latency | `rate(cilium_datapath_packet_processing_seconds_sum[1m])` | Packet processing time |

### For kube-proxy (iptables):

| Metric | PromQL Query | Description |
|--------|--------------|-------------|
| Sync latency | `rate(kubeproxy_sync_proxy_rules_duration_seconds_sum[1m])` | Time to sync iptables rules |
| Rules count | `kubeproxy_network_programming_duration_seconds_count` | Number of programmed rules |
| Sync errors | `rate(kubeproxy_sync_proxy_rules_last_timestamp_seconds[1m])` | Sync operation frequency |

## 3. Creating Performance Dashboards

Create a Grafana dashboard with these panels:

**Cilium Performance:**
```json
{
  "panels": [
    {
      "title": "Endpoint Update Latency",
      "targets": [{
        "expr": "histogram_quantile(0.95, rate(cilium_endpoint_regeneration_time_stats_seconds_bucket[1m]))",
        "legendFormat": "P95 latency"
      }]
    },
    {
      "title": "Service Update Operations",
      "targets": [{
        "expr": "rate(cilium_service_events_total[1m])",
        "legendFormat": "Updates/sec"
      }]
    }
  ]
}
```

**kube-proxy Performance:**
```json
{
  "panels": [
    {
      "title": "iptables Sync Latency",
      "targets": [{
        "expr": "histogram_quantile(0.95, rate(kubeproxy_sync_proxy_rules_duration_seconds_bucket[1m]))",
        "legendFormat": "P95 sync time"
      }]
    },
    {
      "title": "Rules Count",
      "targets": [{
        "expr": "kubeproxy_network_programming_duration_seconds_count",
        "legendFormat": "Total rules"
      }]
    }
  ]
}
```

## 4. Running Controlled Tests

### Scaling Test (1→44 pods):

1. **Before scaling:**
   ```bash
   # Capture baseline metrics
   kubectl get --raw "/api/v1/namespaces/monitoring/services/kube-prometheus-prometheus:9090/proxy/api/v1/query?query=cilium_endpoint_regeneration_time_stats_seconds_sum" > cilium_before.json
   ```

2. **Trigger scaling:**
   ```bash
   date +%s; kubectl scale deploy/test --replicas=44; date +%s
   ```

3. **Monitor metrics during scaling:**
   ```promql
   # Cilium endpoint updates
   rate(cilium_endpoint_regeneration_time_stats_seconds_sum[30s])
   
   # kube-proxy sync operations
   rate(kubeproxy_sync_proxy_rules_duration_seconds_sum[30s])
   ```

### Rolling Restart Test:

1. **Before restart:**
   ```bash
   kubectl get --raw "/api/v1/namespaces/monitoring/services/kube-prometheus-prometheus:9090/proxy/api/v1/query?query=kubeproxy_sync_proxy_rules_duration_seconds_count" > kubeproxy_before.json
   ```

2. **Trigger restart:**
   ```bash
   kubectl rollout restart deployment/test
   ```

3. **Monitor impact:**
   ```promql
   # Connection drops (if you have proper metrics exposed)
   rate(http_requests_total{status=~"5.."}[30s])
   
   # Latency spikes
   histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[30s]))
   ```

## 5. Expected Results in Prometheus

| Scenario | Cilium Metric Pattern | kube-proxy Metric Pattern |
|----------|-----------------------|--------------------------|
| Scaling 1→44 | Brief spike in `cilium_endpoint_regeneration_time_stats` | Sustained high `kubeproxy_sync_proxy_rules_duration_seconds` |
| Rolling restart | Multiple small spikes in service metrics | Long sync duration with possible errors |
| Steady state | Low, consistent processing times | Variable latency based on service count |

## 6. Additional Useful Metrics

```promql
# CPU impact comparison
rate(container_cpu_usage_seconds_total{container=~"cilium-agent|kube-proxy"}[1m])

# Memory comparison
container_memory_working_set_bytes{container=~"cilium-agent|kube-proxy"}

# Network performance
rate(cilium_drop_count_total[1m])  # Packet drops
rate(cilium_forward_count_total[1m])  # Forwarding rate
```

Would you like me to provide specific alert rules based on these metrics to detect performance degradation thresholds?
