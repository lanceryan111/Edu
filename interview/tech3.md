# DevOps Engineer 面试技术问题参考答案

**TD Bank - WebBroker Platform**

基于 Job Description 的 25 道核心技术面试题

---

## Part 1: Kubernetes / AKS

---

### Q1: AKS 中 Pod CrashLoopBackOff 排查步骤

**问题：** 在 AKS 中，如果一个 Pod 一直处于 CrashLoopBackOff 状态，你的排查步骤是什么？

**参考答案：**

**第一步：查看 Pod 状态和事件**

```bash
kubectl describe pod <pod-name> -n <namespace>
```

重点关注 Events 部分，看是否有 OOMKilled、ImagePullBackOff、或 Liveness Probe failed 等信息。同时查看 Last State 中的 Exit Code：

- Exit Code 0: 应用正常退出但不应该退出，检查应用逻辑
- Exit Code 1: 应用崩溃，查看日志
- Exit Code 137: OOMKilled，需要增加 memory limits
- Exit Code 139: Segfault，代码或依赖问题

**第二步：查看容器日志**

```bash
kubectl logs <pod-name> -n <namespace> --previous
```

用 `--previous` 查看上一次崩溃的日志，而不是当前正在重启的容器。

**第三步：检查资源配置**

- Resource limits/requests 是否合理（特别是 memory limit）
- Liveness/Readiness Probe 配置是否正确（initialDelaySeconds 是否足够）
- ConfigMap/Secret 是否存在且挂载正确
- 环境变量是否配置正确（数据库连接串等）

**第四步：临时调试**

```bash
kubectl run debug --image=<same-image> --rm -it -- /bin/sh
```

用相同镜像启动一个临时 Pod，手动执行启动命令来定位问题。

---

### Q2: AKS 零停机部署策略

**问题：** 请描述你如何在 AKS 中实现零停机部署（zero-downtime deployment）？Rolling update 的 maxSurge 和 maxUnavailable 怎么配？

**参考答案：**

**核心配置：**

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1          # 每次多创建1个新Pod
    maxUnavailable: 0    # 保证始终有足够 Pod 服务
```

设置 `maxUnavailable: 0` 是关键，确保在新 Pod 就绪之前不会删除旧 Pod。

**必须配合 Readiness Probe：**

```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 3
```

Readiness Probe 告诉 K8s 这个 Pod 是否真正可以接收流量。只有当新 Pod 的 Readiness Probe 通过后，旧 Pod 才会被终止。

**其他最佳实践：**

- PodDisruptionBudget (PDB): 设置 minAvailable 防止集群升级时误删太多 Pod
- preStop hook: 给应用时间完成正在处理的请求
- terminationGracePeriodSeconds: 设置合理的优雅关闭时间
- 蓝绿部署: 对于更关键的服务，可以用两套 Deployment + Service 切换实现

---

### Q3: Helm vs Kustomize 选择

**问题：** Helm 和 Kustomize 各自的优缺点是什么？在什么场景下你会选择哪个？

**参考答案：**

**Helm 优势：**

- 模板化能力强，支持条件逻辑、循环、函数
- 内置版本管理和回滚（helm rollback）
- 丰富的生态系统，大量社区 Chart 可用
- 支持 hooks（pre-install, post-upgrade 等）

**Kustomize 优势：**

- 无需模板引擎，直接对原生 YAML 做 overlay
- kubectl 原生集成（kubectl apply -k）
- 更容易理解和审计，因为没有模板逻辑
- 适合 GitOps 工作流

**选择建议：**

如果需要部署第三方应用（如 nginx-ingress, prometheus）或需要复杂的参数化，用 Helm。如果是内部应用的多环境配置差异管理，Kustomize 更轻量。实际中两者可以结合使用：Helm 生成基础 manifests，Kustomize 做环境 overlay。

---

## Part 2: Ansible Automation Pipeline

---

### Q4: 设计 Ansible 自动化部署 Pipeline

**问题：** 请设计一个 Ansible pipeline：当有新代码 push 到 main branch 时，自动通过 Ansible 部署到多台 RHEL 服务器上。

**参考答案：**

**整体架构：**

GitHub Actions (trigger) → Build & Test → Artifact Upload → Ansible Playbook → Target RHEL Servers

**Step 1 - GitHub Actions Workflow：**

```yaml
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
```

**Step 2 - Inventory 管理：**

```yaml
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
```

**Step 3 - Playbook 结构：**

```yaml
# deploy.yml
- hosts: webservers
  serial: 1              # 一台一台滚动部署
  become: yes
  roles:
    - role: pre_check      # 磁盘空间、服务状态检查
    - role: deploy_app     # 停服务、部署新版本、启服务
    - role: post_check     # 健康检查、烟雾测试
