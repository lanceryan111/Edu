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


Part 1: Kubernetes / AKS

Q1: AKS 中 Pod CrashLoopBackOff 排查步骤
问题： 在 AKS 中，如果一个 Pod 一直处于 CrashLoopBackOff 状态，你的排查步骤是什么？
第一步：查看 Pod 状态和事件

kubectl describe pod <pod-name> -n <namespace>


重点关注 Events 部分，看是否有 OOMKilled、ImagePullBackOff、或 Liveness Probe failed 等信息。同时查看 Last State 中的 Exit Code：
	∙	Exit Code 0: 应用正常退出但不应该退出，检查应用逻辑
	∙	Exit Code 1: 应用崩溃，查看日志
	∙	Exit Code 137: OOMKilled，需要增加 memory limits
	∙	Exit Code 139: Segfault，代码或依赖问题
第二步：查看容器日志

kubectl logs <pod-name> -n <namespace> --previous


用 --previous 查看上一次崩溃的日志，而不是当前正在重启的容器。
第三步：检查资源配置
	∙	Resource limits/requests 是否合理（特别是 memory limit）
	∙	Liveness/Readiness Probe 配置是否正确（initialDelaySeconds 是否足够）
	∙	ConfigMap/Secret 是否存在且挂载正确
	∙	环境变量是否配置正确（数据库连接串等）
第四步：临时调试

kubectl run debug --image=<same-image> --rm -it -- /bin/sh


用相同镜像启动一个临时 Pod，手动执行启动命令来定位问题。

Q2: AKS 零停机部署策略
问题： 请描述你如何在 AKS 中实现零停机部署？Rolling update 的 maxSurge 和 maxUnavailable 怎么配？
核心配置：

strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1          # 每次多创建1个新Pod
    maxUnavailable: 0    # 保证始终有足够 Pod 服务


设置 maxUnavailable: 0 是关键，确保在新 Pod 就绪之前不会删除旧 Pod。
必须配合 Readiness Probe：

readinessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 3


Readiness Probe 告诉 K8s 这个 Pod 是否真正可以接收流量。只有当新 Pod 的 Readiness Probe 通过后，旧 Pod 才会被终止。
其他最佳实践：
	∙	PodDisruptionBudget (PDB): 设置 minAvailable 防止集群升级时误删太多 Pod
	∙	preStop hook: 给应用时间完成正在处理的请求
	∙	terminationGracePeriodSeconds: 设置合理的优雅关闭时间
	∙	蓝绿部署: 对于更关键的服务，可以用两套 Deployment + Service 切换实现

Q3: Helm vs Kustomize 选择
问题： Helm 和 Kustomize 各自的优缺点是什么？在什么场景下你会选择哪个？
Helm 优势：
	∙	模板化能力强，支持条件逻辑、循环、函数
	∙	内置版本管理和回滚（helm rollback）
	∙	丰富的生态系统，大量社区 Chart 可用
	∙	支持 hooks（pre-install, post-upgrade 等）
Kustomize 优势：
	∙	无需模板引擎，直接对原生 YAML 做 overlay
	∙	kubectl 原生集成（kubectl apply -k）
	∙	更容易理解和审计，因为没有模板逻辑
	∙	适合 GitOps 工作流
选择建议：
如果需要部署第三方应用（如 nginx-ingress, prometheus）或需要复杂的参数化，用 Helm。如果是内部应用的多环境配置差异管理，Kustomize 更轻量。实际中两者可以结合使用：Helm 生成基础 manifests，Kustomize 做环境 overlay。

Part 2: Ansible Automation Pipeline

Q4: 设计 Ansible 自动化部署 Pipeline
问题： 请设计一个 Ansible pipeline：当有新代码 push 到 main branch 时，自动通过 Ansible 部署到多台 RHEL 服务器上。
整体架构：
GitHub Actions (trigger) → Build & Test → Artifact Upload → Ansible Playbook → Target RHEL Servers
Step 1 - GitHub Actions Workflow：

on:
  push:
    branches: [main]
jobs:
  deploy:
    runs-on: [self-hosted, linux]
    steps:
      - uses: actions/checkout@v4
      - name: Build and test
        run: ./gradlew build test
      - name: Run Ansible playbook
        run: |
          ansible-playbook -i inventory/prod \
            deploy.yml \
            --extra-vars "app_version=${{ github.sha }}"


Step 2 - Inventory 管理：

# inventory/prod/hosts.yml
all:
  children:
    webservers:
      hosts:
        web01.td.com:
        web02.td.com:
    appservers:
      hosts:
        app01.td.com:
        app02.td.com:


