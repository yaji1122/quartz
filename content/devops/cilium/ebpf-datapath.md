---
title: "Cilium — eBPF Datapath Deep Dive"
date: 2026-05-20
tags:
  - research
  - kubernetes
  - ebpf
  - cilium
  - datapath
  - kernel
aliases:
  - cilium-ebpf-datapath
draft: false
---

## eBPF Hooks Used by Cilium

Cilium attaches BPF programs at multiple hook points in the Linux kernel networking stack. Each hook has different characteristics in terms of performance and available packet metadata.

```
                    ┌──────────────────────────────┐
    Incoming        │         Network Driver        │
    Packet ────────►│              │                │
                    │         ┌────▼─────┐          │
                    │         │ XDP Hook │ ← earliest, raw packet
                    │         └────┬─────┘          │
                    │              │                │
                    │    ┌─────────▼──────────┐     │
                    │    │  Network Stack      │     │
                    │    │  (skb allocated)    │     │
                    │    └─────────┬──────────┘     │
                    │              │                │
                    │    ┌─────────▼──────────┐     │
                    │    │ TC Ingress Hook     │     │
                    │    │ (cilium_host, veth) │     │
                    │    └─────────┬──────────┘     │
                    │              │                │
                    │    ┌─────────▼──────────┐     │
                    │    │  L3/L4 Processing   │     │
                    │    └─────────┬──────────┘     │
                    │              │                │
                    │    ┌─────────▼──────────┐     │
                    │    │ Socket Operations   │ ← cgroup attach
                    │    │ Socket Send/Recv    │ ← per-socket
                    │    └─────────────────────┘     │
                    └──────────────────────────────┘
```

### 1. XDP (eXpress Data Path)

- **Hook point:** Earliest possible in the network driver — before `sk_buff` allocation
- **Use in Cilium:** **Prefilter** — high-performance packet filtering/dropping
- **Ideal for:** DDoS mitigation, dropping known-bad CIDRs at line rate
- **Performance:** ~24 Mpps per core (no skb allocation overhead)
- **Limitation:** Limited to ingress only, no access to full kernel networking stack metadata

### 2. TC (Traffic Control) Ingress/Egress

- **Hook point:** After initial stack processing, before L3 routing decision
- **Use in Cilium:** **Primary datapath** — attached to `cilium_host`, `cilium_net`, and host-side veth pairs
- **What it does:**
  - **Endpoint policy enforcement** — check identity, apply L3/L4 rules
  - **Service load balancing** — DNAT, Maglev consistent hashing
  - **Encapsulation/decapsulation** — VXLAN, Geneve
  - **Traffic redirection** — to local endpoints, to proxy (Envoy), to overlay
- **Why TC over XDP for most cases:** TC has access to full `sk_buff` metadata and can modify, redirect, or encapsulate packets. XDP is faster but more limited.

### 3. Socket Operations Hook

- **Hook point:** Attached to the **root cgroup**, fires on TCP state transitions
- **Use in Cilium:** Monitors for TCP `ESTABLISHED` state transitions
- **When a socket becomes ESTABLISHED:**
  - If the peer is **node-local** (same-node pod or local proxy), Cilium attaches a socket send/recv program
  - This enables **socket-level acceleration** (bypassing the full TC path for established connections)

### 4. Socket Send/Recv Hook

- **Hook point:** Every `sendmsg()` / `recvmsg()` call on the attached socket
- **Use in Cilium:** **Datapath acceleration** — for TCP connections between local endpoints
- **What it does:**
  - Inspect the message
  - **Drop** it (policy deny)
  - **Send** it to TCP layer normally
  - **Redirect** it directly to the destination socket (bypassing the full network stack)
- **Result:** Dramatically lower latency and CPU overhead for local pod-to-pod traffic

---

## Virtual Interfaces

Cilium creates several virtual network interfaces on each node:

| Interface | Type | Purpose |
|---|---|---|
| `cilium_host` | veth (host side) | Local stack endpoint — where packets enter/leave the kernel's IP stack for Cilium-managed traffic |
| `cilium_net` | veth (netns side) | Paired with `cilium_host`, enables routing between Cilium's datapath and the host network stack |
| `cilium_vxlan` | VXLAN tunnel | Overlay interface when using encapsulation mode. All cross-node pod traffic is encapsulated here. |
| `lxc_XXXX` | veth (host side) | One per pod — the host-side of each pod's veth pair. TC ingress BPF programs are attached here. |

### How They Work Together

```
Pod (eth0) ←→ veth pair ←→ lxc_XXXX (host side, TC BPF attached)
                                │
                                ▼
                         cilium_host ←→ cilium_net ←→ host routing table
                                │
                                ▼
                         cilium_vxlan (if overlay mode)
                                │
                                ▼
                             eth0 (physical)
```

---

## eBPF Maps

