---
title: "Cilium — Technical Architecture"
date: 2026-05-20
tags:
  - research
  - kubernetes
  - networking
  - ebpf
  - cilium
  - architecture
aliases:
  - cilium-architecture
draft: false
---

## Component Architecture

A Cilium deployment consists of the following components running in a Kubernetes cluster:

```
┌─────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │
│  │   Node 1    │  │   Node 2    │  │   Node 3    │ │
│  │ ┌─────────┐ │  │ ┌─────────┐ │  │ ┌─────────┐ │ │
│  │ │ Agent   │ │  │ │ Agent   │ │  │ │ Agent   │ │ │
│  │ │ +Hubble │ │  │ │ +Hubble │ │  │ │ +Hubble │ │ │
│  │ │ +CNI    │ │  │ │ +CNI    │ │  │ │ +CNI    │ │ │
│  │ └─────────┘ │  │ └─────────┘ │  │ └─────────┘ │ │
│  └─────────────┘  └─────────────┘  └─────────────┘ │
│                         │                           │
│  ┌──────────────────────────────────────────────┐   │
│  │  Operator (cluster-wide)                     │   │
│  │  • IPAM  • CRD sync  • kvstore heartbeat     │   │
│  └──────────────────────────────────────────────┘   │
│                         │                           │
│  ┌──────────────────────────────────────────────┐   │
│  │  Data Store: Kubernetes CRDs (default)        │   │
│  │  or etcd (optional, for larger scale)          │   │
│  └──────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘

            ┌──────────────┐
            │ Hubble Relay │  ← cluster-wide aggregation
            └──────┬───────┘
                   │
         ┌─────────┴─────────┐
    ┌────┴────┐        ┌─────┴─────┐
    │Hubble UI│        │Hubble CLI │
    │ (GUI)   │        │ (hubble)  │
    └─────────┘        └───────────┘
```

### 1. Cilium Agent (`cilium-agent`)

Runs as a **DaemonSet on every node**. This is the core component:

- **Listens to Kubernetes API** for pod/workload lifecycle events
- **Compiles and loads eBPF programs** into the kernel for each endpoint (pod)
- **Manages security identities** — assigns numeric IDs to label groups
- **Enforces NetworkPolicy** at L3, L4, and L7 via eBPF
- **Implements load balancing** (replaces kube-proxy)
- **Exposes a REST API** for local inspection and debugging
- **Embeds the Hubble server** for per-node flow observability

The agent is the **critical path** for all pod networking. If the agent on a node fails, networking on that node is disrupted. However, existing pod-to-pod connections continue because eBPF programs keep running in the kernel.

### 2. CNI Plugin (`cilium-cni`)

A lightweight binary invoked by the **kubelet** via the Container Network Interface:

- Called when a pod is **created** → triggers eBPF endpoint setup
- Called when a pod is **deleted** → cleans up eBPF programs and maps
- Communicates with the local Cilium agent via its REST API

### 3. Cilium Operator

A **deployment** (not DaemonSet) that handles cluster-wide duties:

- **IP Address Management (IPAM)** — allocates pod CIDRs, especially in cloud environments (AWS ENI, Azure, GCP)
- **CRD synchronization** — manages `CiliumNode`, `CiliumIdentity`, `CiliumEndpoint` resources
- **kvstore heartbeat** — if using etcd, maintains liveness keys
- **Not in the data path** — cluster continues forwarding traffic if the operator is down (only new pod scheduling may be delayed)

### 4. Hubble Observability Stack

| Component | Location | Role |
|---|---|---|
| **Hubble Server** | Per-node (embedded in agent) | Collects eBPF flow events, exposes gRPC + Prometheus |
| **Hubble Relay** | Cluster-wide (Deployment) | Aggregates flows from all Hubble servers, provides unified gRPC API |
| **Hubble CLI** | Local/remote | Queries relay or local server for flow logs |
| **Hubble UI** | Cluster-wide (Deployment) | Web-based service dependency graph and flow browser |

