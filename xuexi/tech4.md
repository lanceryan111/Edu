# 银行 DevOps 面试特别喜欢问的 3 个问题 — 参考答案框架

**TD Bank - WebBroker Platform DevOps Engineer**

-----

## ⭐1: How do you design a reliable CI/CD system for production?

**开场（30秒）：** 我会从五个核心原则来回答这个问题。

-----

### 1. Pipeline 架构分层

```
PR Pipeline (快) → Merge Pipeline (全) → Deploy Pipeline (稳)
```

- **PR 阶段：** build + unit test + Veracode Pipeline Scan（快速反馈，5-10分钟内）
- **Merge 阶段：** full build + integration test + Veracode Full Scan + Docker build + push to Nexus/ACR
- **Deploy 阶段：** Ansible playbook → Helm upgrade → smoke test → 逐环境推进

### 2. 可靠性保障

- **Self-hosted runners 高可用：** 在 AKS 上跑多副本 runner，避免单点故障。我在 TD 实际做过 runner watchdog 系统，用 Ansible 监控 macOS runner 状态，发现 stalled job 自动重启
- **Retry 机制：** GitHub API 调用加 exponential backoff retry，网络抖动不会导致整个 pipeline 失败
- **Artifact 缓存：** Maven/npm 依赖通过 Nexus proxy 缓存，不依赖外网稳定性
- **Pipeline as Code：** 所有 workflow 文件在 Git 中版本控制，变更走 PR review

### 3. 安全集成（Shift Left）

- PR 阶段就跑安全扫描，不是等到部署前才发现问题
- Veracode SAST + SCA 集成在 pipeline 中，Critical/High 自动 block
- Secrets 分层管理：org / repo / environment secrets，prod 环境加 protection rules

### 4. 环境隔离与推进策略

```
dev (自动) → staging (自动 + full test) → prod (审批 + canary)
```

- GitHub Environments + protection rules 控制推进节奏
- Prod 部署需要 Tech Lead 审批，符合 TD 变更管理流程
- 蓝绿/Canary 部署降低风险

### 5. 可观测性

- Pipeline 运行状态发 Slack 通知
- 部署后自动跑 smoke test 验证
- 失败自动创建 Jira ticket 跟踪

### 结合 TD 实际经验举例

> “在 TD mobile banking 项目中，我从零搭建了 GitHub Actions pipeline 替换原来的 Jenkins。pipeline 包含 Gradle build、Veracode 扫描、Docker 构建、通过 Ansible 部署到 AKS。我还实现了自动 PR merge 的 workflow，配合 branch protection bypass 逻辑处理紧急修复场景。整个迁移后 build 时间从 45 分钟降到 20 分钟。”

-----

## ⭐2: How do you handle failed deployments in production?

**开场：** 我会按照时间线来回答 — 部署前的预防、部署中的检测、失败后的响应。

-----

### 1. 部署前 — 预防措施

- **Staging 完整验证：** prod 部署前必须在 staging 通过所有 integration test
- **部署策略选择：**
  - Rolling update（maxUnavailable: 0）保证始终有 Pod 服务
  - 关键服务用蓝绿部署，一键切换流量
  - Canary 先放 10% 流量观察
- **Rollback plan 提前准备：** 每次部署前确认上一个稳定版本号，确保能快速回滚

### 2. 部署中 — 自动检测

```bash
# 自动化健康检查脚本（集成在 pipeline 中）
kubectl rollout status deployment/$APP -n prod --timeout=300s
if [ $? -ne 0 ]; then
  echo "Rollout failed, initiating rollback..."
  kubectl rollout undo deployment/$APP -n prod
fi
```

- **Readiness Probe 是第一道防线：** 新 Pod 如果 health check 不过，不会接收流量
- **Smoke test 是第二道防线：** 部署完自动跑关键业务流程验证
- **Metrics 监控是第三道防线：** 错误率、延迟、5xx 数量超阈值自动告警

### 3. 失败后 — 立即响应

**自动回滚：**

```bash
# Helm rollback
helm rollback payment-service 0 -n production
# 或 kubectl
kubectl rollout undo deployment/payment-service -n production
```

**手动介入流程（如果自动回滚不够）：**

1. **止血：** 立即回滚到上一个已知稳定版本
1. **通知：** Slack 告警通知团队 + PagerDuty on-call
1. **诊断：** 查日志（`kubectl logs --previous`），查 events（`kubectl describe pod`）
1. **根因分析：** 是代码问题、配置问题、还是基础设施问题？
1. **修复验证：** 在 staging 复现并修复后，重新走完整 pipeline
1. **Post-mortem：** 写 incident report，更新 runbook，改进 pipeline 防止类似问题

