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

---

Here are the detailed metrics with expected values for Cilium (eBPF) vs kube-proxy (iptables) in your EKS cluster:

### 1. **Control Plane Update Latency Metrics**

| Metric | PromQL Query | Cilium Expected Value | kube-proxy Expected Value | Notes |
|--------|--------------|-----------------------|---------------------------|-------|
| Endpoint Update Time | `histogram_quantile(0.95, rate(cilium_endpoint_regeneration_time_stats_seconds_bucket[1m]))` | 5-50ms per endpoint | N/A | Cilium updates endpoints individually |
| Service Sync Latency | `histogram_quantile(0.95, rate(kubeproxy_sync_proxy_rules_duration_seconds_bucket[1m]))` | N/A | 500ms-5s per sync | kube-proxy updates entire rule set |
| Service Update Time | `rate(cilium_service_update_time_stats_seconds_sum[1m])` | 1-10ms per service | N/A | eBPF map updates are atomic |

### 2. **Data Plane Performance Metrics**

| Metric | PromQL Query | Cilium Expected Value | kube-proxy Expected Value |
|--------|--------------|-----------------------|---------------------------|
| Packet Processing Time | `rate(cilium_datapath_packet_processing_seconds_sum[1m])` | 10-100μs per packet | 50-500μs per packet |
| Connection Setup Time | `rate(cilium_forward_count_total[1m]) / rate(cilium_flows_to_world_total[1m])` | 0.1-1ms | 1-10ms |
| Rule Lookup Time | `rate(cilium_policy_l7_denied_total[1m])` | Sub-millisecond | 1-10ms (depends on chain length) |

### 3. **Scaling Event Metrics (1→44 pods)**

| Metric | Expected Cilium Behavior | Expected kube-proxy Behavior |
|--------|--------------------------|-----------------------------|
| Total Update Time | 100-500ms (parallel updates) | 5-30s (serial iptables rebuild) |
| CPU Spike | <5% increase | 20-50% increase during sync |
| Packet Loss | 0% | Possible 1-5% during sync |
| API to Dataplane Latency | 50-200ms | 2-10s |

### 4. **Rolling Restart Metrics (44 pods)**

| Metric | Cilium Values | kube-proxy Values |
|--------|---------------|-------------------|
| Per-Pod Update Time | 10-50ms | N/A (bulk updates) |
| Total Impact Duration | ~10s | ~30-60s |
| Concurrent Connections Dropped | 0-0.1% | 0.5-5% |
| Max Latency During Update | <2x baseline | 5-10x baseline |

### 5. **Steady-State Metrics**

| Metric | PromQL Query | Cilium Normal Range | kube-proxy Normal Range |
|--------|--------------|---------------------|-------------------------|
| Proxy CPU Usage | `rate(container_cpu_usage_seconds_total{container=~"cilium-agent|kube-proxy"}[5m])` | 0.1-0.5 cores | 0.3-1.5 cores |
| Memory Usage | `container_memory_working_set_bytes{container=~"cilium-agent|kube-proxy"}` | 200-500MB | 100-300MB |
| Rules Count | `kubeproxy_network_programming_duration_seconds_count` | N/A | 500-5000 rules |
| eBPF Map Entries | `cilium_bpf_map_ops_total` | 50-500 ops/sec | N/A |

### 6. **Critical Thresholds for Alerts**

```yaml
# For Cilium
- alert: CiliumEndpointUpdateSlow
  expr: histogram_quantile(0.95, rate(cilium_endpoint_regeneration_time_stats_seconds_bucket[1m])) > 0.1
  for: 5m

# For kube-proxy
- alert: KubeProxySyncTooSlow
  expr: histogram_quantile(0.95, rate(kubeproxy_sync_proxy_rules_duration_seconds_bucket[1m])) > 5
  for: 2m
```

### Expected Results During Your Tests:

**1→44 Scaling Test:**
- Cilium will show:
  - Brief spike in `cilium_endpoint_regeneration_time_stats`
  - `cilium_events_total{type="service"}` will increment by 44 quickly
- kube-proxy will show:
  - Sustained high values in `kubeproxy_sync_proxy_rules_duration_seconds`
  - Possible increase in `kubeproxy_sync_proxy_rules_errors_total`

**Rolling Restart 44 Pods:**
- Cilium metrics will show:
  - Multiple small spikes in `cilium_service_update_time_stats`
  - Steady `cilium_forward_count_total` during updates
- kube-proxy metrics will show:
  - 1-2 large spikes in sync duration
  - Possible drops in `kubeproxy_network_programming_duration_seconds_count`

Would you like me to provide specific Grafana dashboard JSON to visualize these comparisons?


---

Here are the most directly comparable metrics between Cilium (eBPF) and kube-proxy (iptables), along with expected results and technical explanations:

