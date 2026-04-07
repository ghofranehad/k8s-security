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

## Problem 4: Falco fails on control-plane and worker2 — inotify instance limit exhausted

### Symptom
```
Error: could not initialize inotify handler
```
Only the worker node pod runs successfully. Control-plane and worker2 pods crash.

### Root Cause
WSL2 defaults to `fs.inotify.max_user_instances=128`. The control-plane node
consumes most of those instances with its system components (etcd, kube-apiserver,
kube-scheduler, kube-controller-manager). Worker2 also hits the limit under load.
Falco cannot obtain an inotify instance and crashes.

### Fix
Run on the WSL2 host before creating Kind clusters:

```bash
echo "fs.inotify.max_user_instances=1024" | sudo tee -a /etc/sysctl.conf
echo "fs.inotify.max_user_watches=1048576" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

**This is a documented Kind prerequisite for Linux/WSL2.** Apply it once on the
host — it persists across reboots via `/etc/sysctl.conf`.

---

## Problem 5: container.name and k8s.pod.name always null

### Symptom
```json
{"rule": "Sensitive File Read in Container", "output_fields": {"container.name": null, "k8s.pod.name": null, "proc.cmdline": "cat /etc/passwd"}}
```
Falco detects the syscall correctly but cannot enrich alerts with container or pod identity.

### Root Cause
Falco enriches events by querying the container runtime via the CRI socket
(`/host/run/containerd/containerd.sock`). In Kind/WSL2, the CRI metadata
query does not return container information reliably — the socket is accessible
but the enrichment pipeline does not associate the syscall's cgroup with a
container entry in time.

### Impact
- Rules fire correctly (syscall detection works)
- Cannot filter alerts by pod/container name
- False positives from host processes (`cron`, `runc`) cannot be suppressed by container name

### Production note
In a real cluster (bare metal or cloud), the CRI socket is stable and enrichment
works. Alerts include full `k8s.pod.name`, `container.name`, `k8s.ns.name` fields,
enabling precise per-workload alerting and SIEM correlation.

---

## Decision

Apply `excludeResourceRules` + `matchConditions` pattern to all Kyverno policies.
Disable `falcoctl` entirely on WSL2/Kind environments.
Raise inotify limits on WSL2 host before cluster creation.
Accept container metadata null as a WSL2/Kind lab limitation — syscall detection works.

## Status

Resolved — Falco running on all 3 Kind nodes, rules firing correctly.
Container metadata enrichment not available in WSL2/Kind (documented limitation).
