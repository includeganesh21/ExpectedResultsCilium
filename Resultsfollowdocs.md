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


----
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


