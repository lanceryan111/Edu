可以，下面这套就是 **最像 DevOps 面试现场会让你“手写/白板写”的 10 个题**。
我按 **题目 + 可直接写出来的标准答案 + 面试官看点** 来整理。
你不需要一字不差背，但最好熟到能现场默写 70% 以上。

---

# 1. 写一个最基础的 GitHub Actions CI pipeline

### 题目

> Write a GitHub Actions workflow that checks out code, installs dependencies, runs tests, and builds the project.

### 标准答案

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  build-test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test

      - name: Build project
        run: npm run build
```

### 面试官看点

* 有 `checkout`
* 有 dependency install
* 有 test 和 build
* 结构清晰

---

# 2. 写一个带 Docker build 的 GitHub Actions pipeline

### 题目

> Write a pipeline that builds a Docker image and tags it with the Git commit SHA.

### 标准答案

```yaml
name: Docker Build

on:
  push:
    branches: [main]

jobs:
  docker-build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Build Docker image
        run: docker build -t my-app:${{ github.sha }} .
```

### 加分版

如果想更完整一点：

```yaml
name: Docker Build

on:
  push:
    branches: [main]

jobs:
  docker-build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Build Docker image
        run: |
          docker build -t my-app:latest -t my-app:${{ github.sha }} .
```

### 面试官看点

* 知道用 commit SHA 做版本标识
* 知道 image tagging

---

# 3. 写一个 Kubernetes Deployment YAML

### 题目

> Write a Kubernetes Deployment for an application with 3 replicas.

### 标准答案

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: my-app:1.0.0
          ports:
            - containerPort: 8080
```

### 面试官看点

* `selector` 和 `labels` 一致
* `replicas`
* container image / port

---

# 4. 写一个更像生产环境的 Kubernetes Deployment

### 题目

> Add health checks and rolling update strategy to the deployment.

### 标准答案

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: my-app:1.0.0
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 20
```

### 面试官看点

* `readinessProbe` 和 `livenessProbe`
* rolling update
* 有 downtime awareness

---

# 5. 写一个 Kubernetes Service

### 题目

> Expose the application internally in the cluster.

### 标准答案

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
```

### 面试官看点

* 知道内部服务一般用 `ClusterIP`
* `selector` 对应 pod label
* `port` / `targetPort` 概念正确

---

# 6. 写一个最基础的 Ansible playbook：安装 nginx

### 题目

> Write an Ansible playbook to install nginx on web servers.

### 标准答案

```yaml
- name: Install nginx
  hosts: web
  become: yes

  tasks:
    - name: Install nginx package
      apt:
        name: nginx
        state: present
        update_cache: yes
```

### 面试官看点

* `hosts`
* `become: yes`
* 用 module，不是瞎写 shell
* `state: present` 体现 idempotency

---

# 7. 写一个 Ansible playbook：启动并启用服务

### 题目

> Ensure nginx is started and enabled after installation.

### 标准答案

```yaml
- name: Install and start nginx
  hosts: web
  become: yes

  tasks:
    - name: Install nginx package
      apt:
        name: nginx
        state: present
        update_cache: yes

    - name: Ensure nginx service is running
      service:
        name: nginx
        state: started
        enabled: yes
```

### 面试官看点

* 会把 package install 和 service management 分开
* `enabled: yes`
* 自动化完整性

---

# 8. 写一个 Ansible playbook：拷贝配置文件并重启服务

### 题目

> Deploy an nginx config file and restart nginx only if the config changes.

### 标准答案

```yaml
- name: Configure nginx
  hosts: web
  become: yes

  tasks:
    - name: Copy nginx config
      copy:
        src: nginx.conf
        dest: /etc/nginx/nginx.conf
        owner: root
        group: root
        mode: '0644'
      notify: Restart nginx

  handlers:
    - name: Restart nginx
      service:
        name: nginx
        state: restarted
```

### 面试官看点

