# ADR-003: Why Kyverno over OPA/Gatekeeper

## Context

Admission control is required to validate and mutate workloads at deploy time.
The two main options in the Kubernetes ecosystem are Kyverno and OPA/Gatekeeper.

## Options Considered

### OPA/Gatekeeper
- Policies written in Rego — a purpose-built policy language
- Mature, widely adopted in enterprise environments
- Strong separation between policy engine (OPA) and Kubernetes integration (Gatekeeper)
- Mutation support is limited and complex
- Rego has a steep learning curve — it is not familiar to most Kubernetes engineers
- No built-in generate capability (auto-creating resources)

### Kyverno
- Policies written in YAML + CEL — native Kubernetes style
- Purpose-built for Kubernetes — no separate language to learn
- Full mutate, validate, and generate support in a single tool
- New API (policies.kyverno.io/v1) uses CEL — same language as Kubernetes ValidatingAdmissionPolicy
- Active development, strong community, CNCF project

## Decision

**Kyverno.**

## Reasons

1. **Kubernetes-native policy language** — Kyverno policies look like Kubernetes manifests. Any engineer who can write a Deployment can read a Kyverno policy. Rego requires dedicated training.

2. **CEL alignment** — Kyverno 3.7 uses CEL for the new API, which is the same expression language Kubernetes itself uses for ValidatingAdmissionPolicy. This is the direction the ecosystem is moving.

3. **Generate capability** — Kyverno can auto-generate resources (NetworkPolicies, ConfigMaps) when namespaces are created. OPA/Gatekeeper has no equivalent.

4. **Mutation** — Kyverno's mutate policies are first-class and well-documented. Gatekeeper's mutation support is experimental and limited.

5. **Portfolio relevance** — Kyverno adoption is growing faster than Gatekeeper in 2025-2026 job postings. CKS exam references Kyverno in its curriculum.

## Trade-offs

| Concern | Impact |
|---------|--------|
| Rego is more expressive for complex logic | Not relevant for this project's policy set |
| OPA can be used outside Kubernetes | Out of scope |
| Gatekeeper is more mature in large enterprises | Acceptable for a portfolio/lab environment |

## Status

Decided — Kyverno 3.7.1 deployed via ArgoCD.
