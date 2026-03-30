# ADR-001: Cluster Setup — Kind + Calico CNI

**Date:** 2026-03-30
**Status:** Accepted

---

## Context

This project needs a local Kubernetes cluster that:
- Runs on 8GB RAM (WSL2 on Windows)
- Supports real NetworkPolicy enforcement (required for Layer 4)
- Costs nothing
- Is reproducible from a single config file

---

## Decision 1: Kind over alternatives

| Option | Reason rejected |
|--------|----------------|
| minikube | Single-node only; NetworkPolicy support varies by driver |
| k3s | Replaces too many components (etcd, CNI, ingress) — hard to reason about security surface |
| k0s | Less community tooling; fewer CKS exam parallels |
| **Kind** | Multi-node, uses real kubeadm + containerd, matches how production clusters bootstrap, wide ecosystem support |

---

## Decision 2: Calico over other CNIs

Kind's default CNI is **kindnet**. It handles pod networking but does **not** implement the `NetworkPolicy` API. Policies are accepted by the API server and silently ignored — a false sense of security.

| CNI | NetworkPolicy support | Notes |
|-----|----------------------|-------|
| kindnet (default) | None | Silently ignores policies |
| Flannel | Partial (requires separate policy engine) | Extra complexity |
| Cilium | Full (eBPF) | Excellent, but requires kernel >= 5.10 and more RAM |
| **Calico** | Full (iptables/eBPF) | Stable, well-documented, CKS exam standard |

Calico was chosen because it is the most commonly referenced CNI in CKS exam documentation and production hardening guides, and it runs comfortably within the 8GB RAM constraint.

---

## Decision 3: Kubeadm hardening flags NOT applied at bootstrap

The CIS Kubernetes Benchmark recommends several API server and kubelet hardening flags:

```
anonymous-auth=false
authorization-mode=Webhook (kubelet)
audit-log-path=/var/log/kubernetes/audit.log
```

These were intentionally **not** applied via `kubeadmConfigPatches` in `kind/cluster-config.yaml`.

**Why:** During `kubeadm join`, worker nodes perform anonymous preflight health checks against the API server (`GET /healthz`). With `anonymous-auth=false`, these return `401 Unauthorized` and the join fails. Similarly, forcing `authorizationMode: Webhook` on kubelets before they have a registered client certificate creates a bootstrap deadlock.

**Production equivalent:** In managed clusters (EKS, GKE, RKE2), these flags are applied by the control plane provider after bootstrap is complete. In self-managed clusters, they are applied post-join via configuration management (Ansible, cloud-init). The flags themselves are correct — only the timing is constrained by kubeadm's bootstrap sequence.

**What this project does instead:**
- Documents the recommended production flags here
- Demonstrates the compensating controls (Kyverno, NetworkPolicy, Falco) that enforce equivalent security at the workload level
- The `security-audit.sh` script will flag these missing settings with their CIS benchmark IDs

---

## Consequences

- NetworkPolicies are fully enforced (Calico verified via smoke test)
- Cluster is reproducible with a single command
- API server hardening flags require a post-bootstrap configuration step not implemented in this local environment
- All workload-level security layers are unaffected by this constraint
