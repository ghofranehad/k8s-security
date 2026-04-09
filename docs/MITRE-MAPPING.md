# MITRE ATT&CK Mapping

Maps each security layer in this project to the MITRE ATT&CK techniques it detects or mitigates.
Framework: [MITRE ATT&CK for Containers](https://attack.mitre.org/matrices/enterprise/containers/)

---

## Coverage Overview

| Tactic | Techniques Covered |
|--------|-------------------|
| Initial Access | T1190, T1195 |
| Execution | T1059, T1610 |
| Persistence | T1525, T1611 |
| Privilege Escalation | T1611, T1068 |
| Defense Evasion | T1578, T1036 |
| Credential Access | T1552, T1528 |
| Discovery | T1613, T1082 |
| Lateral Movement | T1210, T1021 |
| Exfiltration | T1041, T1048 |

---

## Layer-by-Layer Mapping

### Layer 1 — CI/CD (Trivy + Cosign)

| Technique | ID | Control |
|-----------|-----|---------|
| Supply Chain Compromise | T1195.002 | Cosign verifies image signature before cluster admission |
| Exploit Public-Facing Application | T1190 | Trivy blocks images with HIGH/CRITICAL CVEs in CI |
| Deploy Container | T1610 | Misconfigured manifests blocked by Trivy config scan |

---

### Layer 2 — GitOps (ArgoCD)

| Technique | ID | Control |
|-----------|-----|---------|
| Implant Internal Image | T1525 | AppProject sourceRepos restricts deployments to trusted repo only |
| Modify Cloud Compute Infrastructure | T1578 | All cluster changes must pass through Git — no direct kubectl apply |
| Valid Accounts | T1078 | ArgoCD RBAC prevents unauthorized application deployment |

---

### Layer 3 — Admission Control (Kyverno)

| Technique | ID | Control |
|-----------|-----|---------|
| Container Administration Command | T1609 | block-privileged-containers denies privileged pods |
| Escape to Host | T1611 | Capabilities drop ALL, no hostPID/hostNetwork via PSS |
| Deploy Container | T1610 | Unsigned or non-compliant workloads denied at admission |
| Masquerading | T1036 | require-resource-limits prevents resource exhaustion via unlimted containers |

---

### Layer 4 — Pod Security Standards

| Technique | ID | Control |
|-----------|-----|---------|
| Escape to Host | T1611 | Privileged containers, hostPath, hostPID blocked by restricted PSS |
| Privilege Escalation | T1068 | allowPrivilegeEscalation: false enforced cluster-wide |
| Container Administration Command | T1609 | runAsNonRoot: true reduces blast radius of container compromise |

---

### Layer 5 — Network Segmentation (Calico + NetworkPolicy)

| Technique | ID | Control |
|-----------|-----|---------|
| Lateral Movement — Remote Services | T1021 | Default deny-all blocks pod-to-pod traffic unless explicitly allowed |
| Lateral Movement — Internal Spearphishing | T1210 | Namespace isolation prevents cross-namespace communication |
| Exfiltration Over C2 Channel | T1041 | Egress restricted to DNS only for app pods |
| Data Exfiltration | T1048 | Unexpected outbound ports blocked by default-deny egress |

---

### Layer 6 — Runtime Detection (Falco)

| Technique | ID | Falco Rule |
|-----------|-----|------------|
| Unix Shell | T1059.004 | Shell Spawned in Container — detects sh/bash/zsh in running containers |
| Credential Dumping — /etc/passwd | T1003 | Sensitive File Read — detects reads of /etc/shadow, /etc/passwd |
| Unsecured Credentials in Files | T1552.001 | Sensitive File Read — detects reads of /root/.ssh/id_rsa |
| Exfiltration Over Alternative Protocol | T1048 | Unexpected Outbound Connection — detects non-standard port egress |
| Discovery — Container and Resource Discovery | T1613 | Shell Spawned in Container — shell is the first step of discovery |

---

### Layer 7 — Secrets Management (Sealed Secrets)

| Technique | ID | Control |
|-----------|-----|---------|
| Unsecured Credentials in Files | T1552.001 | Secrets encrypted in Git — no plaintext credentials committed |
| Credentials from Password Stores | T1555 | Secrets decrypted only in-cluster by controller — never exposed in Git |
| Cloud Instance Metadata API | T1552.005 | No credentials stored in environment variables or config files in Git |

---

## Attack Chain Example — Container Compromise

```
Attacker exploits vulnerability in app container
            │
            ▼
T1059 — Attempts to spawn shell
    → Falco: Shell Spawned in Container (WARNING)
            │
            ▼
T1552 — Reads /etc/shadow for credentials
    → Falco: Sensitive File Read (ERROR)
            │
            ▼
T1210 — Attempts lateral movement to other pods
    → NetworkPolicy: default-deny blocks connection
            │
            ▼
T1048 — Attempts data exfiltration on port 4444
    → NetworkPolicy: egress blocked (not port 80/443/53)
    → Falco: Unexpected Outbound Connection (WARNING)
            │
            ▼
T1611 — Attempts container escape via privileged syscall
    → PSS restricted: allowPrivilegeEscalation: false
    → Kyverno: capabilities.drop ALL injected
```

Every step in the chain is covered by at least one layer.
