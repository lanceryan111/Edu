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


根据你发的 **DevOps Engineer JD**，我把核心技术点拆出来，然后针对 **面试最可能问的技术问题**整理了一组 **15个问题**。
重点我会 **多放在 Ansible / AKS / Azure DevOps / CI-CD / Kubernetes / Security automation** 上，因为这些在描述里出现最多，也是银行 DevOps 面试最容易深入的地方。

我同时按照 **真实面试风格**写问题（不是理论考试，而是场景题）。

---

# 可能会问的 DevOps 技术问题（15个）

## 1 Kubernetes / AKS

### 1️⃣ AKS Architecture

**Question**

> Can you explain how Azure Kubernetes Service (AKS) architecture works and what components Azure manages vs what customers manage?

**面试官想听**

* Control plane **Azure managed**
* Worker nodes in **node pools**
* Managed etcd
* integration with:

  * Azure AD
  * Azure CNI
  * Azure Load Balancer
  * Azure Monitor

**可能追问**

* node pool scaling
* system vs user node pools

---

### 2️⃣ AKS Deployment Strategy

> How would you design a CI/CD pipeline to deploy a microservice to AKS?

关键点

* build container image
* push to **ACR**
* Helm/Kustomize deployment
* pipeline stages

典型回答结构：

```
CI
   build image
   security scan
   push ACR

CD
   deploy via Helm
   rollout strategy
   health check
```

---

### 3️⃣ Kubernetes Rolling Deployment

> How does Kubernetes rolling update work and how do you avoid downtime?

关键点

* deployment strategy
* maxUnavailable
* maxSurge
* readinessProbe

---

### 4️⃣ Kubernetes Troubleshooting

> A pod keeps restarting in AKS. How would you troubleshoot it?

Expected flow

```
kubectl get pods
kubectl describe pod
kubectl logs
kubectl events
resource limits
probe failures
image pull errors
```

---

# 2 Ansible (高概率问题)

### 5️⃣ Ansible vs Terraform

> When would you use Ansible instead of Terraform?

Expected answer

Terraform

* infrastructure provisioning

Ansible

* configuration management
* application deployment

Example

```
Terraform → create VM
Ansible → configure software
```

---

### 6️⃣ Ansible Playbook Structure

> Can you explain the structure of an Ansible playbook?

Key parts

```
hosts
tasks
roles
vars
handlers
```

Example

```
- hosts: web
  tasks:
    - name: install nginx
      apt:
```

---

### 7️⃣ Ansible Idempotency

> What does idempotency mean in Ansible and why is it important?

Answer

* same playbook run multiple times
* same state
* avoids configuration drift

Example

```
state: present
```

---

### 8️⃣ Ansible Role Design

> How would you design reusable Ansible roles for multiple environments?

Expected answer

```
roles/
  nginx/
     tasks
     templates
     vars
     defaults
```

Environment variables

```
group_vars
host_vars
```

---

# 3 Azure DevOps / CI-CD

（这个很可能深挖）

---

### 9️⃣ Azure DevOps Pipeline Design

> How would you design a multi-stage pipeline in Azure DevOps?

Expected answer

```
Build stage
Test stage
Security scan
Deploy stage
Approval gates
```

Example

```
dev → test → prod
```

---

### 🔟 Secure Pipeline

> How do you manage secrets in Azure DevOps pipelines?

Expected

* Azure Key Vault
* pipeline secrets
* managed identity
* variable groups

---

### 1️⃣1️⃣ Pipeline Optimization

> Our pipeline takes 25 minutes to build. How would you improve it?

Possible answers

* parallel jobs
* caching dependencies
* incremental build
* container build optimization

---

# 4 Security Automation

JD提到 **Veracode scanning**

---

### 1️⃣2️⃣ Security Scanning in CI/CD

> How would you integrate security scanning into a CI/CD pipeline?

Expected

```
SAST
dependency scanning
container scanning
```

Example tools

```
Veracode
Trivy
Snyk
```

---

### 1️⃣3️⃣ Handling Vulnerabilities

> What would you do if a security scan fails during pipeline?

Expected

* severity threshold
* block release
* create ticket
* notify team

---

# 5 GitHub Actions / Automation

---

### 1️⃣4️⃣ GitHub Actions vs Azure DevOps

> What are the differences between GitHub Actions and Azure DevOps pipelines?

Expected

GitHub Actions

* tight GitHub integration

Azure DevOps

* enterprise pipeline control
* approvals
* artifacts

---