```

**关键设计原则：**

- Idempotency（幂等性）：每个 task 都可以安全重复执行
- serial: 1 实现滚动部署，一台失败即停止
- 每台部署完后做 health check 确认服务正常
- 使用 Ansible Vault 管理敏感信息（密码、证书）

---

### Q5: Ansible 执行失败的处理策略

**问题：** 你的 Ansible playbook 在执行到一半时某台机器失败了，你怎么处理？如何确保可以安全重跑？

**参考答案：**

**立即处理：**

Ansible 默认会生成 retry 文件（playbook.retry），记录失败的主机。可以用 `--limit` 只重跑失败的机器：

```bash
ansible-playbook deploy.yml --limit @deploy.retry
```

**在 Playbook 中的错误处理：**

```yaml
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
```

**保证幂等性的关键做法：**

- 用 Ansible 内置模块（copy, template, systemd）而不是 shell/command
- 每个操作前先备份（backup: yes 参数）
- 用 creates/removes 参数让 shell 任务也具备幂等性
- max_fail_percentage 控制失败比例，避免全部机器都部署失败

---

### Q6: 多环境 Inventory 和变量管理

**问题：** 如何管理不同环境（dev/staging/prod）的 Ansible inventory 和变量？敏感信息怎么处理？

**参考答案：**

**目录结构：**

```
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
```

**Ansible Vault 用法：**

```bash
# 加密整个文件
ansible-vault encrypt inventory/prod/group_vars/vault.yml

# 加密单个变量
ansible-vault encrypt_string 'db_password_123' --name 'db_password'

