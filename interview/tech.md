## 2) Tech Round — K8s / RHEL / Azure Use Cases & What to Say

### A) Kubernetes (K8s / AKS / on-prem)

**1) “Design a CI/CD pipeline to deploy microservices to Kubernetes.”**
**Answer outline**

* Build → unit tests → container build (Docker/Podman) → push to registry
* Security gate (SCA / image scan / Veracode where applicable)
* Deploy via Helm or Kustomize; env differences via values/overlays
* Progressive delivery: rolling / canary / blue-green
* Post-deploy smoke test + health gates (readiness)
* Versioning + rollback strategy (no `latest`, chart versioning)

**2) “Helm vs Kustomize—when do you use each?”**

* Helm: strong packaging + ecosystem + versioned charts
* Kustomize: clean overlays for multi-env patching
* Decide by team standards, audit needs, and existing assets

**3) “Pod CrashLoopBackOff—how do you troubleshoot?”**

* `kubectl describe pod` (events)
* `kubectl logs` + `--previous`
* Resources (OOMKilled), env vars/Secrets/ConfigMaps
* Probe config (readiness/liveness), dependency failures, network policies

**4) “Readiness vs Liveness vs Startup probes?”**

* readiness: traffic gating
* liveness: restart decision
* startup: protects slow-start apps from being killed early

**5) “How do you handle scaling?”**

* right-size requests/limits first
* HPA based on CPU/memory or custom metrics
* coordinate with PDB and cluster autoscaling (if enabled)

---

### B) RHEL / Linux Administration

**1) “Service won’t start—what do you check?”**

* `systemctl status`
* `journalctl -u <svc>`
* permissions, env, ports, dependencies
* SELinux/audit logs if access is denied

**2) “How do you patch/upgrade safely?”**

* staged rollout, maintenance windows, rollback plan
* verification checklist and health checks
* document changes + coordinate stakeholders

---

### C) Azure Networking / Application Gateway

**1) “Explain VNet/Subnet/NSG/UDR.”**

* VNet boundary, subnets segmentation
* NSG for L3/L4 rules
* UDR for forced routing (firewalls/NVA), traffic control

**2) “App Gateway health probes failing—what could cause it?”**

* wrong probe path/host header
* TLS/cert mismatch
* timeouts or backend response codes
* WAF rules blocking
* misconfigured HTTP settings (port/protocol)

**3) “How do you secure CI/CD credentials?”**

* prefer OIDC federated auth to avoid long-lived secrets
* Key Vault for secrets
* least privilege RBAC per environment
* rotation and audit

---

### D) Veracode CLI in pipelines

**“How do you integrate Veracode scans and decide fail conditions?”**

* run in CI after build, before deploy
* publish reports to dev teams
* fail on critical/high or “new high severity introduced”
* baseline legacy issues but enforce “no regression”