Step 3 - Playbook 结构：

# deploy.yml
- hosts: webservers
  serial: 1              # 一台一台滚动部署
  become: yes
  roles:
    - role: pre_check      # 磁盘空间、服务状态检查
    - role: deploy_app     # 停服务、部署新版本、启服务
    - role: post_check     # 健康检查、烟雾测试


关键设计原则：
	∙	Idempotency（幂等性）：每个 task 都可以安全重复执行
	∙	serial: 1 实现滚动部署，一台失败即停止
	∙	每台部署完后做 health check 确认服务正常
	∙	使用 Ansible Vault 管理敏感信息（密码、证书）

Q5: Ansible 执行失败的处理策略
问题： 你的 Ansible playbook 在执行到一半时某台机器失败了，你怎么处理？如何确保可以安全重跑？
立即处理：
Ansible 默认会生成 retry 文件（playbook.retry），记录失败的主机。可以用 --limit 只重跑失败的机器：

ansible-playbook deploy.yml --limit @deploy.retry


在 Playbook 中的错误处理：

- block:
    - name: Stop service
      systemd: name=myapp state=stopped
    - name: Deploy new version
      copy: src=app.jar dest=/opt/myapp/
    - name: Start service
      systemd: name=myapp state=started
  rescue:
    - name: Rollback to previous version
      copy: src=/opt/myapp/app.jar.bak dest=/opt/myapp/app.jar
    - name: Restart with old version
      systemd: name=myapp state=restarted
    - name: Notify team
      slack: msg="Deploy failed on {{ inventory_hostname }}"


保证幂等性的关键做法：
	∙	用 Ansible 内置模块（copy, template, systemd）而不是 shell/command
	∙	每个操作前先备份（backup: yes 参数）
	∙	用 creates/removes 参数让 shell 任务也具备幂等性
	∙	max_fail_percentage 控制失败比例，避免全部机器都部署失败

Q6: 多环境 Inventory 和变量管理
问题： 如何管理不同环境（dev/staging/prod）的 Ansible inventory 和变量？敏感信息怎么处理？
目录结构：

inventory/
  dev/
    hosts.yml
    group_vars/
      all.yml           # dev 环境通用变量
      webservers.yml
      vault.yml          # 加密的敏感变量
  staging/
    hosts.yml
    group_vars/
      all.yml
      vault.yml
  prod/
    hosts.yml
    group_vars/
      all.yml
      vault.yml


Ansible Vault 用法：

# 加密整个文件
ansible-vault encrypt inventory/prod/group_vars/vault.yml

# 加密单个变量
ansible-vault encrypt_string 'db_password_123' --name 'db_password'

# 运行时解密
ansible-playbook deploy.yml -i inventory/prod --ask-vault-pass


变量优先级（从低到高）：
role defaults → inventory group_vars/all → inventory group_vars/<group> → inventory host_vars → play vars → extra vars
在 CI/CD 中，Vault password 存在 GitHub Actions Secrets 中，通过环境变量 ANSIBLE_VAULT_PASSWORD_FILE 传入。

Part 3: Ansible + Kubernetes 场景题

Q7: 用 Ansible 部署新 Microservice 到 AKS
问题： 场景：你需要用 Ansible 自动化在 AKS 集群上部署一个新的 microservice，包括创建 namespace、部署 Helm chart、配置 ingress、验证 pod 健康状态。
Playbook 结构：

# deploy_microservice.yml
- hosts: localhost
  connection: local
  collections:
    - kubernetes.core
  vars:
    namespace: "payment-service"
    chart_path: "./charts/payment-service"
    app_version: "{{ lookup('env', 'APP_VERSION') }}"

  tasks:
    - name: Create namespace
      k8s:
        state: present
        definition:
          apiVersion: v1
          kind: Namespace
          metadata:
            name: "{{ namespace }}"
            labels:
              env: production

    - name: Deploy Helm chart
      helm:
        name: payment-service
        chart_ref: "{{ chart_path }}"
        release_namespace: "{{ namespace }}"
        values:
          image:
            tag: "{{ app_version }}"
          ingress:
            enabled: true
            host: payment.td.com

    - name: Wait for deployment ready
      k8s_info:
        kind: Deployment
        namespace: "{{ namespace }}"
        name: payment-service
      register: deploy_status
      until: >
        deploy_status.resources[0].status.readyReplicas
        == deploy_status.resources[0].spec.replicas
      retries: 30
      delay: 10

    - name: Verify pod health
      uri:
        url: "https://payment.td.com/health"
        status_code: 200
      retries: 5
      delay: 5


