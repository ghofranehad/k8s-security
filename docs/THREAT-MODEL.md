# Threat Model

Documents the threat actors, attack scenarios, and mitigations implemented in this project.
Methodology: STRIDE + MITRE ATT&CK for Containers.

---

## Threat Actors

| Actor | Goal | Capability |
|-------|------|-----------|
| External attacker | Compromise workloads, steal data | Exploits public-facing apps, supply chain |
| Malicious insider | Abuse cluster access, exfiltrate secrets | kubectl access, Git access |
| Compromised CI/CD | Deploy backdoored images | Push access to registry |
| Compromised container | Escape to host, lateral movement | Code execution inside a pod |

---

## Attack Scenarios

### Scenario 1 — Vulnerable Image Deployed

**Threat:** A developer pushes a Deployment using an image with a critical CVE.

**Attack path:**
```
Developer pushes code → CI builds image with vulnerable dependency
                     → Image pushed to registry
                     → Deployment applied to cluster
                     → Attacker exploits CVE inside running container
```

**Mitigations:**
| Layer | Control |
|-------|---------|
| CI/CD (Trivy) | Blocks images with HIGH/CRITICAL CVEs before merge |
| Supply Chain (Cosign) | Verifies image was built by trusted pipeline |
| Admission (Kyverno) | Denies pods without resource limits — limits blast radius |

---

### Scenario 2 — Supply Chain Attack

**Threat:** Attacker compromises Docker Hub and replaces a legitimate image with a backdoored version.

**Attack path:**
```
Attacker gains registry write access
→ Replaces falcosecurity/falco:latest with malicious image
→ Cluster pulls tampered image on next pod restart
→ Attacker has code execution inside Falco pod (security tool compromised)
```

**Mitigations:**
| Layer | Control |
|-------|---------|
| Supply Chain (Cosign) | Signature verification fails — tampered image has no valid signature → pod denied |
| CI/CD (Trivy) | SBOM generation tracks exact package versions — drift is visible |

---

### Scenario 3 — Privileged Container Escape

**Threat:** Attacker deploys a privileged container to escape to the host node.

**Attack path:**
```
Attacker with kubectl access creates pod with privileged: true
→ Pod admitted → mounts host filesystem
→ Attacker writes cron job to host → persistence
→ Full node compromise
```

**Mitigations:**
| Layer | Control |
|-------|---------|
| Admission (Kyverno) | block-privileged-containers denies privileged: true |
| PSS restricted | Blocks hostPath, hostPID, hostNetwork at namespace level |
| GitOps (ArgoCD) | Unauthorized kubectl apply tracked — all changes must come from Git |

---

### Scenario 4 — Credential Theft from Running Container

**Threat:** Attacker with code execution inside a container reads secrets from the filesystem.

**Attack path:**
```
Attacker exploits RCE in app container
→ Reads /etc/shadow, /root/.ssh/id_rsa
→ Uses credentials to pivot to other systems
```

**Mitigations:**
| Layer | Control |
|-------|---------|
| Runtime (Falco) | Sensitive File Read rule fires on /etc/shadow, /etc/passwd, SSH keys |
| PSS (readOnlyRootFilesystem) | Container filesystem is read-only — limits write-based persistence |
| Secrets (Sealed Secrets) | No plaintext credentials in Git or environment variables |
| Admission (Kyverno) | runAsNonRoot: true — attacker runs as UID 1000, not root |

---

### Scenario 5 — Lateral Movement Between Namespaces

**Threat:** Attacker with code execution in one pod attempts to reach pods in other namespaces.

**Attack path:**
```
Attacker compromises pod in namespace A
→ Scans internal network for other services
→ Connects to database pod in namespace B
→ Exfiltrates data
```

**Mitigations:**
| Layer | Control |
|-------|---------|
| Network (default-deny) | All ingress/egress blocked by default — attacker pod cannot send traffic |
| Network (NetworkPolicy) | Only explicitly allowed traffic passes — no cross-namespace by default |
| Runtime (Falco) | Unexpected Outbound Connection fires on non-standard port egress |

---

### Scenario 6 — Malicious Insider Deploys Backdoor

**Threat:** Developer with Git access pushes a malicious workload to the cluster.

**Attack path:**
```
Insider pushes malicious Deployment to Git
→ ArgoCD syncs → deploys backdoored pod
→ Pod phones home to attacker C2 server
```

**Mitigations:**
| Layer | Control |
|-------|---------|
| GitOps (ArgoCD) | AppProject sourceRepos restricts deployments to trusted repo — PR review is the gate |
| Supply Chain (Cosign) | Images must be signed by trusted pipeline — locally built images fail verification |
| Network (default-deny) | Egress to C2 server blocked — only DNS, 80, 443 allowed |
| Runtime (Falco) | Unexpected Outbound Connection fires on C2 callback port |

---

## Residual Risks

| Risk | Reason not fully mitigated |
|------|---------------------------|
| Zero-day in admitted image | CVE scanning only covers known vulnerabilities |
| Compromised Kyverno | Security tooling itself is a high-value target |
| Falco evasion | Sophisticated attackers may use techniques that avoid syscall detection |
| Key compromise (Sealed Secrets) | If controller private key is leaked, all secrets are exposed |
| ArgoCD compromise | Full cluster access — protect with MFA and audit logs |

Residual risks are documented, not ignored. Defense-in-depth reduces the probability that any single failure leads to full compromise.