# 运行时解密
ansible-playbook deploy.yml -i inventory/prod --ask-vault-pass
```

**变量优先级（从低到高）：**

role defaults → inventory group_vars/all → inventory group_vars/\<group\> → inventory host_vars → play vars → extra vars

在 CI/CD 中，Vault password 存在 GitHub Actions Secrets 中，通过环境变量 ANSIBLE_VAULT_PASSWORD_FILE 传入。

---

## Part 3: Ansible + Kubernetes 场景题

---

### Q7: 用 Ansible 部署新 Microservice 到 AKS

**问题：** 场景：你需要用 Ansible 自动化在 AKS 集群上部署一个新的 microservice，包括创建 namespace、部署 Helm chart、配置 ingress、验证 pod 健康状态。

**参考答案：**

```yaml
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
```

**关键点：**

- `connection: local` 因为我们是通过 kubeconfig 操作集群，不需要 SSH 到远程主机
- `kubernetes.core` collection 提供 k8s、helm、k8s_info 模块
- until/retries/delay 实现等待 deployment 就绪的轮询逻辑
- 最后通过实际 HTTP 请求验证服务健康，而不只是看 Pod 状态

---

### Q8: 多 AKS 集群滚动更新

**问题：** 场景：需要在多个 AKS 集群上同时滚动更新一个应用，但要求每个集群更新完成并验证健康后才继续下一个。

**参考答案：**

```yaml
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
```

**Inventory 配置：**

```yaml
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
```

**核心设计：**

- `serial: 1` 确保一个集群完成后才处理下一个
- 每个集群通过不同的 KUBECONFIG 环境变量切换
- 如果某个集群 smoke test 失败，Playbook 会停止，不会继续部署剩余集群

---

### Q9: 从 kubectl apply 迁移到 Ansible 自动化

**问题：** 你的团队想从手动 kubectl apply 迁移到 Ansible 自动化管理 K8s 资源。你会怎么规划？有哪些坑？

**参考答案：**

**迁移规划（分阶段）：**

- Phase 1: 盘点现有资源，导出所有 YAML，建立 Git 仓库管理
- Phase 2: 将 YAML 转换为 Ansible k8s 模块的 tasks，先在 dev 环境验证
- Phase 3: 集成到 CI/CD pipeline，通过 PR 触发 Ansible playbook
- Phase 4: 添加 drift detection，定期检查实际状态与期望状态是否一致

**常见的坑：**

- **State drift:** 有人手动 kubectl edit 了资源，导致 Ansible 里的定义和实际不一致。解决：用 RBAC 限制手动操作权限 + 定期 reconciliation
- **已有资源冲突:** 第一次运行 Ansible 时，k8s 模块会尝试创建已存在的资源。解决：用 `state: present`，它会自动 patch 而不是报错
- **Secret 管理:** K8s Secrets 的 base64 编码在 Ansible 中处理很容易出错。建议用 Ansible Vault + 模板生成

**Ansible vs GitOps (ArgoCD) 的取舍：**

如果团队已经重度使用 Ansible，用 Ansible 管理 K8s 资源可以保持工具链统一。但如果是纯 K8s 环境且团队规模大，ArgoCD 的声明式 + 自动 reconciliation 更适合。实际中也可以 Ansible 管理基础设施（namespace, RBAC, quotas），ArgoCD 管理应用部署。

---

## Part 4: GitHub Actions / CI/CD

---

### Q10: GitHub Actions Secrets 管理

**问题：** GitHub Actions workflow 中如何安全管理 secrets？在 self-hosted runner 上访问内部 Nexus 仓库怎么配置？

**参考答案：**

**Secrets 分层管理：**

- Organization secrets: 跨 repo 共享（如 Nexus 认证）
- Repository secrets: 单个 repo 专用（如特定应用的 API key）
- Environment secrets: 绑定特定环境，配合 protection rules（如 prod 环境需要审批）

**Nexus 认证配置：**

```yaml
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
```

**Self-hosted Runner 安全注意：**

- Runner 不应存储永久化 credentials，每次从 Secrets 动态获取
- 使用 ephemeral runners（--ephemeral），每次 job 用完即销毁
- Runner 应在独立的网络区域，只开放必要的出口端口

---

### Q11: Monorepo 多组件 CI 策略

**问题：** 如何设计一个 GitHub Actions workflow 同时 build 多个 microservice 并只触发有变更的组件？

**参考答案：**

**方案：Path Filter + Matrix Strategy**

```yaml
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
```

**优势：**

- 只构建有变更的服务，节省 CI 资源
- Matrix 并行执行，多个服务同时构建
- 易于扩展，新增服务只需在 filter 中添加一行

---

## Part 5: Security / Veracode

---

### Q12: Veracode 集成到 CI/CD

**问题：** 如何把 Veracode 安全扫描集成到 CI/CD pipeline 中？扫描发现 critical vulnerability 时怎么处理？

**参考答案：**

**集成架构：**

```yaml
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
```

**Policy Gate 策略：**

- Critical/High: 直接 break build，不允许部署
- Medium: 发警告但允许部署，创建 Jira ticket 跟踪
- Low: 记录在报告中，定期 review

**实际工作流：**

在 PR 阶段运行 Pipeline Scan（快速、增量），在 merge 到 main 后运行 Full Scan（完整但耗时）。扫描结果自动生成报告发送到开发团队 Slack channel，并同步到 Veracode Platform 做统一管理。

---

## Part 6: Scripting & Troubleshooting

---

### Q13: 自动回滚 Shell 脚本

**问题：** 写一个 shell 脚本：检查某个 deployment 的所有 pod 是否 ready，如果 5 分钟内没有全部 ready 就自动回滚。

**参考答案：**

```bash
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
```

**用法：**

```bash
./auto_rollback.sh payment-service production
```

---

### Q14: Terraform State Drift 处理

**问题：** 一个 Terraform 管理的 Azure 资源和实际环境出现了 drift，你怎么发现和解决？

**参考答案：**

**发现 Drift：**

```bash
# 检测实际状态与 state 文件的差异
terraform plan -refresh-only