关键点：
	∙	connection: local 因为我们是通过 kubeconfig 操作集群，不需要 SSH 到远程主机
	∙	kubernetes.core collection 提供 k8s、helm、k8s_info 模块
	∙	until/retries/delay 实现等待 deployment 就绪的轮询逻辑
	∙	最后通过实际 HTTP 请求验证服务健康，而不只是看 Pod 状态

Q8: 多 AKS 集群滚动更新
问题： 场景：需要在多个 AKS 集群上同时滚动更新一个应用，但要求每个集群更新完成并验证健康后才继续下一个。

# multi_cluster_update.yml
- hosts: aks_clusters
  serial: 1                    # 一个集群一个集群来
  vars:
    app_name: payment-service
    namespace: production

  tasks:
    - name: Set kubeconfig for this cluster
      set_fact:
        kubeconfig: "{{ cluster_kubeconfig }}"

    - name: Update Helm release
      helm:
        name: "{{ app_name }}"
        chart_ref: ./charts/{{ app_name }}
        release_namespace: "{{ namespace }}"
        values:
          image:
            tag: "{{ new_version }}"
      environment:
        KUBECONFIG: "{{ kubeconfig }}"

    - name: Wait for rollout complete
      k8s_info:
        kind: Deployment
        namespace: "{{ namespace }}"
        name: "{{ app_name }}"
      register: deploy
      until: >
        deploy.resources[0].status.readyReplicas ==
        deploy.resources[0].spec.replicas
      retries: 60
      delay: 10
      environment:
        KUBECONFIG: "{{ kubeconfig }}"

    - name: Run smoke test
      uri:
        url: "https://{{ cluster_endpoint }}/{{ app_name }}/health"
        status_code: 200
      retries: 3
      delay: 5

    - name: Log success
      debug:
        msg: "Cluster {{ inventory_hostname }} updated successfully"


Inventory 配置：

# inventory/clusters.yml
aks_clusters:
  hosts:
    cluster-east:
      cluster_kubeconfig: /etc/kube/east.conf
      cluster_endpoint: east.aks.td.com
    cluster-west:
      cluster_kubeconfig: /etc/kube/west.conf
      cluster_endpoint: west.aks.td.com
    cluster-central:
      cluster_kubeconfig: /etc/kube/central.conf
      cluster_endpoint: central.aks.td.com


核心设计：
	∙	serial: 1 确保一个集群完成后才处理下一个
	∙	每个集群通过不同的 KUBECONFIG 环境变量切换
	∙	如果某个集群 smoke test 失败，Playbook 会停止，不会继续部署剩余集群

Q9: 从 kubectl apply 迁移到 Ansible 自动化
问题： 你的团队想从手动 kubectl apply 迁移到 Ansible 自动化管理 K8s 资源。你会怎么规划？有哪些坑？
迁移规划（分阶段）：
	∙	Phase 1: 盘点现有资源，导出所有 YAML，建立 Git 仓库管理
	∙	Phase 2: 将 YAML 转换为 Ansible k8s 模块的 tasks，先在 dev 环境验证
	∙	Phase 3: 集成到 CI/CD pipeline，通过 PR 触发 Ansible playbook
	∙	Phase 4: 添加 drift detection，定期检查实际状态与期望状态是否一致
常见的坑：
	∙	State drift: 有人手动 kubectl edit 了资源，导致 Ansible 里的定义和实际不一致。解决：用 RBAC 限制手动操作权限 + 定期 reconciliation
	∙	已有资源冲突: 第一次运行 Ansible 时，k8s 模块会尝试创建已存在的资源。解决：用 state: present，它会自动 patch 而不是报错
	∙	Secret 管理: K8s Secrets 的 base64 编码在 Ansible 中处理很容易出错。建议用 Ansible Vault + 模板生成
Ansible vs GitOps (ArgoCD) 的取舍：
如果团队已经重度使用 Ansible，用 Ansible 管理 K8s 资源可以保持工具链统一。但如果是纯 K8s 环境且团队规模大，ArgoCD 的声明式 + 自动 reconciliation 更适合。实际中也可以 Ansible 管理基础设施（namespace, RBAC, quotas），ArgoCD 管理应用部署。

Part 4: GitHub Actions / CI/CD