eBPF programs cannot share state through normal memory. Instead, they use **BPF maps** — key-value data structures shared between BPF programs and userspace (the Cilium agent).

Key maps in Cilium's datapath:

| Map Name | Type | Purpose |
|---|---|---|
| `cilium_policy_XXXX` | Hash | Per-endpoint policy lookup — maps source identity → action (allow/deny) |
| `cilium_lb4_services_v2` | Hash | Service load balancer — maps service VIP:port → backend list |
| `cilium_lb4_backends_v2` | Hash | Backend pool — maps backend ID → IP:port |
| `cilium_ct4_global` | LRU Hash | Connection tracking — NAT state, TCP sequence adjustments |
| `cilium_ipcache` | Hash | IP → identity cache — quick identity lookup for received packets |
| `cilium_tunnel_map` | Hash | VXLAN/Geneve tunnel endpoint mapping — destination IP → tunnel endpoint |
| `cilium_metrics` | Per-CPU Array | Datapath metrics (packets, bytes, drops) |

### Map Lifecycle

1. **Agent creates maps** when an endpoint (pod) is provisioned
2. **Agent updates maps** when policies change (add/remove rules, modify backends)
3. **BPF programs read** maps on every packet — all lookups are O(1) hash table operations
4. **Agent destroys maps** when the endpoint is removed

---

## Life of a Packet (Three Scenarios)

### Scenario 1: Endpoint to Endpoint (Same Node, No Proxy)

```
Pod A (10.0.1.5, identity: 12345)
  │
  ▼ sends packet to Pod B (10.0.1.6)
lxc_AAAA (veth host side)
  │
  ▼ TC ingress BPF:
  │  1. Lookup destination IP in ipcache → identity: 67890 (Pod B)
  │  2. Lookup (src=12345, dst=67890) in policy map → ALLOW
  │  3. Redirect to lxc_BBBB (Pod B's veth)
  │
  ▼
Pod B (receives packet directly, no kernel stack traversal)
```

### Scenario 2: Endpoint to Endpoint (With L7 HTTP Policy)

```
Pod A → lxc_AAAA → TC ingress
  │
  ▼ BPF: L3/L4 ALLOW, but L7 rule exists → redirect to proxy
Envoy (per-node, listening on localhost)
  │
  ▼ HTTP filter: check method=GET, path=/public/* → ALLOW
  │
  ▼ redirect back to BPF → lxc_BBBB → Pod B
```

**Socket-level optimization:** After the TCP handshake completes (ESTABLISHED), the socket operations hook can attach a send/recv program that redirects data directly from Pod A's socket to Pod B's socket, **bypassing TC and Envoy entirely** for the data path.

### Scenario 3: Pod to External (Egress with DNS Policy)

```
Pod A → lxc_AAAA → TC ingress BPF
  │
  ▼ 1. Lookup destination: 8.8.8.8:53 (DNS)
  │ 2. Policy check: DNS allowed to *.google.com → record DNS query
  │ 3. Forward to host stack → eth0 → internet
  │
  ▼ (later) Pod A tries api.example.com:443
  │ 4. DNS proxy (per-node) intercepts DNS → resolves to 1.2.3.4
  │ 5. DNS response triggers dynamic IP cache update: 1.2.3.4 → policy ALLOW
  │ 6. Pod A's TCP SYN → TC BPF → ipcache hit → policy ALLOW → forward
```

---

## Iptables Usage in Cilium

Despite being "the iptables replacement," Cilium still uses iptables in a **minimal, non-critical role**:

- **kube-proxy interoperability** — during migration, Cilium can coexist with kube-proxy
- **Host-level rules** — a small set of rules to ensure host-to-pod traffic routes correctly
- **Fallback** — if eBPF is not available (ancient kernel), Cilium can degrade to iptables mode

In normal operation with eBPF, **the datapath does not touch iptables at all** for pod-to-pod or service traffic.

---

## Key Takeaways

1. **XDP for prefiltering** — line-rate DDoS protection at the driver level
2. **TC ingress for the main datapath** — policy, LB, encap, redirects all happen here
3. **Socket hooks for acceleration** — for TCP, established connections bypass the full datapath
4. **eBPF maps = O(1)** — policy lookup, service lookup, connection tracking are all hash-table based
5. **No iptables in the critical path** — unlike kube-proxy/Calico, Cilium's datapath avoids the linear rule chain bottleneck entirely

---

## References

- [Cilium eBPF Datapath Introduction](https://docs.cilium.io/en/stable/network/ebpf/intro/)
- [Life of a Packet in Cilium](https://docs.cilium.io/en/stable/network/ebpf/life-of-a-packet/)
- [BPF and XDP Reference Guide](https://docs.cilium.io/en/stable/bpf/)
- [eCHO Episode 51: Life of a Packet with Cilium](https://www.youtube.com/watch?v=0BKUZA-7UyY)
