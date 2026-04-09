# CIS Kubernetes Benchmark Mapping

Maps the security controls in this project to the [CIS Kubernetes Benchmark v1.9](https://www.cisecurity.org/benchmark/kubernetes).

---

## Coverage Summary

| Section | Title | Coverage |
|---------|-------|----------|
| 1 | Control Plane Components | Partial |
| 2 | etcd | Not applicable (Kind managed) |
| 3 | Control Plane Configuration | Partial |
| 4 | Worker Node Configuration | Partial |
| 5 | Policies | Full |

---

## Section 1 — Control Plane

| CIS ID | Recommendation | Control | Status |
|--------|---------------|---------|--------|
| 1.2.1 | Ensure anonymous-auth is not enabled | Kind managed — not configurable without bootstrap deadlock (ADR-001) | Partial |
| 1.2.6 | Ensure AlwaysPullImages admission plugin is enabled | Kyverno image verification policy (Audit) | Partial |
| 1.2.16 | Ensure audit logging is enabled | Kind default — not customized | Not implemented |

---

## Section 3 — Control Plane Configuration

| CIS ID | Recommendation | Control | Status |
|--------|---------------|---------|--------|
| 3.2.1 | Ensure logging is configured | Falco runtime logging via JSON | Implemented |
| 3.2.2 | Ensure audit logs capture pod creation | Kind managed | Partial |

---

## Section 4 — Worker Node

| CIS ID | Recommendation | Control | Status |
|--------|---------------|---------|--------|
| 4.2.1 | Ensure anonymous-auth is not enabled on kubelet | Kind default | Partial |
| 4.2.6 | Ensure streaming connection timeout is not set to 0 | Kind default | Partial |

---

## Section 5 — Policies

| CIS ID | Recommendation | Control | Status |
|--------|---------------|---------|--------|
| 5.1.1 | Ensure cluster-admin role is used only where required | ArgoCD AppProject scoping | Implemented |
| 5.1.6 | Ensure service account tokens are not auto-mounted where not necessary | Kyverno — future policy | Planned |
| 5.2.1 | Ensure privileged containers are not admitted | Kyverno block-privileged-containers (Deny) | Implemented |
| 5.2.2 | Ensure host PID namespace sharing is not admitted | PSS restricted | Implemented |
| 5.2.3 | Ensure host IPC namespace sharing is not admitted | PSS restricted | Implemented |
| 5.2.4 | Ensure host network namespace sharing is not admitted | PSS restricted | Implemented |
| 5.2.5 | Ensure privileged escalation is not permitted | Kyverno mutation + PSS restricted | Implemented |
| 5.2.6 | Ensure root containers are not admitted | Kyverno block-root-containers (Deny) | Implemented |
| 5.2.7 | Ensure NET_RAW capability is restricted | Kyverno capabilities.drop ALL | Implemented |
| 5.2.8 | Ensure added capabilities are restricted | Kyverno capabilities.drop ALL | Implemented |
| 5.2.9 | Ensure containers run with read-only root filesystem | Kyverno require-readonly-rootfs (Deny) | Implemented |
| 5.3.1 | Ensure network policies are in place | Kyverno GeneratingPolicy + NetworkPolicies | Implemented |
| 5.3.2 | Ensure all namespaces have NetworkPolicies | GeneratingPolicy auto-creates on namespace creation | Implemented |
| 5.4.1 | Prefer secrets as files over environment variables | Sealed Secrets + GitOps workflow | Implemented |
| 5.4.2 | Consider external secret storage | Sealed Secrets (ADR-005 documents ESO as production alternative) | Implemented |
| 5.7.1 | Create administrative boundaries between resources using namespaces | ArgoCD AppProject + namespace PSS labels | Implemented |
| 5.7.2 | Ensure seccomp profile is set to docker/default or runtime/default | Kyverno mutation injects RuntimeDefault | Implemented |
| 5.7.3 | Apply SecurityContext to pods and containers | Kyverno MutatingPolicy injects full securityContext | Implemented |
| 5.7.4 | Ensure default namespace is not used | Enforced by team convention + ArgoCD app namespacing | Partial |

---

## Gaps

| CIS ID | Recommendation | Reason not implemented |
|--------|---------------|----------------------|
| 1.2.16 | Audit logging | Kind does not expose audit log configuration without bootstrap issues |
| 5.1.6 | Disable automounting service account tokens | Planned — future Kyverno policy |
| 5.6.1 | RBAC least privilege | ArgoCD scoped — full RBAC audit not in scope for this project |

---

## Score Estimate

Based on Section 5 (Policies) — the most relevant section for a workload security project:

**~85% of applicable CIS Section 5 controls are implemented.**

Full CIS benchmark scoring requires access to kube-bench. To run it:

```bash
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml
kubectl logs job/kube-bench
```