### 1. **Service Update Latency**
**Metric:** Time from Kubernetes API change to dataplane implementation  
**How to Compare:**
```promql
# Cilium
histogram_quantile(0.95, rate(cilium_service_update_time_stats_seconds_bucket[5m]))

# kube-proxy 
histogram_quantile(0.95, rate(kubeproxy_sync_proxy_rules_duration_seconds_bucket[5m]))
```

**Expected Results:**
- Cilium: **1-10ms**  
  (eBPF maps update atomically per service)
- kube-proxy: **500ms-5s**  
  (Requires full iptables rebuild)

**Why?**  
Cilium updates individual eBPF map entries while kube-proxy rewrites entire iptables chains (O(n) vs O(n²) complexity).

---

### 2. **Packet Processing Latency**
**Metric:** Per-packet forwarding time  
**How to Compare:**
```promql
# Cilium
rate(cilium_datapath_packet_processing_seconds_sum[1m])

# kube-proxy (indirect via node CPU profiling)
rate(node_cpu_seconds_total{mode="softirq"}[1m])
```

**Expected Results:**
- Cilium: **10-100μs**  
  (Single eBPF program lookup)
- kube-proxy: **50-500μs**  
  (Linear iptables rule traversal)

**Why?**  
eBPF programs compile to native machine code while iptables requires sequential rule evaluation.

---

### 3. **Connection Setup Time**
**Metric:** First packet to established connection  
**How to Compare:**
```promql
# Cilium
rate(cilium_flows_to_world_total[1m]) / rate(cilium_forward_count_total[1m])

# kube-proxy (requires application metrics)
rate(http_request_duration_seconds_count{phase="connect"}[1m])
```

**Expected Results:**
- Cilium: **0.1-1ms**  
  (Direct pod-to-pod via eBPF)
- kube-proxy: **1-10ms**  
  (Extra NAT hops in iptables)

**Why?**  
Cilium's eBPF datapath eliminates intermediate NAT steps.

---

### 4. **CPU Impact During Scaling Events**
**Metric:** CPU usage during 1→44 pod scale-up  
**How to Compare:**
```promql
rate(container_cpu_usage_seconds_total{container=~"cilium-agent|kube-proxy"}[30s])
```

**Expected Results:**
- Cilium: **<5% spike**  
  (Incremental updates)
- kube-proxy: **20-50% spike**  
  (Full rule recomputation)

**Why?**  
eBPF updates only changed endpoints while iptables rebuilds all rules.

---

### 5. **Dataplane Sync Frequency**
**Metric:** Number of control plane syncs  
**How to Compare:**
```promql
# Cilium
rate(cilium_events_total{type="service"}[1m])

# kube-proxy
rate(kubeproxy_sync_proxy_rules_last_timestamp_seconds[1m])
```

**Expected Results:**
- Cilium: **1-5 events/sec**  
  (Event-driven updates)
- kube-proxy: **0.1-1 syncs/sec**  
  (Batch processing)

**Why?**  
Cilium reacts to individual endpoint changes while kube-proxy uses periodic syncs.

---

### 6. **Memory Efficiency**
**Metric:** Memory per service  
**How to Compare:**
```promql
# Cilium
cilium_bpf_maps_virtual_memory_bytes / cilium_services_total

# kube-proxy
container_memory_usage_bytes{container="kube-proxy"} / kubeproxy_network_programming_duration_seconds_count
```

**Expected Results:**
- Cilium: **~0.5KB/service**  
  (Compact eBPF map entries)
- kube-proxy: **~2-5KB/service**  
  (iptables rule expansion)

**Why?**  
eBPF stores only essential forwarding info while iptables generates verbose rules.

---

### 7. **Failure Recovery Time**
**Metric:** Time to recover after pod crash  
**How to Compare:**
```promql
# Time between endpoint deletion and new endpoint becoming active
(max_over_time(cilium_endpoint_regeneration_time_stats_seconds_sum[1m]) - min_over_time(cilium_endpoint_regeneration_time_stats_seconds_sum[1m]))
```

**Expected Results:**
- Cilium: **50-200ms**  
  (Immediate eBPF map update)
- kube-proxy: **1-5s**  
  (Wait for next sync cycle)

**Why?**  
Cilium's Kubernetes watchers trigger instant updates.

---

### Key Insights from Direct Comparison:
1. **Latency:** Cilium wins by 10-100x in packet processing due to avoiding sequential iptables evaluation
2. **Scalability:** eBPF shows O(1) update characteristics vs kube-proxy's O(n²)
3. **Resource Usage:** Cilium maintains consistent low CPU/memory regardless of service count
4. **Reliability:** Zero packet loss during updates vs kube-proxy's potential drops

For the most definitive comparison, focus on **service update latency** and **packet processing time** - these show the starkest differences between the architectures.