Q10: GitHub Actions Secrets 管理
问题： GitHub Actions workflow 中如何安全管理 secrets？在 self-hosted runner 上访问内部 Nexus 仓库怎么配置？
Secrets 分层管理：
	∙	Organization secrets: 跨 repo 共享（如 Nexus 认证）
	∙	Repository secrets: 单个 repo 专用（如特定应用的 API key）
	∙	Environment secrets: 绑定特定环境，配合 protection rules（如 prod 环境需要审批）
Nexus 认证配置：

# 在 workflow 中使用
- name: Configure Nexus auth
  run: |
    mkdir -p ~/.gradle
    cat > ~/.gradle/gradle.properties << EOF
    nexusUser=${{ secrets.NEXUS_USERNAME }}
    nexusPass=${{ secrets.NEXUS_PASSWORD }}
    EOF

# 或者用于 curl
- name: Upload artifact to Nexus
  run: |
    curl -u "${{ secrets.NEXUS_USERNAME }}:${{ secrets.NEXUS_PASSWORD }}" \
      --upload-file app.jar \
      "https://nexus.td.com/repository/releases/"


Self-hosted Runner 安全注意：
	∙	Runner 不应存储永久化 credentials，每次从 Secrets 动态获取
	∙	使用 ephemeral runners（–ephemeral），每次 job 用完即销毁
	∙	Runner 应在独立的网络区域，只开放必要的出口端口

Q11: Monorepo 多组件 CI 策略
问题： 如何设计一个 GitHub Actions workflow 同时 build 多个 microservice 并只触发有变更的组件？
方案：Path Filter + Matrix Strategy

on:
  push:
    branches: [main]

jobs:
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      services: ${{ steps.filter.outputs.changes }}
    steps:
      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            payment: 'services/payment/**'
            auth: 'services/auth/**'
            gateway: 'services/gateway/**'

  build:
    needs: detect-changes
    if: needs.detect-changes.outputs.services != '[]'
    strategy:
      matrix:
        service: ${{ fromJson(needs.detect-changes.outputs.services) }}
    runs-on: [self-hosted, linux]
    steps:
      - uses: actions/checkout@v4
      - name: Build ${{ matrix.service }}
        run: cd services/${{ matrix.service }} && ./gradlew build
      - name: Docker build and push
        run: |
          docker build -t nexus.td.com/${{ matrix.service }}:${{ github.sha }} .
          docker push nexus.td.com/${{ matrix.service }}:${{ github.sha }}


优势：
	∙	只构建有变更的服务，节省 CI 资源
	∙	Matrix 并行执行，多个服务同时构建
	∙	易于扩展，新增服务只需在 filter 中添加一行

Part 5: Security / Veracode

Q12: Veracode 集成到 CI/CD
问题： 如何把 Veracode 安全扫描集成到 CI/CD pipeline 中？扫描发现 critical vulnerability 时怎么处理？
集成架构：

# 在 GitHub Actions 中集成 Veracode
- name: Veracode Static Analysis
  uses: veracode/veracode-uploadandscan-action@v1
  with:
    appname: 'WebBroker-PaymentService'
    filepath: 'build/libs/app.jar'
    vid: ${{ secrets.VERACODE_API_ID }}
    vkey: ${{ secrets.VERACODE_API_KEY }}
    criticality: 'VeryHigh'
    scantimeout: 30

- name: Veracode SCA (Software Composition Analysis)
  run: |
    curl -sSL https://download.sourceclear.com/ci.sh | bash
  env:
    SRCCLR_API_TOKEN: ${{ secrets.SRCCLR_TOKEN }}


Policy Gate 策略：
	∙	Critical/High: 直接 break build，不允许部署
	∙	Medium: 发警告但允许部署，创建 Jira ticket 跟踪
	∙	Low: 记录在报告中，定期 review
实际工作流：
在 PR 阶段运行 Pipeline Scan（快速、增量），在 merge 到 main 后运行 Full Scan（完整但耗时）。扫描结果自动生成报告发送到开发团队 Slack channel，并同步到 Veracode Platform 做统一管理。

Part 6: Scripting & Troubleshooting

Q13: 自动回滚 Shell 脚本
问题： 写一个 shell 脚本：检查某个 deployment 的所有 pod 是否 ready，如果 5 分钟内没有全部 ready 就自动回滚。

#!/bin/bash
DEPLOYMENT=$1
NAMESPACE=${2:-default}
TIMEOUT=300  # 5 minutes
INTERVAL=10

echo "Monitoring deployment: $DEPLOYMENT in $NAMESPACE"