# 在 CI 中定期运行
terraform plan -detailed-exitcode
# Exit code 0 = no changes, 2 = changes detected
```

可以在 GitHub Actions 中设置定时任务（cron）每天运行 plan，发现 drift 时发 Slack 警报。

**解决方案（根据情况选择）：**

- 手动修改是误操作: `terraform apply` 将资源恢复到期望状态
- 手动修改是必要的: 更新 Terraform 代码匹配实际状态，然后 `terraform plan` 确认无差异
- 资源被手动创建: `terraform import` 将其纳入 state 管理

**预防措施：**

- 用 Azure Policy 限制手动修改（比如禁止直接通过 Portal 修改 tag）
- State file 存在 Azure Storage Account，启用 state locking
- PR review 强制要求所有基础设施变更通过 Terraform
- terraform plan 输出附在 PR comment 中，审核人可以看到具体变更

---

## Part 7: 综合架构题

---

### Q15: WebBroker 完整 DevOps 流水线设计

**问题：** 请设计 WebBroker 平台的一个完整 DevOps 流水线：从开发者 push 代码，到最终部署到 AKS 生产环境。

**参考答案：**

**完整流水线架构：**

Developer → Feature Branch → PR → CI Pipeline → Staging → Approval → Production

**Stage 1: 代码提交与 PR**

- 开发者在 feature branch 开发，提交 PR 到 main
- Branch protection: 要求至少 2 人 review，CI 必须通过

**Stage 2: CI Pipeline（PR 触发）**

- Build: Gradle build，运行 unit tests
- Static Analysis: Veracode Pipeline Scan（快速扫描）
- SCA: 依赖库漏洞检查（Veracode SCA）
- Docker Build: 构建镜像并推送到 Nexus/ACR
- Helm Lint: 验证 Helm chart 语法

**Stage 3: Deploy to Staging**

- 自动触发：merge 到 main 后自动部署到 staging
- Ansible playbook 执行 Helm upgrade 到 staging AKS 集群
- 运行 Integration tests 和 Smoke tests
- Veracode Full Scan（完整静态分析）

**Stage 4: Production Approval**

- GitHub Environment protection rule: 需要 Tech Lead 审批
- Change ticket 自动创建（符合 TD 变更管理流程）

**Stage 5: Deploy to Production**

- Ansible playbook 执行蓝绿部署到 prod AKS
- Canary 测试：先将 10% 流量切到新版本
- 监控 metrics（响应时间、错误率），确认正常后全量切换
- 自动回滚机制：如果错误率超阈值自动 rollback

**Stage 6: Post-deployment**

- Azure Monitor + Prometheus 监控应用指标
- PagerDuty 告警集成
- 部署结果通知 Slack channel

**工具链总结：**

- 代码管理: GitHub Enterprise
- CI/CD: GitHub Actions (self-hosted runners on AKS)
- 制品仓库: Nexus / Azure Container Registry
- 配置管理: Ansible + Ansible Vault
- 容器编排: AKS + Helm
- 安全扫描: Veracode (SAST + SCA)
- 基础设施: Terraform + Azure
- 监控: Azure Monitor + Prometheus + Grafana
- 网络: Azure Application Gateway + Azure Networking

---

## Part 8: Build Tools / Artifact Management

---

### Q16: Maven vs npm 构建流程差异

**问题：** 你同时管理 Java (Maven) 和 Node.js (npm) 项目的 CI/CD pipeline，两者在构建、依赖管理、制品发布上有什么关键差异？你怎么统一管理？

**参考答案：**

**依赖管理差异：**

- Maven: `pom.xml` 定义依赖，从 Maven Central 或 Nexus 私服下载 JAR，依赖树是确定性的（通过 `dependency:tree` 分析）
- npm: `package.json` + `package-lock.json`，从 npm registry 或 Nexus npm proxy 下载，lock 文件保证确定性

**构建差异：**

```bash
# Maven
mvn clean package -DskipTests=false
# 产出: target/app.jar 或 target/app.war