### 4. 预防复发

- 在 pipeline 中加入失败场景的自动化测试
- 更新 Confluence runbook
- 如果是配置问题，加入 config validation step
- 如果是资源问题，调整 HPA/VPA 策略

### 结合 TD 实际经验举例

> “有一次我们在部署 Android build pipeline 时，Gradle daemon 在 unit test 后 hang 住导致整个 runner 卡死。我通过 Ansible playbook 实现了自动检测 stalled 进程并 kill + restart runner 的 watchdog 机制。之后又在 pipeline 中加了 timeout 和 post-failure cleanup step，从根本上预防了这类问题。”

-----

## ⭐3: How do you ensure infrastructure changes are auditable and secure?

**开场：** 在银行环境中，审计和安全不是可选项，是强制要求。我从四个层面来保证。

-----

### 1. Infrastructure as Code — 一切变更可追溯

- **所有基础设施通过 Terraform / Ansible 管理，** 禁止手动 Portal/CLI 操作
- **Git 作为 single source of truth：** 每次变更都有 commit 记录 — who, when, what, why
- **PR review 强制：** 所有 IaC 变更必须经过至少 2 人 review 才能 merge
- **terraform plan 输出附在 PR comment 中，** reviewer 可以清楚看到将要发生什么变更

```yaml
# GitHub Actions: 自动在PR中展示 terraform plan
- name: Terraform Plan
  run: terraform plan -no-color -out=tfplan
- name: Post plan to PR
  uses: actions/github-script@v7
  with:
    script: |
      github.rest.issues.createComment({
        issue_number: context.issue.number,
        body: '```\n' + planOutput + '\n```'
      })
```

### 2. 访问控制 — 最小权限原则

- **RBAC 分层：**
  - Dev: 只能 view staging/prod，不能 apply
  - DevOps Lead: 可以 apply staging，prod 需要审批
  - 紧急情况: break-glass 流程，事后必须审计
- **GitHub Environment protection rules：** prod 部署需要指定人员审批
- **Ansible Vault / Azure Key Vault：** 敏感信息加密存储，运行时动态获取
- **Self-hosted runner 隔离：** 不同环境用不同 runner pool，prod runner 在独立网络区域

### 3. 合规扫描 — 自动化检查

- **Veracode SAST + SCA：** 每次 merge 自动扫描代码和依赖漏洞
- **Container scanning：** Docker image 构建后扫描 CVE
- **Terraform compliance：** 用 `tfsec` 或 `checkov` 扫描 IaC 中的安全配置问题

```bash
# 在 pipeline 中自动扫描
- name: Scan Terraform
  run: |
    checkov -d ./terraform --quiet --compact
    # 检查：是否有开放的 NSG 规则、未加密的存储、缺少 tag 的资源等
```

- **Compliance reports 自动发布：** 扫描结果存档到 Confluence，满足审计要求

### 4. Drift Detection — 持续监控

- **定期 terraform plan：** 每天 cron job 跑一次，检测是否有人绕过 IaC 手动修改
- **Azure Policy：** 在云平台层面强制执行安全策略（如所有存储必须加密、所有资源必须有 tag）
- **Git audit log：** GitHub Enterprise 的 audit log 记录所有操作

### 5. 变更管理流程（TD 特定）

- 所有 prod 变更必须有 Change Ticket
- Pipeline 自动创建 Change Request，附带变更内容和 rollback plan
- 变更窗口外的紧急变更走 Emergency Change 流程，事后补 post-mortem

### 结合 TD 实际经验举例

> “在 TD 我推动了所有 CI/CD pipeline 配置从手动管理迁移到 Git 版本控制。每条 pipeline 的变更都走 PR review，Veracode 扫描结果自动附在 PR 中。我还建立了 Ansible playbook 来统一管理多台 runner 的配置，消除了手动配置导致的环境不一致和审计盲区。对于 vault 中的 secret rotation，我设计了自动化流程确保密钥定期轮换并记录审计日志。”

-----

## 面试 Tips

这三个问题的回答策略是 **STAR + 框架**：

1. 先给出清晰的框架/层次（让面试官知道你有体系化思考）
1. 每个层次给具体工具和配置（证明你有实操经验）
1. 最后用 TD 的实际案例收尾（展示你不是纸上谈兵）

回答时间控制在每题 3-5 分钟，留时间给面试官追问。