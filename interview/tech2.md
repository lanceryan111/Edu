Kubernetes / AKS (重点)

在 AKS 中，如果一个 Pod 一直处于 CrashLoopBackOff 状态，你的排查步骤是什么？ (考察 kubectl logs, describe pod, events, resource limits, liveness/readiness probes)
请描述你如何在 AKS 中实现零停机部署（zero-downtime deployment）？Rolling update 的 maxSurge 和 maxUnavailable 怎么配？ (考察 deployment strategy, readiness probe 配合)
Helm 和 Kustomize 各自的优缺点是什么？在什么场景下你会选择哪个？TD 的 EDP 平台用的是哪种方式？ (考察实际工程判断)


Ansible Automation Pipeline (重点)

请设计一个 Ansible pipeline：当有新代码 push 到 main branch 时，自动通过 Ansible 部署到多台 RHEL 服务器上。你会怎么架构这个流程？ (场景题：考察 GitHub Actions trigger → Ansible playbook → inventory management → idempotency)
你的 Ansible playbook 在执行到一半时某台机器失败了，你怎么处理？如何确保可以安全地重跑而不影响已成功的机器？ (考察 --limit, retry files, idempotency, serial 关键字, error handling with block/rescue)
如何管理不同环境（dev/staging/prod）的 Ansible inventory 和变量？你用 group_vars/host_vars 还是 Ansible Vault？敏感信息怎么处理？ (考察 inventory 结构, ansible-vault encrypt_string, 变量优先级)


Ansible + Kubernetes Automation 场景题 (重点)

场景：你需要用 Ansible 自动化以下流程 — 在 AKS 集群上部署一个新的 microservice，包括创建 namespace、部署 Helm chart、配置 ingress、验证 pod 健康状态。请描述你的 playbook 结构。 (考察 kubernetes.core collection, helm module, k8s module, 任务编排顺序, wait_for 验证)
场景：需要在多个 AKS 集群上同时滚动更新一个应用，但要求每个集群更新完成并验证健康后才继续下一个。用 Ansible 怎么实现？ (考察 serial: 1, health check tasks, delegate_to, 以及 k8s_info 模块验证 deployment ready replicas)
你的团队想从手动 kubectl apply 迁移到 Ansible 自动化管理 K8s 资源。你会怎么规划这个迁移？有哪些坑需要注意？ (考察 state drift 管理, 已有资源的 import, ansible vs GitOps 如 ArgoCD 的取舍)


GitHub Actions / CI/CD

GitHub Actions workflow 中如何安全地管理 secrets？如果需要在 self-hosted runner 上访问内部 Nexus 仓库，你怎么配置认证？ (考察 secrets, environment protection rules, runner 上的 credential 管理)
如何设计一个 GitHub Actions workflow 同时 build 多个 microservice 并只触发有变更的组件？ (考察 path filter, matrix strategy, monorepo CI 策略)


Security / Veracode

如何把 Veracode 安全扫描集成到 CI/CD pipeline 中？扫描发现 critical vulnerability 时 pipeline 应该怎么处理？ (考察 Veracode CLI integration, policy gate, break build on severity threshold)


Scripting & Troubleshooting

写一个 shell 脚本：检查某个 deployment 的所有 pod 是否 ready，如果5分钟内没有全部 ready 就自动回滚到上一个版本。 (考察 kubectl rollout status, timeout handling, rollback command)
一个 Terraform 管理的 Azure 资源和实际环境出现了 drift，你怎么发现和解决？ (考察 terraform plan, state management, import, 以及防止手动修改的策略)


综合架构题

请设计 WebBroker 平台的一个完整 DevOps 流水线：从开发者 push 代码，到最终部署到 AKS 生产环境，包括安全扫描、审批、蓝绿部署。画出整体架构并说明每个环节用什么工具。 (综合考察：GitHub Actions + Veracode + Ansible + Helm + AKS + Azure Networking + monitoring)