### 5. Data Store

Cilium needs a data store for **state propagation** between agents:

- **Default: Kubernetes CRDs** — No external dependencies. Sufficient for most clusters.
- **Optional: etcd** — For clusters with 5,000+ nodes or very high pod churn, etcd provides more efficient change notifications and lower API server load.

---

## Identity Model

This is Cilium's most important architectural concept.

### Problem

Traditional firewalls (iptables, network ACLs) filter by **source IP** and **destination port**. In Kubernetes:
- Pod IPs are ephemeral (seconds to hours)
- A single port (e.g., 80) serves many different logical services
- IP addresses tell you nothing about *what* the workload is

### Cilium's Solution

Cilium assigns a **numeric security identity** to every *group of pods that share the same security-relevant labels*. For example:

```
Labels: app=frontend, env=prod    → Identity: 12345
Labels: app=backend, env=prod     → Identity: 67890
Labels: app=database, env=prod    → Identity: 11111
```

These identities are:
- **Deterministic** — same labels → same identity across the cluster
- **Propagated** — stored in etcd/CRDs, synced to all agents
- **Encoded in every packet** — identity is embedded in packet metadata (VXLAN, Geneve, or kernel marks in native routing)

### Policy Enforcement Flow

```
Pod A (identity: 12345) ──► eBPF on source node
                                │
                                ▼ encodes identity in packet
                            Network (VXLAN/Geneve/native)
                                │
                                ▼
                            eBPF on dest node
                                │
                                ▼ looks up identity in policy hash table
                            ALLOW / DENY (O(1) hash table lookup)
                                │
                                ▼
                            Pod B (identity: 67890)
```

Key insight: The **receiving node** enforces policy by checking the *source identity* against known policies — no need to know the source IP, no need for distributed firewall rule updates.

---

## Data Flow: Life of a Packet

See the dedicated [[cilium-ebpf-datapath]] page for the full eBPF packet walkthrough. Here's the architectural summary:

### Endpoint to Endpoint (same node)

```
Pod A → veth (host) → TC ingress eBPF → endpoint policy → redirect → veth (Pod B) → Pod B
```

With L7 policy (HTTP filter):
```
Pod A → veth → TC ingress → endpoint policy → proxy (Envoy per node) → L7 check
                                                      │
                                                      ▼ ALLOW
                                              redirect → veth → Pod B
```

With **socket-level enforcement** (performance optimization for TCP):
```
Pod A → connect() → eBPF socket hook → redirect directly to Pod B's socket
          ↑                                  (bypasses TC/stack for ESTABLISHED connections)
```

### Egress to External (North-South)

```
Pod A → veth → TC ingress → eBPF → overlay encap (VXLAN) or native route
                                          │
                                          ▼
                                    eth0 → internet
```

### Ingress from External (North-South)

```
eth0 → XDP prefilter (optional, high-perf drop) → TC ingress → overlay decap
                                                                    │
                                                                    ▼
                                                              veth → Pod A
```

---

## Performance Characteristics

| Aspect | Cilium (eBPF) | Traditional (iptables) |
|---|---|---|
| Policy lookup | O(1) hash table | O(n) linear chain |
| Load balancing | Socket-level redirect (no per-packet NAT) | Per-packet DNAT |
| Service scale | 10,000+ services tested | Degrades at 5,000+ rules |
| Connection tracking | eBPF CT maps (hash-based) | conntrack table (global lock) |
| Encryption overhead | In-kernel IPSec/WireGuard (no proxy) | Sidecar TLS termination |

---

## References

- [Cilium Component Overview](https://docs.cilium.io/en/stable/overview/component-overview/)
- [Cilium Security Identity Model](https://docs.cilium.io/en/stable/security/policy/#security-identity)
- [BPF and XDP Reference Guide](https://docs.cilium.io/en/stable/bpf/)