# npm
npm ci          # 比 npm install 更严格，完全按 lock 文件安装
npm run build   # 产出: dist/ 目录
```

**Nexus 统一管理：**

- Maven: 配置 `settings.xml` 指向 Nexus hosted/proxy repo
- npm: 配置 `.npmrc` 指向 Nexus npm group repo

```bash
# .npmrc for TD internal
registry=https://nexus.td.com/repository/npm-group/
//nexus.td.com/repository/npm-group/:_auth=${NPM_TOKEN}
```

**在 GitHub Actions 中的统一策略：**

- 两者都通过 Nexus proxy 拉依赖，减少外网依赖
- 构建产物统一推送到 Nexus hosted repo
- Docker image 统一推送到 ACR/Nexus Docker registry

---

### Q17: Maven Staging to Release 发布流程

**问题：** 解释一下 Nexus 中 Maven artifact 从 staging 到 release 的流程。如果 staging repo 中的 artifact promote 失败，你怎么排查？

**参考答案：**

**标准流程：**

1. `mvn deploy` 将 SNAPSHOT 或 release artifact 推送到 Nexus staging repo
2. 在 staging 中做验证（checksum、签名、依赖完整性）
3. 通过 Nexus REST API 或 UI 执行 promote/close 操作
4. Promote 成功后 artifact 进入 release repo，对所有人可见

**Promote 失败常见原因：**

- **Version 冲突：** Release repo 不允许覆盖已有版本。解决：确保版本号递增，用 `-SNAPSHOT` 做开发版本
- **Missing POM：** artifact 上传了 JAR 但缺少 POM。解决：确保 `mvn deploy` 完整执行
- **Checksum 不匹配：** 网络问题导致上传不完整。解决：重新 deploy
- **Rule violation：** Nexus 的 staging rule 检查失败（如缺少 javadoc JAR）

**排查命令：**

```bash
# 查看 staging repo 内容
curl -u admin:pass "https://nexus.td.com/service/rest/v1/staging/repository/<repo-id>"

# 查看 promote 活动日志
curl -u admin:pass "https://nexus.td.com/service/rest/v1/staging/activities/<repo-id>"
```

---

## Part 9: Docker / Container Best Practices

---

### Q18: Dockerfile 最佳实践

**问题：** 如何编写一个安全、高效的 Dockerfile？你在 TD 构建容器镜像时遵循哪些 best practices？

**参考答案：**

**Multi-stage build 示例：**

```dockerfile
# Stage 1: Build
FROM maven:3.9-eclipse-temurin-17 AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline    # 利用 Docker layer cache
COPY src/ ./src/
RUN mvn package -DskipTests

# Stage 2: Runtime
FROM eclipse-temurin:17-jre-alpine
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
WORKDIR /app
COPY --from=builder /app/target/app.jar .
USER appuser
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget -qO- http://localhost:8080/health || exit 1
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**关键 Best Practices：**

- **Multi-stage build：** 构建环境和运行环境分离，最终镜像只有 JRE，不含 Maven 和源码
- **Non-root user：** 用 `USER appuser` 运行应用，不用 root
- **Layer cache 优化：** 先 COPY pom.xml 再 COPY src，依赖不变时跳过下载
- **Alpine 基础镜像：** 更小的攻击面和镜像体积
- **HEALTHCHECK：** 容器层面的健康检查
- **不存敏感信息：** 不在 Dockerfile 中写密码，用运行时环境变量或 K8s Secret 注入
- **固定版本号：** 基础镜像用具体 tag 而不是 `latest`
- **.dockerignore：** 排除不需要的文件减少 build context

**TD 特定要求：**

- 使用 TD 批准的基础镜像（从内部 registry 拉取）
- 镜像扫描：构建后用 Veracode Container Security 或 Trivy 扫描漏洞
- 镜像签名：用 cosign 签名确保完整性

---

## Part 10: Python / Shell Scripting

---

### Q19: Python 自动化脚本场景题

**问题：** 写一个 Python 脚本的思路：自动检查所有 GitHub Actions workflow 的运行状态，如果有连续失败 3 次以上的 workflow，发送邮件通知团队。

**参考答案：**

