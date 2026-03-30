# k8s-security-hardening

A production-grade Kubernetes security hardening toolkit demonstrating defense-in-depth across 6 security layers. Built to run locally on [Kind](https://kind.sigs.k8s.io/) — no cloud account required.

> This is not a tutorial. It is a working security system that real teams could adapt to protect their clusters.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Developer Workflow                        │
│                                                                  │
│   git push → Layer 1: CI/CD (Trivy scan + IaC lint)            │
│              └── Blocks images with HIGH/CRITICAL CVEs          │
│              └── Blocks misconfigured manifests                 │
└──────────────────────────┬──────────────────────────────────────┘
                           │ Only clean images reach the cluster
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Kubernetes Cluster (Kind + Calico)            │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Layer 2: Admission Control (Kyverno)                   │   │
│  │  └── Validates workloads at deploy time                 │   │
│  │  └── Mutates resources to enforce secure defaults       │   │
│  │  └── Generates NetworkPolicies for new namespaces       │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Layer 3: Pod Security Standards (native Kubernetes)    │   │
│  │  └── Namespace-level enforcement (baseline/restricted)  │   │
│  │  └── Blocks privileged containers, hostPath, etc.       │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Layer 4: Network Segmentation (Calico + NetworkPolicy) │   │
│  │  └── Default deny-all (zero-trust)                      │   │
│  │  └── Explicit allow per service                         │   │
│  │  └── Namespace isolation                                │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Layer 5: Runtime Detection (Falco)                     │   │
│  │  └── Custom rules for real attack patterns              │   │
│  │  └── Shell in container, privilege escalation, etc.     │   │
│  │  └── Alert → auto-remediation (pod isolation)           │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Layer 6: Secrets Management (Sealed Secrets)           │   │
│  │  └── No plaintext secrets in Git                        │   │
│  │  └── Encrypted at rest, decrypted only in-cluster       │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Security Coverage

| Layer | Tool | What it prevents |
|-------|------|-----------------|
| CI/CD | Trivy + GitHub Actions | Vulnerable images reaching the cluster |
| Admission | Kyverno | Non-compliant workloads being deployed |
| Pod Security | PSS (native) | Privileged containers, host namespace access |
| Network | Calico + NetworkPolicy | Lateral movement between pods/namespaces |
| Runtime | Falco | In-progress attacks (shell, exfiltration, escalation) |
| Secrets | Sealed Secrets | Plaintext credentials in Git |

MITRE ATT&CK coverage: see [docs/MITRE-MAPPING.md](docs/MITRE-MAPPING.md)
CIS Kubernetes Benchmark mapping: see [docs/CIS-BENCHMARK-MAPPING.md](docs/CIS-BENCHMARK-MAPPING.md)

---

## Quick Start

### Prerequisites

```bash
kind version    # >= 0.20
docker info     # Docker must be running
kubectl version --client
helm version    # >= 3.0
```

### 1. Create the cluster

```bash
kind create cluster --config kind/cluster-config.yaml
```

### 2. Install Calico CNI

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/custom-resources.yaml
kubectl wait --timeout=120s --for=condition=Ready pods --all -n calico-system
kubectl get nodes  # all nodes should be Ready
```

### 3. Verify NetworkPolicy enforcement

```bash
# A passing smoke test confirms Calico is enforcing policies, not just installed
bash scripts/setup/verify-netpol.sh
```

### 4. Install security layers

Each layer is independent. Install in order:

```bash
# Layer 2 — Admission control
helm install kyverno kyverno/kyverno -n kyverno --create-namespace -f kyverno/values.yaml
kubectl apply -f kyverno/policies/

# Layer 5 — Runtime detection
helm install falco falcosecurity/falco -n falco --create-namespace -f falco/values.yaml
kubectl apply -f falco/custom-rules/
```

> **Resource note:** This project runs on 8GB RAM (WSL2). Do not install all layers simultaneously. Each layer section documents its memory footprint.

---

## Project Structure

```
k8s-security-hardening/
├── kind/                    # Cluster bootstrap (Kind + Calico)
├── kyverno/                 # Admission control policies
│   └── policies/
│       ├── validate/        # Block non-compliant workloads
│       ├── mutate/          # Enforce secure defaults
│       └── generate/        # Auto-create NetworkPolicies
├── network-policies/        # Zero-trust network segmentation
├── pod-security/            # Pod Security Standards
├── falco/                   # Runtime threat detection
│   └── custom-rules/        # Rules mapped to MITRE ATT&CK
├── sealed-secrets/          # Encrypted secrets management
├── trivy/                   # Image scanning pipeline
├── scripts/
│   ├── setup/               # Cluster setup and verification
│   ├── audit/               # security-audit.sh (CIS scoring)
│   └── remediation/         # auto-isolate.sh (Falco → action)
├── tests/
│   ├── compliance/          # Manifests that SHOULD be admitted
│   ├── violations/          # Manifests that SHOULD be blocked
│   └── attacks/             # Attack simulations (before/after)
└── docs/
    ├── adr/                 # Architecture Decision Records
    ├── MITRE-MAPPING.md
    ├── CIS-BENCHMARK-MAPPING.md
    └── THREAT-MODEL.md
```

---

## Attack Simulations

The `tests/attacks/` directory contains realistic attack scenarios with before/after comparisons:

- Container escape attempt
- Privilege escalation via RBAC misconfiguration
- Secrets exfiltration
- Lateral movement between namespaces
- Crypto mining simulation

Each scenario shows what an attacker does, which layer detects/blocks it, and the forensic evidence captured.

---

## Design Decisions

All non-obvious architectural choices are documented as Architecture Decision Records (ADRs) in `docs/adr/`. Each ADR explains the problem, the options considered, the decision made, and the consequences.

| ADR | Decision |
|-----|----------|
| [ADR-001](docs/adr/ADR-001-cluster-setup.md) | Kind + Calico over alternatives; kubeadm hardening tradeoffs |

---

## Author

**Ghofrane Haddedi** — DevSecOps Engineer
[GitHub](https://github.com/ghofranehad) · CKA · CKS

---

## License

MIT — see [LICENSE](LICENSE)
