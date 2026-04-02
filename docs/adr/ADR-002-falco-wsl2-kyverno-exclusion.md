# ADR-002: Falco on WSL2 + Kyverno Policy Exclusion Strategy

## Context

Deploying Falco (Layer 5 — Runtime Detection) on a Kind cluster running inside WSL2
exposed two independent problems: Kyverno policies interfering with Falco pod startup,
and a WSL2 kernel limitation in Falco's rule-update mechanism.

---

## Problem 1: Kyverno MutatingPolicy blocking Falco pods

### Symptom
```
container has runAsNonRoot and image will run as root
back-off restarting failed container=falcoctl-artifact-follow
```

### Root Cause
The `add-default-securitycontext` MutatingPolicy injects `runAsNonRoot: true`
into every pod across all namespaces. Falco requires root to access kernel-level
eBPF syscall data. The mutation overrides Falco's own security context, causing
the pod to crash on startup.

### Fix
Exclude infrastructure namespaces from all Kyverno policies (both mutate and validate)
using `matchConditions` with an explicit namespace list.

---

## Problem 2: Kyverno namespace exclusion — three failed approaches

Excluding infra namespaces from Kyverno policies proved non-trivial due to how
Kyverno pre-validates workload resources (DaemonSets, Deployments) against pod policies.

### Approach 1: `matchConditions` with `object.metadata.namespace` — FAILED

```yaml
matchConditions:
  - name: exclude-infra-namespaces
    expression: >-
      !(object.metadata.namespace in ['falco', ...])
```

**Error:**
```
daemonsets.apps "falco" is forbidden: expression
'!((object.kind == 'DaemonSet') || ...) ||
(!(object.spec.template.metadata.namespace in [...]))'
resulted in error: no such key: namespace
```

**Why it fails:** Kyverno pre-validates DaemonSets/Deployments by checking their
pod template against pod policies. Internally, Kyverno rewrites
`object.metadata.namespace` to `object.spec.template.metadata.namespace` for
workload resources. But pod templates never have `namespace` set in their metadata
(pods inherit the namespace from the parent workload). The generated CEL crashes.

### Approach 2: `namespaceSelector` in `matchConstraints` — FAILED

```yaml
matchConstraints:
  namespaceSelector:
    matchExpressions:
      - key: kubernetes.io/metadata.name
        operator: NotIn
        values: [falco, ...]
```

**Error:**
```
PolicyViolation policy block-root-containers/evaluation error:
failed to load context: no such key: namespace
```

**Why it fails:** `namespaceSelector` requires Kyverno to resolve the namespace
of the resource being evaluated. During workload pre-validation, Kyverno evaluates
against the pod template — which has no namespace. The namespace context lookup
fails before any CEL expression runs.

### Approach 3: `excludeResourceRules` + `matchConditions` — WORKS

```yaml
matchConstraints:
  resourceRules:
    - apiGroups: ['']
      apiVersions: [v1]
      operations: [CREATE, UPDATE]
      resources: [pods]
  excludeResourceRules:
    - apiGroups: ['apps']
      apiVersions: ['*']
      operations: ['*']
      resources: ['daemonsets', 'deployments', 'replicasets', 'statefulsets']
    - apiGroups: ['batch']
      apiVersions: ['*']
      operations: ['*']
      resources: ['jobs', 'cronjobs']
matchConditions:
  - name: exclude-infra-namespaces
    expression: >-
      !(object.metadata.namespace in ['falco', ...])
```

**Why it works:**
- `excludeResourceRules` prevents Kyverno from ever running its workload
  pre-validation logic for DaemonSets, Deployments, etc.
- `matchConditions` now only runs against actual Pod objects, which always have
  `metadata.namespace` set — safe to access without optional chaining.
- Pods in infra namespaces are excluded by the CEL expression.
- Pods in app namespaces are validated/mutated as intended.

**Trade-off:** Workload resources (DaemonSets, Deployments) are not pre-validated
against pod policies. Violations are only caught when the pod is actually created,
not at the Deployment level. Acceptable for this setup.

---

## Problem 3: Falco `falcoctl artifact follow` fails on WSL2

### Symptom
```
Hostname value has been overridden via environment variable to: k8s-security-hardening-worker
Error: could not initialize inotify handler
```

### Root Cause
`falcoctl artifact follow` is a sidecar that watches for Falco rule updates using
Linux `inotify`. WSL2 uses a custom Microsoft kernel that does not support inotify
for the filesystem paths Falco monitors inside Kind nodes.

### Fix
Disable `falcoctl artifact follow` in Helm values. The one-time `artifact install`
init container still runs on pod startup, loading the default Falco ruleset.
Rules are not auto-updated, but this is acceptable in a static lab environment.

```yaml
falcoctl:
  artifact:
    follow:
      enabled: false
```

**Production note:** In a real cluster (non-WSL2), re-enable `follow: true` to
receive automatic rule updates from the Falco rules repository.

---

## Decision

Apply `excludeResourceRules` + `matchConditions` pattern to all Kyverno policies.
Disable `falcoctl artifact follow` on WSL2/Kind environments.

## Status

Resolved — Falco running on Kind/WSL2 with modern_ebpf driver and custom rules loaded.