```python
import requests
import smtplib
from email.mime.text import MIMEText
from collections import defaultdict

GITHUB_API = "https://github.td.com/api/v3"
TOKEN = os.environ["GH_TOKEN"]
HEADERS = {"Authorization": f"token {TOKEN}"}
REPO = "td-bank/webbroker-platform"

def get_workflow_runs():
    url = f"{GITHUB_API}/repos/{REPO}/actions/runs"
    params = {"per_page": 100, "status": "completed"}
    resp = requests.get(url, headers=HEADERS, params=params)
    return resp.json()["workflow_runs"]

def find_consecutive_failures(runs):
    # Group by workflow name
    workflows = defaultdict(list)
    for run in runs:
        workflows[run["name"]].append(run["conclusion"])

    alerts = []
    for name, conclusions in workflows.items():
        # Check first 3 runs (most recent)
        recent = conclusions[:3]
        if len(recent) >= 3 and all(c == "failure" for c in recent):
            alerts.append(name)
    return alerts

def send_alert(failed_workflows):
    body = "The following workflows have failed 3+ times:\n\n"
    for wf in failed_workflows:
        body += f"  - {wf}\n"
    body += f"\nCheck: https://github.td.com/{REPO}/actions"

    msg = MIMEText(body)
    msg["Subject"] = "[ALERT] CI Workflow Failures"
    msg["From"] = "ci-alerts@td.com"
    msg["To"] = "devops-team@td.com"

    with smtplib.SMTP("smtp-relay.td.com", 25) as server:
        server.send_message(msg)

if __name__ == "__main__":
    runs = get_workflow_runs()
    failures = find_consecutive_failures(runs)
    if failures:
        send_alert(failures)
        print(f"Alert sent for: {failures}")
    else:
        print("All workflows healthy")
```

**关键设计点：**

- 用 GitHub REST API 获取 workflow runs，按 workflow name 分组
- 判断最近 3 次是否都是 failure
- 邮件通过 TD 内部 SMTP relay 发送，不需要外部邮件服务
- 可以作为 cron job 在 GitHub Actions 中定期运行

---

### Q20: Shell 脚本 — 日志收集与分析

**问题：** 写一个 shell 脚本思路：收集多台服务器上某个应用最近 1 小时的错误日志，汇总后生成报告。

**参考答案：**

```bash
#!/bin/bash
SERVERS="web01 web02 app01 app02"
LOG_PATH="/var/log/myapp/application.log"
REPORT="/tmp/error_report_$(date +%Y%m%d_%H%M).txt"
SINCE=$(date -d '1 hour ago' '+%Y-%m-%d %H:%M')

echo "Error Report - Generated at $(date)" > $REPORT
echo "Covering errors since: $SINCE" >> $REPORT
echo "========================================" >> $REPORT

for server in $SERVERS; do
  echo "" >> $REPORT
  echo "--- $server ---" >> $REPORT

  # 收集最近1小时的ERROR级别日志
  error_count=$(ssh $server \
    "awk -v since='$SINCE' '\$0 >= since && /ERROR/' $LOG_PATH | wc -l")

  ssh $server \
    "awk -v since='$SINCE' '\$0 >= since && /ERROR/' $LOG_PATH | tail -20" \
    >> $REPORT

  echo "Total errors: $error_count" >> $REPORT
done

# 汇总
echo "" >> $REPORT
echo "======== SUMMARY ========" >> $REPORT
for server in $SERVERS; do
  count=$(ssh $server \
    "awk -v since='$SINCE' '\$0 >= since && /ERROR/' $LOG_PATH | wc -l")
  echo "$server: $count errors" >> $REPORT
done

echo "Report saved to: $REPORT"
```

**注意：** 在实际工作中，这种任务更适合用 Ansible 来做：

```yaml
- hosts: all
  tasks:
    - name: Collect recent errors
      shell: >
        awk -v since="$(date -d '1 hour ago' '+%Y-%m-%d %H:%M')"
        '$0 >= since && /ERROR/' /var/log/myapp/application.log
      register: error_logs
    - name: Save to local report
      local_action:
        module: copy
        content: "{{ error_logs.stdout }}"
        dest: "/tmp/errors_{{ inventory_hostname }}.log"
```

---

## Part 11: JIRA / Confluence / 文档

---

### Q21: DevOps 文档和流程管理

**问题：** 作为 DevOps lead，你如何在 JIRA 和 Confluence 中管理 DevOps 改进项目？如何确保团队的知识共享？