### 1️⃣5️⃣ Release Management

> How do you manage releases for multiple microservices?

Expected answer

```
versioning
tagging
artifact repo
deployment orchestration
```

Example

```
service-based deployment
```

---

# 银行 DevOps 面试特别喜欢问的 3 个问题

我强烈建议你准备这三个：

### ⭐1

> How do you design a reliable CI/CD system for production?

---

### ⭐2

> How do you handle failed deployments in production?

---

### ⭐3

> How do you ensure infrastructure changes are auditable and secure?

---

# 如果是 **Lead DevOps role**

他们还可能问：

> How would you improve DevOps maturity in a development team?

答案要提

* CI/CD standardization
* infrastructure as code
* monitoring
* deployment safety

---

# 如果你愿意

我可以 **再帮你做一版更狠的银行面试版本：**

**20个 Azure + AKS + DevOps 深度技术问题**

比如：

* AKS networking architecture
* Azure Application Gateway ingress controller
* GitOps with ArgoCD
* AKS scaling strategies
* Ansible dynamic inventory
* Azure DevOps deployment gates

这些是 **Lead DevOps 面试最容易问的**。

我根据你这个 **TD DevOps Engineer JD**（里面有 **AKS / Azure Networking / Ansible / GitHub Actions / Security scanning / automation**），整理了一套 **更接近银行 DevOps Lead 面试难度的 20 个深度技术问题**。
这些问题基本覆盖 **Azure + AKS + DevOps + Reliability + Security + Automation**。

我按 **模块分类**，这样你复习也比较清晰。

---

# 20 个 Azure + AKS + DevOps 深度技术面试问题

---

# 1 AKS Architecture (4题)

### 1️⃣ Explain the architecture of AKS. What components are managed by Azure and what are customer responsibilities?

重点：

Azure managed

* control plane
* etcd
* API server

Customer responsibility

* node pools
* networking
* RBAC
* deployments

---

### 2️⃣ What are the differences between Azure CNI and Kubenet networking in AKS?

面试官非常爱问。

Azure CNI

* pods get VNET IP
* better network integration
* consumes more IPs

Kubenet

* NAT based
* smaller IP consumption
* limited network features

---

### 3️⃣ How would you scale workloads in AKS?

Expected points

* HPA (Horizontal Pod Autoscaler)
* Cluster Autoscaler
* KEDA

Example

```text
pod scaling → HPA
node scaling → cluster autoscaler
event scaling → KEDA
```

---

### 4️⃣ How do you design high availability for AKS workloads?

Expected

* multi-zone node pools
* readiness / liveness probes
* multiple replicas
* pod disruption budgets

---

# 2 AKS Operations / Troubleshooting (3题)

### 5️⃣ A pod in AKS keeps restarting. How would you troubleshoot it?

Typical process

```text
kubectl get pods
kubectl describe pod
kubectl logs
kubectl events
check probes
check resource limits
```

---

### 6️⃣ How would you debug networking issues between pods in AKS?

Expected

* kubectl exec
* DNS check
* service discovery
* network policy

Tools

```text
nslookup
curl
netstat
```

---

### 7️⃣ How do you upgrade an AKS cluster safely?

Expected

* upgrade node pool
* rolling upgrade
* cordon + drain nodes
* blue/green strategy

---

# 3 Azure DevOps Pipelines (4题)

### 8️⃣ How would you design a CI/CD pipeline to deploy containerized apps to AKS?

Typical pipeline

```
Build
Test
Security scan
Push to ACR
Deploy via Helm
Smoke test
```

---

### 9️⃣ How do you secure secrets in Azure DevOps pipelines?

Expected

* Azure Key Vault
* variable groups
* managed identities
* service connections

---

### 🔟 How do you implement approval gates before production deployment?

Expected

Azure DevOps features

* environment approvals
* manual validation
* release gates

Example

```
dev → test → approval → prod
```

---

### 1️⃣1️⃣ Pipeline performance optimization

> A pipeline takes 30 minutes. How would you optimize it?

Expected

* parallel jobs
* caching dependencies
* build layers
* container caching

---

# 4 Infrastructure as Code / Ansible (3题)

### 1️⃣2️⃣ How would you use Ansible to configure infrastructure after provisioning?

Typical flow

```
Terraform → provision VM
Ansible → configure services
```

Example

```
install packages
configure nginx
deploy application
```

---

### 1️⃣3️⃣ What is idempotency in Ansible?

Expected

* repeated runs produce same result
* avoids configuration drift

