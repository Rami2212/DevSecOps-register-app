# CI/CD Pipeline: Jenkins + Maven + SonarQube + Docker + Trivy + ArgoCD + EKS (GitOps)

This project demonstrates a complete CI/CD pipeline using:

- **Jenkins Master + Jenkins Agent** (builds run on agent)
- **Maven** (build + test)
- **SonarQube** (static code analysis + quality gate)
- **Docker** (image build + push to DockerHub)
- **Trivy** (container image vulnerability scan)
- **AWS EKS** (Kubernetes cluster)
- **ArgoCD** (GitOps deployment from manifest repo)
- **Slack / Email** notification (after CD)

---

## Architecture / Flow

### CI Flow (Application Repo)
1. Developer pushes code to **GitHub Application Repository**
2. Jenkins **CI pipeline** triggers
3. Jenkins pulls source code
4. Build artifact using **Maven**
5. Run **Maven tests**
6. Run **SonarQube analysis**
7. Wait for **Quality Gate**
8. Build Docker image
9. Push Docker image to **DockerHub**
10. Run **Trivy scan** on Docker image
11. Cleanup workspace/artifacts
12. Trigger Jenkins **CD pipeline**

### CD Flow (GitOps Repo)
1. Jenkins **CD pipeline** triggers automatically
2. CD job updates **image tag (build number)** inside `deployment.yaml`
3. CD job commits & pushes updated manifest to **GitOps repository**
4. **ArgoCD** detects updated manifest from GitHub
5. ArgoCD deploys resources to **EKS**
6. Send notification via **Slack / Email**

---

## Prerequisites

- AWS Account
- GitHub account (2 repos)
- DockerHub account (with access token)
- Jenkins Master VM (Ubuntu)
- Jenkins Agent VM (Ubuntu, Docker installed)
- SonarQube VM (Ubuntu + PostgreSQL)
- EKS cluster (created using `eksctl`)
- ArgoCD installed on EKS
- (Optional) Slack webhook or Email SMTP configured in Jenkins

---

## Repositories

You will maintain **two repositories**:

### 1) Application Repository (CI)
Contains:
- Application source code
- `Jenkinsfile` (CI pipeline)

### 2) GitOps Repository (CD)
Contains:
- Kubernetes manifests (`deployment.yaml`, `service.yaml`, etc.)
- `Jenkinsfile` (CD pipeline)

---

## Infrastructure Setup (High Level)

### A) Jenkins Master (Ubuntu EC2)
- Open inbound port **8080**
- Install:
  - Java
  - Jenkins

### B) Jenkins Agent (Ubuntu EC2)
- Install:
  - Java
  - Docker
- Add docker permissions:
  - `sudo usermod -aG docker ubuntu`
- Configure SSH key auth between master → agent
- Add node in Jenkins via **SSH** using private key

### C) SonarQube Server (Ubuntu EC2)
- Open inbound port **9000**
- Install:
  - PostgreSQL
  - Java (Adoptium)
  - SonarQube
- Configure:
  - DB credentials in `sonar.properties`
  - SonarQube systemd service

### D) EKS Bootstrap Server (Ubuntu EC2)
Install:
- AWS CLI
- kubectl
- eksctl

Create EKS cluster using:
- `eksctl create cluster ...`

### E) ArgoCD on EKS
- Create namespace `argocd`
- Install ArgoCD manifests
- Expose ArgoCD service using LoadBalancer
- Login and update admin password
- Add EKS cluster into ArgoCD
- Connect GitOps repo to ArgoCD

---

## Jenkins Plugins Required

### CI Side
- Maven Integration
- Pipeline Maven Integration
- Eclipse Temurin Installer
- SonarQube Scanner
- Sonar Quality Gates / Quality Gates
- Docker
- Docker Commons
- Docker Pipeline
- Docker API
- Docker Build Step
- CloudBees Docker Build and Publish

---

## Jenkins Global Tool Configuration

Go to: **Manage Jenkins → Tools**

- Maven:
  - Name: `Maven3`
  - Install automatically ✅
- JDK:
  - Name: `Java17`
  - Install automatically ✅
  - Install from Adoptium (example: 17.0.5+8)
- Sonar Scanner:
  - Name: `SonarQubeScanner`
  - Install automatically ✅

---

## Credentials Required in Jenkins

Go to: **Manage Jenkins → Credentials**

### GitHub (for CI + CD repos)
- Kind: Username with password
- ID: `github`
- Username: `<your_github_username>`
- Password: `<github_personal_access_token>`

### DockerHub
- Kind: Username with password
- ID: `dockerhub`
- Username: `<your_dockerhub_username>`
- Password: `<dockerhub_access_token>`

### SonarQube Token
- Kind: Secret text
- ID: `jenkins-sonarqube-token`
- Secret: `<sonarqube_generated_token>`

### Jenkins API Token (to trigger CD pipeline from CI)
- Kind: Secret text
- ID: `jenkins-api-token`
- Secret: `<jenkins_user_api_token>`

---

## SonarQube Integration

### 1) Configure SonarQube in Jenkins
Go to:
**Manage Jenkins → System → SonarQube servers**

- Name: `SonarQubeServer`
- URL: `http://<sonarqube_private_ip>:9000`
- Authentication token: `jenkins-sonarqube-token`

### 2) Create SonarQube Webhook
In SonarQube:
**Administration → Configuration → Webhooks → Create**

Webhook URL example:
- `http://<jenkins_master_ip>:8080/sonarqube-webhook/`

> Without this webhook, the quality gate stage may fail.

---

## CI Jenkinsfile (Application Repo)

Your CI Jenkinsfile typically includes stages:
- Cleanup workspace
- Checkout SCM
- Build (mvn clean package)
- Test (mvn test)
- SonarQube Analysis
- Quality Gate
- Build & Push Docker Image
- Trivy Scan
- Cleanup artifacts
- Trigger CD pipeline (remote trigger)

> Update names like agent label, tool names, credential IDs to match your Jenkins.

---

## CD Jenkinsfile (GitOps Repo)

Your CD Jenkinsfile typically includes stages:
- Cleanup workspace
- Checkout SCM
- Update image tag in `deployment.yaml`
- Commit and push changes back to GitOps repo

The CD job is configured to accept a parameter:
- `IMAGE_TAG` (string)

---

## ArgoCD App Configuration

In ArgoCD:
- Create new app
- Project: `default`
- Sync Policy: **Automatic**
- Enable:
  - Prune resources ✅
  - Self Heal ✅
- Source:
  - Repo URL: GitOps repo
  - Path: `./`
- Destination:
  - Cluster: Your EKS cluster
  - Namespace: `default`

ArgoCD will continuously apply the latest `deployment.yaml` from GitHub.

---

## Triggering CI Automatically

In Jenkins CI job:
**Build Triggers**
- Poll SCM: `* * * * *` (every minute)

So any push to GitHub triggers CI automatically.

---

## Verification Steps (End-to-End)

1. Modify a file in application repo (ex: `index.jsp`)
2. Push to GitHub
3. Jenkins CI triggers automatically
4. Docker image is pushed with new build tag
5. CI triggers CD pipeline
6. CD updates GitOps `deployment.yaml` image tag
7. ArgoCD syncs and deploys to EKS
8. Confirm:
   - `kubectl get pods`
   - `kubectl get svc`
   - App is accessible via LoadBalancer DNS