**参考答案：**

**JIRA 管理策略：**

- 创建专门的 "DevOps Improvements" Epic，下面拆分 Stories
- 每个改进项用 Story 跟踪，包含明确的 acceptance criteria
- 使用 Labels 分类：`pipeline`, `security`, `infra`, `automation`
- Sprint planning 中为 DevOps 改进预留 20% 容量

**JIRA Story 模板示例：**

```
Title: Integrate Veracode SCA into PR pipeline
Epic: DevOps Improvements Q1
Labels: security, pipeline
Acceptance Criteria:
  - SCA scan runs on every PR
  - Critical vulnerabilities block merge
  - Results posted as PR comment
  - Documentation updated in Confluence
```

**Confluence 知识库结构：**

- Runbooks: 每个服务的部署、回滚、故障排查步骤
- Architecture Decision Records (ADR): 记录为什么选择某个工具或方案
- Pipeline Documentation: 每条 pipeline 的配置说明和触发条件
- Onboarding Guide: 新人上手的所有链接和步骤

**确保知识共享：**

- 每次 DevOps 改进完成后必须更新 Confluence 文档
- 定期 "DevOps Office Hours" 分享新工具和最佳实践
- PR review 中包含对应的文档更新
- Post-incident review 后更新 runbook

---

## Part 12: Distributed Systems / SpringBoot & Angular

---

### Q22: 分布式系统概念

**问题：** 在 WebBroker 这种分布式交易系统中，你作为 DevOps 需要关注哪些分布式系统的挑战？如何在基础设施层面应对？

**参考答案：**

**核心挑战和 DevOps 应对：**

**1. 服务发现和负载均衡**

- 在 AKS 中用 Kubernetes Service 做服务发现
- Azure Application Gateway 做外部负载均衡
- Ingress Controller 管理内部路由

**2. 高可用和故障转移**

- Pod replicas 跨多个 Availability Zone 部署
- PodDisruptionBudget 保证滚动更新时的最小可用数
- AKS 集群配置多个 node pool，分散风险

**3. 可观测性（Observability）**

- Distributed tracing: 接入 Azure Application Insights 或 Jaeger
- Centralized logging: 所有 Pod 日志收集到 Azure Log Analytics
- Metrics: Prometheus + Grafana 监控 API 延迟、错误率、吞吐量

**4. 配置一致性**

- ConfigMap 和 Secret 统一管理应用配置
- Ansible 确保所有环境的配置一致性
- 环境变量通过 Helm values 注入，不同环境不同 values 文件

**5. Circuit Breaker 和 Retry**

- 虽然这是应用层（SpringBoot Resilience4j）的事情，但 DevOps 需要配置相应的监控告警
- Kubernetes 的 liveness/readiness probe 是基础设施层面的健康检查

---

### Q23: SpringBoot + Angular 应用的 CI/CD 特点

**问题：** WebBroker 使用 SpringBoot 后端和 Angular 前端，在 CI/CD pipeline 中有什么特殊考虑？

**参考答案：**

**前后端分离的 Pipeline 设计：**

```yaml
# 后端 (SpringBoot)
backend-build:
  steps:
    - mvn clean package              # 编译 + 单元测试
    - mvn verify                     # 集成测试
    - veracode-scan app.jar          # 安全扫描
    - docker build -f Dockerfile.backend .
    - docker push nexus.td.com/webbroker-api:$VERSION

# 前端 (Angular)
frontend-build:
  steps:
    - npm ci                         # 安装依赖
    - npm run lint                   # 代码规范检查
    - npm run test -- --no-watch     # 单元测试 (Karma/Jest)
    - npm run build -- --prod        # 生产构建 (AOT编译, tree-shaking)
    - veracode-sca scan              # 依赖漏洞扫描
    - docker build -f Dockerfile.frontend .
    - docker push nexus.td.com/webbroker-ui:$VERSION
```

**特殊考虑：**