Example

```
state: present
```

---

### 1️⃣4️⃣ How do you structure Ansible roles for reuse?

Expected

```
roles/
  webserver/
     tasks
     templates
     defaults
     handlers
```

---

# 5 Azure Networking (3题)

### 1️⃣5️⃣ How does Azure Application Gateway integrate with AKS?

Expected

* Application Gateway Ingress Controller (AGIC)

Flow

```
Internet
↓
Application Gateway
↓
AKS ingress
↓
services
```

Benefits

* WAF
* L7 routing

---

### 1️⃣6️⃣ How do you expose AKS services externally?

Options

```
LoadBalancer
NodePort
Ingress Controller
Application Gateway
```

Most production

```
Ingress + Application Gateway
```

---

### 1️⃣7️⃣ How would you design secure network access to AKS?

Expected

* private AKS cluster
* NSG
* network policies
* Azure Firewall

---

# 6 Security / DevSecOps (2题)

### 1️⃣8️⃣ How would you integrate security scanning into CI/CD?

Typical security pipeline

```
SAST
Dependency scan
Container scan
Secrets scan
```

Tools

```
Veracode
Snyk
Trivy
```

---

### 1️⃣9️⃣ What would you do if a vulnerability is found in production images?

Expected

```
patch dependency
rebuild image
redeploy
```

Also

* CVE tracking
* vulnerability management

---

# 7 DevOps Reliability (1题)

### 2️⃣0️⃣ How do you design a reliable deployment system for microservices?

Expected answer

```
rolling deployment
blue/green
canary release
automated rollback
health checks
```

Example

```
Helm deployment
readiness probes
monitoring alerts
```

---

# 银行 DevOps 面试最喜欢问的 5 个问题（你一定要准备）

我根据你 **TD银行环境**帮你筛选一下：

### ⭐最可能

1️⃣ AKS architecture
2️⃣ CI/CD pipeline design
3️⃣ Ansible vs Terraform
4️⃣ Secrets management in pipelines
5️⃣ Troubleshooting Kubernetes pods

---

# 我给你一个额外建议（很关键）

因为你是 **internal candidate + DevOps background**，他们其实最想看的是：

> **你有没有 platform / architecture thinking**

所以回答尽量包含：

* scalability
* reliability
* security
* automation

比如：

不要只说

> deploy to AKS

要说

> automated CI/CD pipeline with security scanning, Helm deployment, and health checks to ensure reliability.

---

如果你愿意，我可以再给你做一个 **银行 DevOps 面试 killer list**：

**15 个 AKS / Azure / DevOps “高阶追问问题”**

比如：

* AKS networking deep dive
* Azure identity integration
* GitOps vs CI/CD
* AKS cost optimization
* AKS disaster recovery

这些是 **Lead DevOps / Senior DevOps 面试经常用来刷人的问题**。

下面是 **银行 / 大型企业 DevOps 面试里经常用来“刷人”的 15 个 Killer Questions**。
这些问题通常不是考基础，而是看你 **是否有 architecture / production / reliability 经验**。
尤其像你这种 **internal move + DevOps background + Lead level**，面试官很容易问这种。

我按 **AKS / Azure / DevOps architecture / Security / Reliability** 分组。

---

# 🔥 Azure + AKS + DevOps Killer Question List (15)

---

# 1 AKS Networking Deep Dive

### 1️⃣ How does networking work in AKS and how would you design it for a large enterprise environment?

面试官想听：

* Azure CNI
* VNET integration
* private cluster
* subnet planning
* ingress architecture

高分答案会提：

```
AKS
│
VNET
│
subnet separation
│
Application Gateway / Ingress
│
private endpoint
```

---

### 2️⃣ What problems might you encounter with Azure CNI in large AKS clusters?

重点

* IP exhaustion
* subnet sizing
* scaling limits

好的回答：

> Plan subnet CIDR carefully to support pod scaling.

---

# 2 AKS Production Operations

### 3️⃣ How would you design disaster recovery for an AKS cluster?

Expected：

* multi-region deployment
* container registry replication
* backup etcd state
* infrastructure as code

Example

```
region A
AKS cluster

region B
standby cluster
```

---

### 4️⃣ What monitoring strategy would you implement for AKS?

Expected tools

* Azure Monitor
* Prometheus
* Grafana
* Log Analytics

关键指标：

```
CPU
memory
pod restart
node pressure
```

---

### 5️⃣ How do you handle configuration management in Kubernetes?

