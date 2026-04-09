# ADR-004: Why Calico over Cilium/Flannel

## Context

Kind disables its default CNI when `disableDefaultCNI: true` is set.
A CNI must be chosen that supports NetworkPolicy enforcement — the default
Kind CNI does not enforce NetworkPolicies.

## Options Considered

### Flannel
- Simple overlay network, easy to install
- Does NOT enforce NetworkPolicies — requires an additional plugin (Calico or Canal)
- No eBPF support
- Eliminated immediately: NetworkPolicy is a core requirement of this project

### Cilium
- eBPF-based CNI — highest performance, deepest observability
- Native NetworkPolicy + extended CiliumNetworkPolicy
- Excellent Hubble observability layer
- Complex installation and configuration
- High memory footprint — problematic on 8GB RAM WSL2
- Requires kernel >= 4.19, works best on >= 5.10

### Calico
- Mature, production-proven CNI
- Full NetworkPolicy enforcement (standard + Calico-extended policies)
- Supports eBPF dataplane (optional) — standard dataplane works on all kernels
- Lower memory footprint than Cilium
- Well-documented Kind integration
- CNCF project

## Decision

**Calico with standard dataplane.**

## Reasons

1. **NetworkPolicy enforcement** — Calico enforces standard Kubernetes NetworkPolicies out of the box. Flannel does not.

2. **RAM constraint** — Cilium's memory footprint is significantly higher than Calico. On 8GB RAM WSL2 with a 3-node Kind cluster, Cilium causes resource pressure. Calico runs comfortably.

3. **Maturity** — Calico is battle-tested in large production clusters. For a security-focused project, a stable CNI is more important than cutting-edge features.

4. **Kind compatibility** — Calico has official Kind documentation and a well-known installation procedure (`tigera-operator` + `custom-resources.yaml`).

## Trade-offs

| Concern | Impact |
|---------|--------|
| Cilium has deeper eBPF observability (Hubble) | Falco covers runtime observability in this project |
| Cilium's CiliumNetworkPolicy is more expressive | Standard NetworkPolicy is sufficient for this project |
| Calico has less native L7 policy support | Out of scope |

## Status

Decided — Calico v3.28 deployed via tigera-operator.