- **Angular AOT 编译：** `--prod` flag 启用 Ahead-of-Time 编译，构建时间较长但运行更快
- **环境配置注入：** Angular 的 `environment.ts` 在构建时确定，需要为每个环境构建不同镜像，或者用 runtime config（nginx 注入环境变量）
- **API Gateway 配置：** 前端和后端部署在不同 Pod，需要配置 Ingress 路由规则
- **版本协调：** 前后端版本需要兼容，可以用 API 版本化（`/api/v1/`, `/api/v2/`）管理

---

## Part 13: Cloud Configuration Troubleshooting

---

### Q24: Azure 网络和 Application Gateway 排查

**问题：** 在 AKS 中，应用部署成功了但外部用户无法访问，你怎么排查？涉及 Azure Networking 和 Application Gateway。

**参考答案：**

**排查路径（从外到内）：**

**Step 1: DNS 解析**

```bash
nslookup webbroker.td.com
# 确认 DNS 指向正确的 Application Gateway 公网 IP
```

**Step 2: Application Gateway 健康状态**

```bash
# Azure CLI 查看后端健康状态
az network application-gateway show-backend-health \
  --name webbroker-appgw \
  --resource-group webbroker-rg
```

常见问题：后端健康探针失败。检查 health probe 的路径、端口、协议是否匹配。

**Step 3: NSG (Network Security Group) 规则**

```bash
az network nsg rule list --nsg-name webbroker-nsg --resource-group webbroker-rg
# 确认入站规则允许 80/443 端口
# 确认 AKS subnet 和 AppGW subnet 之间的通信被允许
```

**Step 4: AKS 内部检查**

```bash
# 检查 Service 是否有 Endpoints
kubectl get endpoints <service-name> -n <namespace>

# 检查 Ingress 配置
kubectl describe ingress <ingress-name> -n <namespace>

# 从 Pod 内部测试
kubectl exec -it <pod-name> -- curl localhost:8080/health
```

**Step 5: 常见根因**

- Application Gateway 的后端池 IP 过期（AKS node 重建后 IP 变了）
- SSL 证书过期或配置错误
- Ingress class 不匹配（AGIC vs nginx ingress controller）
- AKS 和 Application Gateway 在不同 VNet 且没有 peering

---

### Q25: EDP Pipeline (TD Standard) 相关

**问题：** 你对 TD EDP（Enterprise Delivery Platform）的 XL 和 GT pipeline 有什么了解？如果让你从零创建一条新 pipeline，你的步骤是什么？

**参考答案：**

**EDP 概述：**

- EDP XL: 面向较大型应用的标准化 pipeline 框架，集成了 TD 的安全扫描、合规检查、部署审批流程
- EDP GT: 更轻量的 pipeline 模板，适合较小的项目或快速迭代

**从零创建 Pipeline 的步骤：**

**Phase 1: 需求分析**

- 确定应用类型（Java/Node.js/Python）
- 确定部署目标（AKS/VM/Azure App Service）
- 确定需要哪些安全扫描（Veracode SAST、SCA、DAST）
- 确定审批流程（哪些环境需要人工审批）

**Phase 2: Pipeline 搭建**

- 在 GitHub Enterprise 中创建 `.github/workflows/` 目录
- 基于 EDP 标准模板创建 workflow 文件
- 配置 self-hosted runner labels 匹配 TD 的 runner pool
- 集成 Nexus 用于依赖拉取和制品推送
- 集成 Veracode 用于安全扫描

**Phase 3: 环境配置**

- 配置 GitHub Environments（dev, staging, prod）
- 设置 environment protection rules（prod 需要审批）
- 配置 secrets（Nexus credentials, Veracode API keys, kubeconfig）

**Phase 4: 验证和文档**

- 在 dev 环境完整运行一次 pipeline
- 修复所有问题后推广到 staging 和 prod
- 在 Confluence 上记录 pipeline 架构和配置说明
- 在 JIRA 上创建 ticket 跟踪后续改进

**Cloud Configuration Troubleshooting 经验：**

- Runner 网络问题：self-hosted runner 无法访问外部 registry → 检查 proxy 配置和 NSG 规则
- SSL 证书错误：自签名证书在 AKS runner 上不被信任 → 添加 CA cert 到 runner 的信任链
- pip/npm 路由问题：受限网络环境中包管理器无法直连 → 配置 Nexus proxy 作为中间层