Expected

* ConfigMap
* Secret
* environment variables
* Helm values

---

# 3 DevOps Architecture

### 6️⃣ How would you design a CI/CD pipeline for hundreds of microservices?

这是 **大厂最爱的问题之一**

Expected answer

```
shared pipeline templates
artifact repository
versioning
environment promotion
```

Example

```
build once
deploy many
```

---

### 7️⃣ How do you prevent pipeline failures from blocking multiple teams?

Expected

* pipeline isolation
* retry strategy
* artifact caching
* dependency control

---

### 8️⃣ GitOps vs CI/CD — when would you use GitOps?

Expected

GitOps tools

```
ArgoCD
Flux
```

优势

* declarative
* Git as source of truth
* auditability

---

# 4 Security / DevSecOps

### 9️⃣ How do you secure container images in production?

Expected

```
image scanning
minimal base image
signed images
```

Tools

```
Trivy
Aqua
Defender for Cloud
```

---

### 🔟 How would you secure access to AKS?

Expected

```
Azure AD integration
RBAC
Managed Identity
private cluster
```

---

### 1️⃣1️⃣ How do you prevent secrets from leaking in CI/CD pipelines?

Expected

```
Key Vault
secret rotation
pipeline masking
managed identity
```

---

# 5 Reliability / Production Engineering

### 1️⃣2️⃣ How do you design safe deployments for production?

Expected

deployment strategies

```
rolling deployment
blue green
canary
```

Health checks

```
readiness probe
liveness probe
```

---

### 1️⃣3️⃣ What happens if a deployment fails in production?

Expected

```
automatic rollback
health check failure
alerting
```

---

### 1️⃣4️⃣ How do you reduce deployment risk in a distributed system?

Expected

```
feature flags
canary release
traffic splitting
```

---

### 1️⃣5️⃣ How would you design DevOps processes for a team with low DevOps maturity?

这是 **Lead role 高频题**

Expected

```
CI/CD standardization
Infrastructure as Code
monitoring
security automation
```

---

# ⭐ 银行 DevOps 面试最可能问的 5 个 Killer Questions

我根据 **TD这种银行环境**帮你筛选一下：

最可能出现：

1️⃣ **Design CI/CD for microservices**
2️⃣ **AKS networking architecture**
3️⃣ **Secrets management strategy**
4️⃣ **Safe deployment strategy**
5️⃣ **Monitoring and reliability in AKS**

---

# ⭐ 我再给你一个非常重要的技巧

银行 DevOps 面试回答最好用这个结构：

```
1 problem
2 design
3 tools
4 reliability
```

比如

**How would you deploy to AKS**

回答结构：

```
CI pipeline
build container
security scan
push to ACR

CD pipeline
Helm deployment
rolling update
health check

monitoring
rollback strategy
```

面试官听到这种结构 **基本会认为你是 senior / lead level**。

---

如果你愿意，我可以再帮你做一个 **终极版本**（这个对你面试特别有帮助）：

**「10个银行 DevOps System Design 面试题 + 标准回答框架」**

比如：

* design enterprise CI/CD platform
* design multi-region AKS architecture
* design secure secrets management system

这些基本是 **Lead DevOps 面试的终极题型**。

根据你这个 **JD（Ansible + GitHub Actions + automation + CI/CD + security scanning）**，我整理了一组 **更偏 automation / pipeline / configuration management 的 Ansible 面试问题**。
这种问题在 **银行 DevOps / platform engineer 面试**里非常常见，因为他们特别看重 **automation maturity**。

我列 **15 个问题**，并按 **基础 → 设计 → pipeline → production troubleshooting** 排序。

---

# 🔧 Ansible Automation / Pipeline 面试问题（15题）

---

# 1 Ansible Fundamentals

### 1️⃣ What problem does Ansible solve compared to traditional shell scripting?

面试官想听

* declarative configuration
* idempotency
* parallel execution
* infrastructure consistency

Example

```text
Shell script → imperative
Ansible → desired state
```

---

### 2️⃣ What is idempotency in Ansible and why is it important for automation pipelines?

关键点

* running playbook multiple times produces same result
* prevents configuration drift
* safe automation

Example

```yaml
state: present
```

---

### 3️⃣ What are the main components of an Ansible playbook?

Expected

```text
inventory
play
tasks
roles
handlers
variables
```

---

# 2 Inventory / Environment Design

### 4️⃣ How would you manage multiple environments (dev/test/prod) using Ansible?

Expected answer

