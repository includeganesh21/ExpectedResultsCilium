# Comparing Cilium (eBPF) vs kube-proxy (iptables) Performance in EKS

To properly compare the performance between Cilium's eBPF-based networking and traditional kube-proxy with iptables, you'll need to design controlled experiments that isolate the specific components you want to measure.

## 1. Measuring Packet Processing Latency

For measuring the time spent in eBPF maps vs iptables traversal:

### Method:
- Use **kernel tracing** to measure the time spent in critical paths:
  - For Cilium: Trace eBPF map lookups and updates
  - For kube-proxy: Trace iptables rule traversal

```bash
# For Cilium/eBPF:
sudo bpftrace -e 'kprobe:__cilium_lxc* { @start[tid] = nsecs; }
                  kretprobe:__cilium_lxc* { @latency = hist(nsecs - @start[tid]); delete(@start[tid]); }'

# For kube-proxy/iptables:
sudo trace-cmd record -e iptables -e nftables
```

### Alternative approach:
- Use **perf** to profile network-related syscalls:
```bash
perf trace -e 'net:*' -a -o cilium.trace
perf trace -e 'net:*' -a -o kube-proxy.trace
```

## 2. Measuring Control Plane Update Latency

For measuring service-to-pod mapping update times:

### Test Design:
1. **Create a benchmark service** with multiple endpoints
2. **Measure update latency** during:
   - Rolling restart of pods (`kubectl rollout restart deployment/your-app`)
   - Scaling up/down (`kubectl scale deployment/your-app --replicas=N`)

### Measurement Tools:
- For Cilium:
  ```bash
  # Monitor eBPF map updates
  cilium monitor --related-to 10.0.0.1 -v
  ```
  
- For kube-proxy:
  ```bash
  # Log iptables changes
  iptables -L -v -n -t nat --line-numbers | grep your-service
  ```

### Expected Results:
- **Cilium** updates should be:
  - Faster (milliseconds range)
  - More consistent (not dependent on number of rules)
  - Scale O(1) with number of services
  
- **kube-proxy** updates will:
  - Take longer (seconds range for large clusters)
  - Scale O(n) with number of services
  - Require full iptables rule rebuilds

## Implementation Tips:

1. **Isolate variables**:
   - Use identical node types/locations
   - Test in the same AZ to minimize network variance
   - Use dedicated test clusters

2. **Measurement tools**:
   - Use `cilium monitor` for Cilium-specific metrics
   - Use `kubectl proxy --v=6` to see kube-proxy update logs
   - Consider Prometheus metrics from both components

3. **Sample test sequence**:
```bash
# 1. Deploy test service
kubectl create deployment test --image=nginx
kubectl expose deployment test --port=80

# 2. Start monitoring
# In one terminal:
cilium monitor -v

# In another:
watch -n 0.1 "iptables -t nat -L KUBE-SERVICES -v --line-numbers | grep test"

# 3. Trigger changes
kubectl scale deployment test --replicas=10
kubectl scale deployment test --replicas=1
```

The key is to measure the time between when the Kubernetes API reports the endpoint change and when the networking layer reflects it in its dataplane.


---
# Expected Output and Methodology for Comparing Cilium vs kube-proxy

## For Your 1-to-44 Endpoint Scaling Test

### Expected Output Patterns:

**Cilium (eBPF) Behavior:**
1. **Scaling Up (1→44 pods):**
   - Near-instantaneous updates (milliseconds per endpoint)
   - Linear scaling time (O(n) but with very small constant factor)
   - No performance degradation during updates
   - Output in `cilium monitor` will show individual endpoint updates

**kube-proxy (iptables) Behavior:**
1. **Scaling Up (1→44 pods):**
   - Noticeable delay (seconds to tens of seconds)
   - Non-linear scaling time (O(n²) due to iptables rebuilds)
   - Possible packet loss during updates
   - `iptables -t nat -L KUBE-SERVICES -v` will show full chain rebuilds

### Methodology:

1. **Baseline Measurement:**
```bash
# For Cilium
cilium status --verbose | grep "Endpoint regeneration"

# For kube-proxy
sudo iptables -t nat -S KUBE-SVC-XXXXXX | wc -l
```

2. **Scaling Test:**
```bash
# Deploy test service
kubectl create deployment test --image=nginx
kubectl expose deployment test --port=80

# Monitor in separate terminals:
# Terminal 1 (Cilium):
cilium monitor -t policy-verdict

# Terminal 2 (kube-proxy):
watch -n 0.1 "sudo iptables -t nat -L KUBE-SERVICES -v --line-numbers | grep test"

# Terminal 3 (timing):
date; kubectl scale deployment test --replicas=44; date
```

3. **Expected Time Differences:**
- Cilium: ~50-500ms total for 44 endpoints
- kube-proxy: ~5-30 seconds for 44 endpoints

## For Rolling Restart of 44 Pods

### Expected Output Patterns:

**Cilium:**
- Smooth, incremental updates
- Minimal latency spikes
- `cilium monitor` shows individual endpoint replacements

**kube-proxy:**
- Bulk iptables updates
- Potential connection drops
- Visible in iptables counters resetting

### Methodology:

1. **Prepare Deployment:**
```bash
kubectl create deployment test --image=nginx --replicas=44
kubectl expose deployment test --port=80
```

2. **Monitoring Setup:**
```bash
# Terminal 1 (Cilium):
cilium monitor -t trace

# Terminal 2 (kube-proxy):
watch -n 0.1 "sudo iptables -t nat -L KUBE-SVC-XXXXXX -v --line-numbers"

# Terminal 3 (metrics):
kubectl top pods -l app=test --watch
```

3. **Execute Rolling Restart:**
```bash
# Time the operation
date; kubectl rollout restart deployment/test; date

# Monitor completion
kubectl rollout status deployment/test
```

4. **Expected Results:**
- **Cilium:**
  - Gradual endpoint updates visible in monitor
  - Completion time: ~10-30 seconds
  - Minimal impact on active connections

- **kube-proxy:**
  - Bulk iptables updates
  - Completion time: ~30-90 seconds
  - Possible packet loss during updates

## Key Metrics to Capture:

1. **Dataplane Update Latency:**
   - Time from API update to dataplane implementation
   - Measure with `date` commands around operations

2. **Connection Impact:**
   - Run concurrent `curl` requests during changes
   - Count failed requests

3. **Resource Usage:**
   - `kubectl top pods` for CPU/memory
   - Node CPU usage during updates

Would you like me to provide specific commands to capture these metrics quantitatively?
