
# nftables vs iptables: Performance Benchmark Summary

This document summarizes real-world benchmark findings and comparisons between `nftables` and `iptables`.

---

## ðŸ”¬ 1. Red Hat Developer Benchmark (2017)

In a comprehensive study by Red Hat, it was observed that:

- `iptables` performance degrades linearly with the number of custom chains.
- `nftables` scales more efficiently and offers better performance as the rule-set size grows.
- With 50 rule jumps, `nftables` already outperforms `iptables` in mean performance.

ðŸ”— **Link**: [Red Hat Benchmark Article](https://developers.redhat.com/blog/2017/04/11/benchmarking-nftables)

---

## ðŸ“Š 2. Academic Study: Latency and Throughput Comparison

A detailed academic paper compared the latency and throughput of `iptables` and `nftables` using various rule-set sizes and frame sizes.

Key findings:
- With linear look-ups, `nftables` performs worse than `iptables` for small frame sizes and large rule sets.
- When using indexed data structures, `nftables` performance improves and can outperform `iptables` in certain scenarios.

ðŸ”— **Link**: [Full Academic Paper (PDF)](https://www.diva-portal.org/smash/get/diva2%3A1212650/FULLTEXT01.pdf)

---

## ðŸ§  Key Takeaways

- **Data Structures**: `nftables` leverages efficient data structures (e.g., sets, maps) allowing for faster rule matching.
- **Scalability**: In environments with large and complex rule sets, `nftables` offers better scalability and performance.
- **Modern Features**: Provides a unified framework for IPv4/IPv6, stateful filtering, and simplified firewall management.

---

Feel free to explore the linked resources for deeper technical analysis.