* 知道 `notify` / `handlers`
* 只有配置变更才 restart
* 这是很典型的 Ansible best practice

---

# 9. 写一个简单的 Azure DevOps multi-stage pipeline

### 题目

> Write an Azure DevOps pipeline with build and deploy stages.

### 标准答案

```yaml
trigger:
  - main

stages:
  - stage: Build
    jobs:
      - job: BuildJob
        pool:
          vmImage: ubuntu-latest
        steps:
          - script: echo "Building application"
            displayName: Build Step

  - stage: Deploy
    dependsOn: Build
    jobs:
      - job: DeployJob
        pool:
          vmImage: ubuntu-latest
        steps:
          - script: echo "Deploying application"
            displayName: Deploy Step
```

### 面试官看点

* 知道 stage 概念
* `dependsOn`
* build / deploy 分离

---

# 10. 写一个更完整的 pipeline flow：build、scan、deploy

### 题目

> Write a pipeline flow that includes build, security scan, and deployment.

### 标准答案

这个可以用伪代码/YAML 混合写，面试里很常见。

```yaml
name: Build-Scan-Deploy

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build app
        run: npm ci && npm run build

  security-scan:
    runs-on: ubuntu-latest
    needs: build
    steps:
      - uses: actions/checkout@v4
      - name: Run security scan
        run: echo "Run SonarQube or Veracode scan here"

  deploy:
    runs-on: ubuntu-latest
    needs: security-scan
    steps:
      - uses: actions/checkout@v4
      - name: Deploy application
        run: echo "Deploy to AKS with Helm"
```

### 面试官看点

* 知道阶段顺序
* `needs`
* security gate 概念
* deployment 在 scan 后面

---

# 现场手写时最容易被追问的 5 个点

## 1. 为什么用 `npm ci` 而不是 `npm install`

答：

* 更适合 CI
* 更可重复
* 基于 lock file，结果更稳定

## 2. readinessProbe 和 livenessProbe 区别

答：

* readiness：是否可以接流量
* liveness：是否需要重启容器

## 3. 为什么 Ansible 推荐 module 而不是 shell

答：

* idempotent
* 更安全
* 更可维护
* 返回结果结构化

## 4. 为什么 build 和 deploy 要分 stage

答：

* 职责清晰
* 便于审批
* 便于失败定位
* 支持 artifact promotion

## 5. 为什么 security scan 要放进 pipeline

答：

* shift left
* 尽早发现问题
* 阻止高风险代码进入生产

---

# 你可以重点背的 4 个“万能骨架”

## GitHub Actions 骨架

```yaml
name: CI
on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm test
      - run: npm run build
```

## Kubernetes Deployment 骨架

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: app
  template:
    metadata:
      labels:
        app: app
    spec:
      containers:
        - name: app
          image: app:1.0
```

## Ansible Playbook 骨架

```yaml
- hosts: web
  become: yes
  tasks:
    - name: Install package
      apt:
        name: nginx
        state: present
