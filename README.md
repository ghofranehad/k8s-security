# k8s-security

A production-grade Kubernetes security hardening project demonstrating defense-in-depth across 8 security layers. All layers are deployed and managed via GitOps with ArgoCD. Built to run locally on [Kind](https://kind.sigs.k8s.io/) — no cloud account required.

> This is not a tutorial. It is a working security system that real teams could adapt to protect their clusters.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Developer Workflow                        │
│                                                                  │
│   git push → Layer 1: CI/CD (Trivy)                            │
│              └── Blocks images with HIGH/CRITICAL CVEs          │
│              └── Blocks misconfigured manifests                 │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│              Layer 2: GitOps (ArgoCD)                           │
│              └── Single source of truth — this repository       │
│              └── Automated sync — cluster matches Git state     │
│              └── AppProject scoping — source repo restriction   │
└──────────────────────────┬──────────────────────────────────────┘
                           │ All layers deployed and reconciled by ArgoCD
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│         Layer 0: Infrastructure (Kind + Calico CNI)             │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Layer 3: Admission Control (Kyverno)                   │   │
│  │  └── Mutate: inject secure defaults into every pod      │   │
│  │  └── Validate: deny non-compliant workloads             │   │
│  │  └── Generate: auto-create NetworkPolicies per ns       │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Layer 4: Pod Security Standards (native Kubernetes)    │   │
│  │  └── restricted: app namespaces                         │   │
│  │  └── baseline:   infra namespaces                       │   │
│  │  └── privileged: security tools (Falco)                 │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Layer 5: Network Segmentation (Calico + NetworkPolicy) │   │
│  │  └── Default deny-all generated per namespace           │   │
│  │  └── Explicit allow rules per workload                  │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Layer 6: Runtime Detection (Falco)                     │   │
│  │  └── Shell spawned in container                         │   │
│  │  └── Sensitive file read                                │   │
│  │  └── Unexpected outbound connection                     │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Layer 7: Secrets Management (Sealed Secrets)           │   │
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
| CI/CD | Trivy + GitHub Actions | Vulnerable images and misconfigured manifests reaching the cluster |
| GitOps | ArgoCD | Unauthorized changes — cluster state always matches Git |
| Admission | Kyverno | Root containers, privileged pods, missing limits, writable rootfs |
| Pod Security | PSS (native) | Privileged containers, host namespace access, unsafe volume types |
| Network | Calico + NetworkPolicy | Lateral movement between pods and namespaces |
| Runtime | Falco | In-progress attacks — shell spawn, credential access, exfiltration |
| Secrets | Sealed Secrets | Plaintext credentials committed to Git |

---

## Quick Start

### Prerequisites

> **WSL2 users:** Before creating the cluster, raise inotify limits or Falco will fail on control-plane nodes:
> ```bash
> echo "fs.inotify.max_user_instances=1024" | sudo tee -a /etc/sysctl.conf
> echo "fs.inotify.max_user_watches=1048576" | sudo tee -a /etc/sysctl.conf
> sudo sysctl -p
> ```

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
```

### 3. Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --timeout=120s --for=condition=Ready pods --all -n argocd
```

### 4. Bootstrap GitOps

```bash
kubectl apply -f argocd/projects/security-project.yaml
kubectl apply -f argocd/apps/main.yaml
```

ArgoCD will automatically deploy all security layers from this repository. Access the UI:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Username: admin
# Password:
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath='{.data.password}' | base64 -d
```

---

## Project Structure

```
k8s-security/
├── .github/workflows/       # Layer 1 — Trivy image + manifest scanning
├── kind/                    # Layer 0 — Cluster bootstrap (Kind + Calico)
├── argocd/
│   ├── apps/                # Layer 2 — ArgoCD Applications (app-of-apps)
│   └── projects/            # Layer 2 — AppProject (source restriction)
├── kyverno/
│   └── policies/
│       ├── validate/        # Layer 3 — Deny non-compliant workloads
│       ├── mutate/          # Layer 3 — Inject secure defaults
│       └── generate/        # Layer 3 — Auto-create NetworkPolicies
├── pod-security/            # Layer 4 — PSS namespace labels
├── network-policies/        # Layer 5 — Explicit allow rules per workload
├── falco/                   # Layer 6 — Runtime detection + custom rules
├── sealed-secrets/          # Layer 7 — Encrypted secrets
├── tests/
│   └── violations/          # Manifests that test policy enforcement
└── docs/
    └── adr/                 # Architecture Decision Records
```

---

## Policy Details

### Kyverno — Mutating Policies
| Policy | Effect |
|--------|--------|
| `add-default-securitycontext` | Injects `runAsNonRoot`, `runAsUser: 1000`, `readOnlyRootFilesystem`, `allowPrivilegeEscalation: false`, `capabilities.drop: ALL`, `seccompProfile: RuntimeDefault` into containers and initContainers |
| `add-resource-limits` | Injects `cpu: 100m`, `memory: 256Mi` limits into containers and initContainers |

Infra namespaces (kyverno, argocd, falco, calico-system, etc.) are excluded.

### Kyverno — Validating Policies
| Policy | Action |
|--------|--------|
| `block-root-containers` | Deny pods with `runAsNonRoot != true` |
| `block-privileged-containers` | Deny pods with `privileged: true` |
| `require-resource-limits` | Deny pods without CPU and memory limits |
| `require-readonly-rootfs` | Deny pods with writable root filesystem |

### Falco — Custom Rules
| Rule | Detects |
|------|---------|
| Shell Spawned in Container | `sh`, `bash`, `zsh` executed inside a running container |
| Sensitive File Read | Access to `/etc/shadow`, `/etc/passwd`, `/root/.ssh/*` |
| Unexpected Outbound Connection | Outbound traffic on non-standard ports |

---

## Testing Policy Enforcement

```bash
# Apply violation manifests — they run because mutation enforces secure defaults
kubectl apply -f tests/violations/

# To test Deny behavior directly, disable mutation first:
kubectl delete mutatingpolicy add-default-securitycontext
kubectl apply -f tests/violations/root-container.yaml
# → Error: pods is forbidden — runAsNonRoot must be true

# Restore mutation:
kubectl apply -f kyverno/policies/mutate/add-default-securitycontext.yaml
```

---

## Design Decisions

| ADR | Decision |
|-----|----------|
| [ADR-001](docs/adr/ADR-001-cluster-setup.md) | Kind + Calico; kubeadm hardening tradeoffs |
| [ADR-002](docs/adr/ADR-002-falco-wsl2-kyverno-exclusion.md) | Falco WSL2 limitations + Kyverno namespace exclusion strategy |

---

## Author

**Ghofrane Haddedi** — DevSecOps Engineer
[GitHub](https://github.com/ghofranehad) · CKA · CKS