```text
inventory/
   dev
   test
   prod
```

or

```text
group_vars
host_vars
```

Example

```yaml
group_vars/prod.yml
group_vars/dev.yml
```

---

### 5️⃣ What is dynamic inventory and when would you use it?

Expected

Dynamic inventory pulls infrastructure information automatically.

Example

```text
Azure
AWS
VM scale sets
```

Tools

```text
azure_rm plugin
```

---

# 3 Ansible Roles / Code Structure

### 6️⃣ How do you structure Ansible roles to keep automation reusable?

Expected structure

ansible/
├── inventories/
│   ├── dev/
│   │   ├── hosts.yml
│   │   └── group_vars/
│   │       ├── all.yml          # dev 全局变量
│   │       ├── webservers.yml
│   │       └── databases.yml
│   ├── staging/
│   │   ├── hosts.yml
│   │   └── group_vars/
│   │       ├── all.yml
│   │       ├── webservers.yml
│   │       └── databases.yml
│   └── production/
│       ├── hosts.yml
│       └── group_vars/
│           ├── all.yml
│           ├── webservers.yml
│           └── databases.yml

Key concept

> modular and reusable automation

---

### 7️⃣ How do you manage configuration templates in Ansible?

Expected

* Jinja2 templates

Example

```jinja
server_name {{ domain }}
```

Use module

```yaml
template:
  src: nginx.conf.j2
  dest: /etc/nginx/nginx.conf
```

---

# 4 Ansible + CI/CD Pipeline

### 8️⃣ How would you integrate Ansible into a CI/CD pipeline?

Typical design

```text
CI
lint playbooks
syntax check

CD
run ansible playbook
deploy configuration
verify service
```

Tools

```text
GitHub Actions
Azure DevOps
Jenkins
```

---

### 9️⃣ What validation steps would you include before running Ansible in production?

Expected

```text
ansible-lint
syntax check
dry run
```

Example

```bash
ansible-playbook --check
```

---

### 🔟 How would you implement safe deployment using Ansible?

Expected

```text
serial deployment
rolling updates
health checks
```

Example

```yaml
serial: 2
```

---

# 5 Pipeline Reliability

### 1️⃣1️⃣ What happens if an Ansible playbook fails in the middle of execution?

Expected

* partial state change
* need retry / idempotency

Possible solutions

```text
rollback
retry tasks
checkpointing
```

---

### 1️⃣2️⃣ How do you ensure Ansible automation is safe to run multiple times?

Expected

* idempotent modules
* avoid shell commands
* use state parameters

Example

```yaml
state: present
```

---

# 6 Scaling Automation

### 1️⃣3️⃣ How would you run Ansible across hundreds of servers?

Expected

* parallel execution
* forks parameter

Example

```bash
ansible-playbook -f 50
```

---

### 1️⃣4️⃣ How do you manage secrets when using Ansible in CI/CD?

Expected

* Ansible Vault
* Azure Key Vault
* HashiCorp Vault

Example

```bash
ansible-vault encrypt
```

---

# 7 Troubleshooting Automation

### 1️⃣5️⃣ An Ansible playbook works locally but fails in CI/CD pipeline. How would you debug it?

Expected steps

```text
check inventory
check credentials
check environment variables
increase verbosity
```

Example

```bash
ansible-playbook -vvv
```

---

# ⭐ 银行 DevOps 面试特别容易问的 5 个 Ansible 问题

我帮你筛选一下 **最可能问的**：

1️⃣ What is idempotency in Ansible
2️⃣ How do you structure reusable Ansible roles
3️⃣ How do you integrate Ansible into CI/CD pipelines
4️⃣ How do you manage secrets in automation
5️⃣ How do you safely deploy changes using Ansible

---

# ⭐ 一个银行 DevOps 面试非常喜欢问的设计题

这个你可以准备一下：

> **Design an automation pipeline using Ansible to deploy application configuration across multiple environments.**

回答框架：

```text
1 Git repository for playbooks
2 CI validation (lint + syntax check)
3 pipeline approval
4 run ansible playbook
5 health check
6 monitoring
```

---

如果你愿意，我可以再帮你做一个 **非常有用的面试准备**：

### 「10个 Ansible + Kubernetes automation 场景题」

例如：

* 用 Ansible 自动化 AKS deployment
* 用 Ansible 管理 Kubernetes secrets
* 用 Ansible 自动化 CI/CD infrastructure

这些是 **DevOps Lead 面试特别爱问的 scenario question**。