```

## Pipeline flow 骨架

```text
checkout → build → test → scan → publish artifact → deploy → verify
```

---

# 最后一个面试技巧

如果现场让你写，不要一上来埋头写。
先说一句：

> I’ll start with a simple version first, then I can extend it with security and reliability controls.

这句话很像 senior / lead，会显得你有层次。

你要的话，我下一条可以直接帮你出一版
**“10个现场手写题的口头讲解版答案（每题 30-60 秒英文回答）”**，这样你不仅会写，还会讲。


太好了，这一部分是 **面试表现拉开差距的关键**🔥
你现在不是只“会写”，而是要做到 **“写 + 讲 + 体现 senior thinking”**

我给你整理了 **10个题的 30–60秒英文口头答案模板**，你可以直接练习说。

---

# 🎤 1. GitHub Actions CI Pipeline

### 🗣️ 标准回答

> I would design a simple CI pipeline triggered on push or pull request.
> First, I would check out the code using the checkout action.
> Then I would install dependencies using npm ci to ensure consistency.
> After that, I would run unit tests to validate the code.
> Finally, I would build the application.
> In a production setup, I would also include caching and possibly artifact publishing to improve performance and traceability.

---

# 🎤 2. Docker Build Pipeline

### 🗣️

> In this pipeline, I would build a Docker image using the Docker CLI.
> I would tag the image with both “latest” and the commit SHA to ensure version traceability.
> This allows us to uniquely identify each build.
> In a real scenario, I would also push the image to a registry like Azure Container Registry and include vulnerability scanning before deployment.

---

# 🎤 3. Kubernetes Deployment（基础）

### 🗣️

> This deployment defines three replicas of the application to ensure availability.
> The selector and labels are aligned to allow Kubernetes to manage the pods correctly.
> Each pod runs a container with a defined image and exposed port.
> This setup provides basic scalability and ensures the application can handle some level of traffic.

---

# 🎤 4. Kubernetes Deployment（生产级）

### 🗣️

> In this version, I added a rolling update strategy to avoid downtime during deployments.
> I also included readiness and liveness probes to ensure the application is healthy before receiving traffic and to automatically restart failed containers.
> This improves reliability and makes the deployment production-ready.

---

# 🎤 5. Kubernetes Service

### 🗣️

> This service exposes the application internally within the cluster using ClusterIP.
> It routes traffic to pods based on matching labels.
> The service abstracts the pod layer and provides stable networking, which is important since pods can be ephemeral.

---

# 🎤 6. Ansible Playbook（安装 nginx）

### 🗣️

> This playbook installs nginx on a group of hosts defined in the inventory.
> I use the apt module instead of shell commands to ensure idempotency.
> The “state: present” ensures that the package is installed only if needed, making the automation safe to run multiple times.

---

# 🎤 7. Ansible + Service 管理

### 🗣️

> In addition to installing nginx, this playbook ensures the service is started and enabled.
> This guarantees that nginx will run after system reboot.
> Separating installation and service management improves clarity and maintainability of the automation.

---

# 🎤 8. Ansible Handler（配置更新）

### 🗣️

> In this example, I use a handler to restart nginx only when the configuration file changes.
> This avoids unnecessary restarts and improves efficiency.
> Handlers are a key feature in Ansible for event-driven automation and help maintain system stability.

---

# 🎤 9. Azure DevOps Pipeline（multi-stage）

### 🗣️

> This pipeline is structured into multiple stages, separating build and deployment.
> The deploy stage depends on the build stage, ensuring that only successful builds are deployed.
> This structure improves pipeline clarity and allows for adding approval gates before production deployment.

---

# 🎤 10. Build → Scan → Deploy Pipeline（重点题🔥）

### 🗣️

> This pipeline follows a secure DevOps workflow.
> First, the application is built and tested.
> Then, a security scan is performed using tools like SonarQube or Veracode to identify vulnerabilities early.
> Only if the scan passes, the deployment stage is triggered.
> This approach enforces security as part of the pipeline and reduces risk in production.

---

# ⭐ 面试加分技巧（你一定要用）

每道题你都可以 **主动加一句**👇

### 🔥 万能加分句

> “In a production environment, I would also consider adding monitoring, rollback strategies, and security controls.”

---

# ⭐ 更高级一点的 closing（lead级）

> “The goal is to make the pipeline reliable, repeatable, and secure.”

---

# 🎯 最后建议（很重要）

你现在可以这样准备：

### 第一步（强烈推荐）

👉 每题 **录音自己讲一遍**

### 第二步

👉 控制在 **30–45秒**

### 第三步

👉 练习结构：

```text
what it does
why it matters
production improvement
```

---

# 如果你想再冲一档（真的能拉开差距）

我可以帮你做👇

👉 **“5个 Whiteboard题的完整英文讲解（2分钟版本）”**

那个是 **final round killer level**，
讲得好基本就是 **offer signal** 🚀

