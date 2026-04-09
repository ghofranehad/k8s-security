# ADR-005: Why Sealed Secrets over External Secrets Operator

## Context

Kubernetes Secrets are base64-encoded, not encrypted. Storing them in Git
exposes credentials. A solution is needed to manage secrets safely in a
GitOps workflow without requiring external secret stores.

## Options Considered

### External Secrets Operator (ESO)
- Syncs secrets from external stores: AWS Secrets Manager, HashiCorp Vault, Azure Key Vault, GCP Secret Manager
- Secrets never stored in Git — fetched at runtime from the external store
- Requires an external secret store to be running and accessible
- Additional infrastructure dependency — the store must be highly available
- Ideal for production environments where a secret store already exists

### HashiCorp Vault
- Full secrets lifecycle management: dynamic secrets, rotation, PKI, encryption as a service
- Complex to operate — requires its own HA cluster, storage backend, unseal process
- Gold standard for enterprise secrets management
- Overkill for a lab environment without an existing Vault cluster

### Sealed Secrets (Bitnami Labs)
- Encrypts secrets with the cluster's public key — SealedSecret CRD stored safely in Git
- No external dependency — the controller and key live inside the cluster
- Fully GitOps-compatible — sealed secrets are committed alongside manifests
- Simple mental model: encrypt with kubeseal CLI, commit, controller decrypts
- CNCF landscape project

## Decision

**Sealed Secrets** for this project. **Vault** is the production recommendation.

## Reasons

1. **GitOps compatibility** — Sealed Secrets are designed for GitOps. The encrypted secret lives in Git next to the workload that uses it. No external dependency to manage.

2. **No infrastructure overhead** — ESO and Vault require an external secret store. In a local Kind cluster, spinning up a Vault HA cluster for lab purposes defeats the purpose.

3. **Simplicity** — The mental model is clear: `kubeseal` encrypts → Git stores → controller decrypts. Any engineer can understand and operate it in minutes.

4. **Portfolio clarity** — Demonstrating Sealed Secrets shows understanding of the GitOps secrets problem. It is a deliberate, justified choice — not a limitation.

## Trade-offs

| Concern | Impact |
|---------|--------|
| No secret rotation | Acceptable for static lab secrets |
| Key loss = all secrets lost | Mitigated by key backup procedure (documented) |
| No dynamic secrets | Out of scope for this project |
| ESO is preferred in cloud environments | Acknowledged — see production note below |

## Production Note

In a production environment with an existing cloud provider or Vault cluster,
External Secrets Operator is preferred:
- Secrets are never stored anywhere — fetched live from the store
- Automatic rotation is supported
- Audit trail in the secret store
- No key management burden

Sealed Secrets is the right choice for GitOps-first lab environments.
ESO/Vault is the right choice for production.

## Status

Decided — Sealed Secrets v0.36.1 deployed via ArgoCD.