elapsed=0
while [ $elapsed -lt $TIMEOUT ]; do
  # Get desired and ready replica counts
  DESIRED=$(kubectl get deploy $DEPLOYMENT -n $NAMESPACE \
    -o jsonpath='{.spec.replicas}')
  READY=$(kubectl get deploy $DEPLOYMENT -n $NAMESPACE \
    -o jsonpath='{.status.readyReplicas}')
  READY=${READY:-0}

  echo "[${elapsed}s] Ready: $READY / $DESIRED"

  if [ "$READY" -eq "$DESIRED" ] && [ "$DESIRED" -gt 0 ]; then
    echo "All pods ready! Deployment successful."
    exit 0
  fi

  sleep $INTERVAL
  elapsed=$((elapsed + INTERVAL))
done

echo "TIMEOUT! Rolling back $DEPLOYMENT..."
kubectl rollout undo deployment/$DEPLOYMENT -n $NAMESPACE

# Wait for rollback
kubectl rollout status deployment/$DEPLOYMENT -n $NAMESPACE --timeout=120s
echo "Rollback complete."
exit 1


用法：

./auto_rollback.sh payment-service production


Q14: Terraform State Drift 处理
问题： 一个 Terraform 管理的 Azure 资源和实际环境出现了 drift，你怎么发现和解决？
发现 Drift：

# 检测实际状态与 state 文件的差异
terraform plan -refresh-only

# 在 CI 中定期运行
terraform plan -detailed-exitcode
# Exit code 0 = no changes, 2 = changes detected


可以在 GitHub Actions 中设置定时任务（cron）每天运行 plan，发现 drift 时发 Slack 警报。
解决方案（根据情况选择）：
	∙	手动修改是误操作: terraform apply 将资源恢复到期望状态
	∙	手动修改是必要的: 更新 Terraform 代码匹配实际状态，然后 terraform plan 确认无差异
	∙	资源被手动创建: terraform import 将其纳入 state 管理
预防措施：
	∙	用 Azure Policy 限制手动修改（比如禁止直接通过 Portal 修改 tag）
	∙	State file 存在 Azure Storage Account，启用 state locking
	∙	PR review 强制要求所有基础设施变更通过 Terraform
	∙	terraform plan 输出附在 PR comment 中，审核人可以看到具体变更

Part 7: 综合架构题

Q15: WebBroker 完整 DevOps 流水线设计
问题： 请设计 WebBroker 平台的一个完整 DevOps 流水线：从开发者 push 代码，到最终部署到 AKS 生产环境。
完整流水线架构：
Developer → Feature Branch → PR → CI Pipeline → Staging → Approval → Production
Stage 1: 代码提交与 PR
	∙	开发者在 feature branch 开发，提交 PR 到 main
	∙	Branch protection: 要求至少 2 人 review，CI 必须通过
Stage 2: CI Pipeline（PR 触发）
	∙	Build: Gradle build，运行 unit tests
	∙	Static Analysis: Veracode Pipeline Scan（快速扫描）
	∙	SCA: 依赖库漏洞检查（Veracode SCA）
	∙	Docker Build: 构建镜像并推送到 Nexus/ACR
	∙	Helm Lint: 验证 Helm chart 语法
Stage 3: Deploy to Staging
	∙	自动触发：merge 到 main 后自动部署到 staging
	∙	Ansible playbook 执行 Helm upgrade 到 staging AKS 集群
	∙	运行 Integration tests 和 Smoke tests
	∙	Veracode Full Scan（完整静态分析）
Stage 4: Production Approval
	∙	GitHub Environment protection rule: 需要 Tech Lead 审批
	∙	Change ticket 自动创建（符合 TD 变更管理流程）
Stage 5: Deploy to Production
	∙	Ansible playbook 执行蓝绿部署到 prod AKS
	∙	Canary 测试：先将 10% 流量切到新版本
	∙	监控 metrics（响应时间、错误率），确认正常后全量切换
	∙	自动回滚机制：如果错误率超阈值自动 rollback
Stage 6: Post-deployment
	∙	Azure Monitor + Prometheus 监控应用指标
	∙	PagerDuty 告警集成
	∙	部署结果通知 Slack channel
工具链总结：
	∙	代码管理: GitHub Enterprise
	∙	CI/CD: GitHub Actions (self-hosted runners on AKS)
	∙	制品仓库: Nexus / Azure Container Registry
	∙	配置管理: Ansible + Ansible Vault
	∙	容器编排: AKS + Helm
	∙	安全扫描: Veracode (SAST + SCA)
	∙	基础设施: Terraform + Azure
	∙	监控: Azure Monitor + Prometheus + Grafana
	∙	网络: Azure Application Gateway + Azure Networking​​​​​​​​​​​​​​​​
