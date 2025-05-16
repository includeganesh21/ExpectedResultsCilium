What is Cilium?
Open-source CNI (Container Network Interface) plugin.
Uses eBPF to provide networking, security, and observability.
Key Cilium Capabilities (relevant to your problems):
eBPF-powered Kube-Proxy Replacement: This is your focus!
Identity-Based Security (Network Policies).
Advanced Load Balancing.
Transparent Encryption.
Rich Network Observability (Hubble).


I have a eks cluster and I have installed cilium as primary CNI in it 
also I want to compare the performance in term of latency but issue is difference is small and even affected by physical location of nodes if I take whole RTT of packet 
how can I get time spent in 1. updates using ebpf maps in case of cilium and the iptable traversal in case of kube-proxy default scenario

also 2nd compare.
time to update the entry of service to pod in ebpf map and that of in iptable in kube proxy

for 2nd compare I nthought of rolling restart and scaling. up and down gibe how to do this and also expected results
