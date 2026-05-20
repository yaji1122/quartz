---
title: "Cilium — eBPF-Powered Kubernetes Networking"
date: 2026-05-20
tags:
  - research
  - kubernetes
  - networking
  - ebpf
  - cilium
  - cncf
draft: false
---

## Overview

**Cilium** is an open-source (Apache 2.0 / MIT dual-licensed) CNCF Incubating project that provides networking, security, and observability for cloud-native workloads. Built on **eBPF** (extended Berkeley Packet Filter), it operates transparently inside the Linux kernel — no application changes, no sidecar proxies required.

### What makes Cilium special?

| Traditional (iptables/IPVS) | Cilium (eBPF) |
|---|---|
| IP address + port based | **Identity based** (Kubernetes labels) |
| Rule explosion at scale | **eBPF hash tables** — O(1) lookup |
| L3/L4 only | **L3–L7** (HTTP, gRPC, Kafka, DNS) |
| Per-packet NAT overhead | **Socket-level** load balancing |
| Sidecar proxies (Envoy per pod) | **Sidecar-free** service mesh (per-node Envoy) |
| Separate CNI + policy + mesh tools | **Unified** CNI, policy, LB, mesh, observability |

### Core Capabilities at a Glance

- **CNI Plugin** — Overlay (VXLAN/Geneve), native routing, BGP
- **NetworkPolicy** — Identity-aware L3–L7 enforcement with DNS, HTTP, gRPC, Kafka filtering
- **Load Balancing** — Replaces kube-proxy; east-west (socket-level) + north-south (XDP/DSR)
- **Service Mesh** — Gateway API, mTLS (IPSec/WireGuard), L7-aware, sidecar-free
- **Cluster Mesh** — Multi-cluster connectivity with global services and unified identity
- **Hubble** — Deep observability: service maps, flow logs, DNS tracking, Prometheus metrics

### Why eBPF?

eBPF allows Cilium to run sandboxed programs directly in the Linux kernel at key hook points (XDP, TC, sockets). This means:
- **No kernel modules** — safe bytecode verified by the kernel
- **No context switches** — in-kernel processing = near line-rate performance
- **Dynamic updates** — policies change without restarting workloads
- **Deep visibility** — every packet, every connection, every DNS lookup is observable

---

## Key Findings

1. **Identity, not IP** — Cilium assigns a *security identity* (numeric label) to groups of pods sharing the same security context. Policies reference these identities instead of ephemeral IP addresses. This is the architectural cornerstone.

2. **eBPF replaces iptables** — Instead of linear rule chains (iptables) that become a bottleneck at scale, Cilium uses eBPF hash tables for O(1) policy lookup. Production clusters run 10,000+ policies without degradation.

3. **Socket-level load balancing** — East-west traffic is intercepted at `connect()` time via eBPF, avoiding per-packet NAT. This is what enables Cilium to fully replace kube-proxy.

4. **Sidecar-free service mesh** — Cilium runs one Envoy proxy *per node* (not per pod), dramatically reducing resource overhead. mTLS is handled in-kernel via IPSec or WireGuard.

5. **Hubble is more than a network observability tool** — It's a security observability platform. You can trace HTTP calls, Kafka topics, DNS resolutions, TLS handshakes, and dropped packets — all with pod identity metadata.

---

## Details

See the linked pages for deep dives:

- [[cilium-architecture]] — Component architecture, agent internals, identity model, data flow
- [[cilium-ebpf-datapath]] — eBPF hooks, life of a packet, maps, socket-level enforcement

## References

- [Cilium Documentation](https://docs.cilium.io/en/stable/)
- [Cilium GitHub](https://github.com/cilium/cilium) (~24k stars)
- [eBPF.io](https://ebpf.io/) — What is eBPF?
- [CNCF Cilium](https://www.cncf.io/projects/cilium/) — Incubating project page
- [Cilium Slack](https://slack.cilium.io/)
- [eCHO Episode 51: Life of a Packet](https://www.youtube.com/watch?v=0BKUZA-7UyY)
- [Cilium Security Audit 2022](https://cilium.io/blog/2022/03/31/cilium-security-audit-2022/)

## Next Steps

- [ ] Try the [Cilium Star Wars Demo](https://docs.cilium.io/en/stable/gettingstarted/demo/)
- [ ] Compare with Calico, Flannel, and Istio in a [[k8s-cni-comparison]] page
- [ ] Benchmark eBPF vs iptables at scale (1,000+ services)
- [ ] Explore Cilium's [Gateway API](https://docs.cilium.io/en/stable/network/servicemesh/gateway-api/) integration
