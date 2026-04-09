# CI/CD and Deployment Tools Guide
## ArgoCD, Jenkins, GitHub Actions, Azure DevOps

## Table of Contents
1. [GitOps Fundamentals](#gitops-fundamentals)
2. [ArgoCD - GitOps for Kubernetes](#argocd---gitops-for-kubernetes)
3. [Jenkins - Traditional CI/CD](#jenkins---traditional-cicd)
4. [GitHub Actions - Native GitHub CI/CD](#github-actions---native-github-cicd)
5. [Azure DevOps - Complete DevOps Platform](#azure-devops---complete-devops-platform)
6. [Integration Patterns](#integration-patterns)
7. [Comparison & Best Practices](#comparison--best-practices)
8. [Interview Questions](#interview-questions)

---

## Part 1: GitOps Fundamentals

### What is GitOps?

**GitOps Principles:**
```
Git as Single Source of Truth
        ↓
Declarative Configuration
        ↓
Automated Continuous Deployment
        ↓
Version Control Everything
        ↓
Self-Healing Systems
```

**GitOps Workflow:**
```
Developer pushes code
        ↓
Git webhook triggers
        ↓
Build & Test
        ↓
Push to Git (manifests)
        ↓
GitOps Controller watches Git
        ↓
Detect drift (desired vs actual)
        ↓
Reconcile to desired state
        ↓
Kubernetes deployment
```

**Benefits:**
- Single source of truth in Git
- Auditable deployments (full history)
- Easy rollbacks (git revert)
- Automated drift correction
- Declarative infrastructure
- Built-in disaster recovery

---

## Part 2: ArgoCD - GitOps for Kubernetes

### 2.1 What is ArgoCD?

**Definition:** Declarative GitOps continuous deployment tool for Kubernetes.

**Architecture:**
```
┌─────────────────────────────────────┐
│      GitHub/GitLab Repository       │
│  (Deployment manifests + config)    │
└────────────────┬────────────────────┘
                 │
                 ▼ (Webhook on commit)
        ┌────────────────────┐
        │   Git Webhook      │
        └────────┬───────────┘
                 │
                 ▼
    ┌─────────────────────────────┐
    │   ArgoCD Controller          │
    │  (Runs in K8s cluster)       │
    ├─────────────────────────────┤
    │ • Git Monitor               │
    │ • Manifest Compiler         │
    │ • Sync Engine               │
    │ • Health Assessment         │
    └────────────┬────────────────┘
                 │
                 ▼
         ┌───────────────┐
         │ Kubernetes    │
         │ Cluster       │
         │ (Deployments) │
         └───────────────┘
```

### 2.2 Installation

**Install ArgoCD:**
```bash
# Create namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Verify installation
kubectl get pods -n argocd

# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Port forward to access UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Access: https://localhost:8080
```

**Helm Installation:**
```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  --values argocd-values.yaml
```

**ArgoCD Helm Values:**
```yaml
global:
  domain: argocd.example.com

server:
  service:
    type: LoadBalancer
  ingress:
    enabled: true
    ingressClassName: nginx
    hosts:
    - argocd.example.com
    tls:
    - secretName: argocd-tls
      hosts:
      - argocd.example.com

redis:
  persistence:
    enabled: true
    size: 10Gi

repoServer:
  replicas: 3
  autoscaling:
    enabled: true
    minReplicas: 3
    maxReplicas: 10

configs:
  secret:
    argocdServerAdminPassword: "$2a$10$..." # bcrypt hash
  
  cm:
    url: https://argocd.example.com
    
    # RBAC configuration
    policy.default: role:readonly
    policy.csv: |
      p, role:admin, applications, *, */*, allow
      p, role:admin, repositories, *, *, allow
      p, role:readonly, applications, get, */*, allow
      g, developers, role:readonly

notifications:
  enabled: true
```

### 2.3 ArgoCD Application Example

**Simple Application:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  
  # Git repository source
  source:
    repoURL: https://github.com/myorg/myapp-config
    targetRevision: main
    path: kubernetes/
    
    # Kustomize support
    kustomize:
      version: v4.5.4
  
  # Destination cluster
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  
  # Sync policy
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
  
  # Health assessment
  ignoreDifferences:
  - group: apps
    kind: Deployment
    jsonPointers:
    - /spec/replicas
```

**Helm-based Application:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx
  namespace: argocd
spec:
  project: default
  
  source:
    repoURL: https://kubernetes.github.io/ingress-nginx
    chart: ingress-nginx
    targetRevision: 4.5.0
    
    helm:
      releaseName: nginx
      values: |
        controller:
          replicas: 3
          resources:
            limits:
              cpu: 200m
              memory: 256Mi
      
      valuesObject:
        controller:
          metrics:
            enabled: true
  
  destination:
    server: https://kubernetes.default.svc
    namespace: ingress-nginx
  
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

**Multi-Environment Application:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapp-multi-env
  namespace: argocd
spec:
  # Generator for multiple environments
  generators:
  - list:
      elements:
      - cluster: dev
        namespace: development
        replicas: 1
      - cluster: staging
        namespace: staging
        replicas: 2
      - cluster: prod
        namespace: production
        replicas: 3
  
  template:
    metadata:
      name: 'myapp-{{cluster}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/myorg/myapp-config
        targetRevision: main
        path: kubernetes/
        
        kustomize:
          patches:
          - target:
              kind: Deployment
            patch: |-
              - op: replace
                path: /spec/replicas
                value: {{replicas}}
      
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{namespace}}'
      
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

### 2.4 ArgoCD CLI Operations

**Login to ArgoCD:**
```bash
argocd login argocd.example.com --username admin --password <password>
```

**Create Application:**
```bash
argocd app create myapp \
  --repo https://github.com/myorg/config \
  --path kubernetes/ \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace production \
  --auto-prune \
  --self-heal
```

**Sync Application:**
```bash
# Manual sync
argocd app sync myapp

# Sync with force
argocd app sync myapp --force

# Sync specific resource
argocd app sync myapp --resource apps:Deployment:myapp-deployment
```

**Monitor Application:**
```bash
# Get app status
argocd app get myapp

# Watch app status
argocd app wait myapp

# Check application health
argocd app get myapp --refresh
```

**Rollback:**
```bash
# Get history
argocd app history myapp

# Rollback to previous version
argocd app rollback myapp <REVISION_ID>

# Rollback to specific revision
argocd app rollback myapp 0
```

### 2.5 ArgoCD Notifications

**Slack Integration:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  service.slack: |
    token: $slack-token

  template.app-deployed: |
    message: |
      Application {{.app.metadata.name}} deployment status is {{.app.status.operationState.phase}}.
    slack:
      attachments: |
        [{
          "color": "#18be52",
          "fields": [
            {
              "title": "Sync Status",
              "value": "{{.app.status.syncStatus.status}}",
              "short": true
            },
            {
              "title": "Repository",
              "value": "{{.app.spec.source.repoURL}}",
              "short": true
            }
          ]
        }]

  trigger.on-deployed: |
    - when: app.status.operationState.phase in ['Succeeded'] and app.status.health.status == 'Healthy'
      oncePer: app.status.operationState.finishedAt
      send: [app-deployed]
```

---

## Part 3: Jenkins - Traditional CI/CD

### 3.1 What is Jenkins?

**Definition:** Open-source automation server for building, testing, and deploying software.

**Architecture:**
```
┌──────────────────────────────────────┐
│  Jenkins Master                      │
│  (Orchestration & UI)                │
├──────────────────────────────────────┤
│ • Job Scheduling                     │
│ • Pipeline Definition                │
│ • Build Coordination                 │
└─────────────┬──────────────────────┬─┘
              │                      │
         ┌────▼────┐          ┌──────▼──────┐
         │ Agent 1  │          │ Agent 2     │
         │ (Docker) │          │ (K8s Pod)   │
         └──────────┘          └─────────────┘
              │                      │
         ┌────▼──────────────────────▼────┐
         │  Execute Build Jobs             │
         │  • Compile                      │
         │  • Test                         │
         │  • Build Docker Image           │
         │  • Push Registry                │
         └─────────────────────────────────┘
```

### 3.2 Jenkins Installation

**Kubernetes Deployment:**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: jenkins

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: jenkins

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jenkins
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: jenkins
  namespace: jenkins

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: jenkins
  namespace: jenkins
spec:
  serviceName: jenkins
  replicas: 1
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      serviceAccountName: jenkins
      containers:
      - name: jenkins
        image: jenkins/jenkins:lts
        ports:
        - containerPort: 8080
          name: http
        - containerPort: 50000
          name: agent
        env:
        - name: JENKINS_OPTS
          value: "--prefix=/jenkins"
        volumeMounts:
        - name: jenkins-home
          mountPath: /var/jenkins_home
        - name: docker-sock
          mountPath: /var/run/docker.sock
      volumes:
      - name: docker-sock
        hostPath:
          path: /var/run/docker.sock
  volumeClaimTemplates:
  - metadata:
      name: jenkins-home
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: gp2
      resources:
        requests:
          storage: 50Gi

---
apiVersion: v1
kind: Service
metadata:
  name: jenkins
  namespace: jenkins
spec:
  selector:
    app: jenkins
  ports:
  - name: http
    port: 8080
    targetPort: 8080
  - name: agent
    port: 50000
    targetPort: 50000
  type: LoadBalancer
```

**Helm Installation:**
```bash
helm repo add jenkinsci https://charts.jenkins.io
helm repo update

helm install jenkins jenkinsci/jenkins \
  --namespace jenkins \
  --create-namespace \
  --values jenkins-values.yaml
```

**Jenkins Helm Values:**
```yaml
controller:
  image: jenkins/jenkins
  tag: "2.387.1"
  
  installPlugins:
  - kubernetes:latest
  - pipeline-model-definition:latest
  - git:latest
  - docker:latest
  - slack:latest
  - parameterized-trigger:latest
  - job-dsl:latest
  
  JCasC:
    configScripts:
      basic: |
        jenkins:
          securityRealm:
            saml:
              binding: "urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST"
          crumbIssuer:
            standard:
              excludeClientIPFromCrumb: true

  persistence:
    enabled: true
    storageClass: gp2
    size: 50Gi

agent:
  enabled: true
  replicas: 2
  image: jenkins/inbound-agent
  tag: latest
```

### 3.3 Jenkins Pipeline

**Declarative Pipeline:**
```groovy
pipeline {
  agent {
    kubernetes {
      yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: docker
    image: docker:latest
    command:
    - cat
    tty: true
    volumeMounts:
    - name: docker-sock
      mountPath: /var/run/docker.sock
  - name: kubectl
    image: bitnami/kubectl:latest
    command:
    - cat
    tty: true
  volumes:
  - name: docker-sock
    hostPath:
      path: /var/run/docker.sock
'''
    }
  }
  
  environment {
    REGISTRY = 'myregistry.azurecr.io'
    IMAGE_NAME = 'myapp'
    IMAGE_TAG = "${BUILD_NUMBER}"
    DOCKER_CREDENTIALS = credentials('docker-credentials')
  }
  
  stages {
    stage('Checkout') {
      steps {
        git branch: 'main',
            url: 'https://github.com/myorg/myapp.git'
      }
    }
    
    stage('Build') {
      steps {
        sh '''
          echo "Building application..."
          ./gradlew build
        '''
      }
    }
    
    stage('Test') {
      steps {
        sh '''
          echo "Running tests..."
          ./gradlew test
        '''
      }
    }
    
    stage('Build Docker Image') {
      steps {
        container('docker') {
          sh '''
            docker build -t ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} .
            docker tag ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} ${REGISTRY}/${IMAGE_NAME}:latest
          '''
        }
      }
    }
    
    stage('Push to Registry') {
      steps {
        container('docker') {
          sh '''
            echo $DOCKER_CREDENTIALS_PSW | docker login -u $DOCKER_CREDENTIALS_USR --password-stdin ${REGISTRY}
            docker push ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
            docker push ${REGISTRY}/${IMAGE_NAME}:latest
          '''
        }
      }
    }
    
    stage('Deploy to K8s') {
      steps {
        container('kubectl') {
          sh '''
            kubectl set image deployment/myapp myapp=${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} \
              -n production \
              --record
            
            kubectl rollout status deployment/myapp -n production
          '''
        }
      }
    }
    
    stage('Health Check') {
      steps {
        sh '''
          sleep 30
          kubectl get pods -n production -l app=myapp
        '''
      }
    }
  }
  
  post {
    success {
      echo "Build successful!"
      // Send Slack notification
      slackSend(channel: '#deployments',
                message: "Build #${BUILD_NUMBER} deployed successfully",
                color: 'good')
    }
    
    failure {
      echo "Build failed!"
      slackSend(channel: '#deployments',
                message: "Build #${BUILD_NUMBER} failed",
                color: 'danger')
    }
  }
}
```

**Scripted Pipeline:**
```groovy
node('kubernetes') {
  try {
    stage('Checkout') {
      checkout scm
    }
    
    stage('Build') {
      sh 'make build'
    }
    
    stage('Test') {
      sh 'make test'
      junit 'build/test-results/**/*.xml'
    }
    
    stage('Code Quality') {
      sh 'make lint'
      publishHTML([
        reportDir: 'build/reports',
        reportFiles: 'index.html',
        reportName: 'Code Quality Report'
      ])
    }
    
    stage('Build Image') {
      def image = docker.build("myapp:${BUILD_NUMBER}")
    }
    
    stage('Deploy') {
      if (env.BRANCH_NAME == 'main') {
        sh 'kubectl apply -f k8s/production/'
        sh 'kubectl rollout status deployment/myapp'
      }
    }
    
  } catch (Exception e) {
    echo "Pipeline failed: ${e}"
    throw e
  } finally {
    cleanWs()
  }
}
```

### 3.4 Jenkins Shared Libraries

**Library Structure:**
```
my-shared-library/
├── src/
│   └── com/myorg/
│       ├── Utils.groovy
│       ├── Docker.groovy
│       └── Deploy.groovy
├── vars/
│   ├── buildImage.groovy
│   ├── deployApp.groovy
│   └── runTests.groovy
└── resources/
```

**Shared Library Usage:**
```groovy
@Library('my-shared-library') _

pipeline {
  agent any
  
  stages {
    stage('Build') {
      steps {
        script {
          buildImage(
            dockerfile: 'Dockerfile',
            registry: 'myregistry.azurecr.io',
            imageName: 'myapp',
            imageTag: "${BUILD_NUMBER}"
          )
        }
      }
    }
    
    stage('Deploy') {
      steps {
        script {
          deployApp(
            environment: 'production',
            imageName: 'myapp',
            imageTag: "${BUILD_NUMBER}"
          )
        }
      }
    }
  }
}
```

---

## Part 4: GitHub Actions - Native GitHub CI/CD

### 4.1 What is GitHub Actions?

**Definition:** GitHub-native CI/CD automation platform integrated with GitHub repositories.

**Workflow Structure:**
```
GitHub Event (push, PR, schedule)
        ↓
Trigger Workflow
        ↓
Run Jobs (parallel/sequential)
        ↓
Execute Steps
        ↓
Run Actions (pre-built or custom)
        ↓
Generate Artifacts
        ↓
Post Results
```

### 4.2 GitHub Actions Workflow

**Basic Workflow:**
```yaml
name: Build and Deploy

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 2 * * *'  # 2 AM daily

env:
  REGISTRY: myregistry.azurecr.io
  IMAGE_NAME: myapp

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
      with:
        fetch-depth: 0
    
    - name: Set up JDK
      uses: actions/setup-java@v3
      with:
        java-version: '17'
        distribution: 'temurin'
    
    - name: Build with Gradle
      run: ./gradlew build
    
    - name: Run tests
      run: ./gradlew test
    
    - name: Publish test results
      uses: EnricoMi/publish-unit-test-result-action@v2
      if: always()
      with:
        files: build/test-results/**/*.xml
    
    - name: SonarQube scan
      uses: sonarsource/sonarcloud-github-action@master
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2
    
    - name: Login to container registry
      uses: docker/login-action@v2
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ secrets.REGISTRY_USERNAME }}
        password: ${{ secrets.REGISTRY_PASSWORD }}
    
    - name: Build and push Docker image
      uses: docker/build-push-action@v4
      with:
        context: .
        platforms: linux/amd64,linux/arm64
        push: true
        tags: |
          ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
        cache-from: type=gha
        cache-to: type=gha,mode=max

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
    - name: Checkout manifests
      uses: actions/checkout@v3
      with:
        repository: myorg/app-manifests
        token: ${{ secrets.DEPLOY_TOKEN }}
    
    - name: Update image tag
      run: |
        sed -i "s|image: .*|image: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}|g" \
          k8s/deployment.yaml
    
    - name: Commit and push
      run: |
        git config user.email "actions@github.com"
        git config user.name "GitHub Actions"
        git add k8s/deployment.yaml
        git commit -m "Update image to ${{ github.sha }}"
        git push
    
    - name: ArgoCD sync
      uses: appleboy/ssh-action@master
      with:
        host: ${{ secrets.ARGOCD_HOST }}
        username: root
        key: ${{ secrets.ARGOCD_SSH_KEY }}
        script: |
          argocd app sync myapp --prune

  security-scan:
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
    
    - name: Run Trivy vulnerability scan
      uses: aquasecurity/trivy-action@master
      with:
        scan-type: 'fs'
        scan-ref: '.'
        format: 'sarif'
        output: 'trivy-results.sarif'
    
    - name: Upload to GitHub Security
      uses: github/codeql-action/upload-sarif@v2
      with:
        sarif_file: 'trivy-results.sarif'
    
    - name: SAST scan with SonarQube
      uses: sonarsource/sonarcloud-github-action@master
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}

  notify:
    runs-on: ubuntu-latest
    if: always()
    needs: [build, deploy]
    
    steps:
    - name: Determine build status
      id: status
      run: |
        if [[ "${{ needs.build.result }}" == "success" && "${{ needs.deploy.result }}" == "success" ]]; then
          echo "status=success" >> $GITHUB_OUTPUT
        else
          echo "status=failure" >> $GITHUB_OUTPUT
        fi
    
    - name: Send Slack notification
      uses: slackapi/slack-github-action@v1.24.0
      with:
        payload: |
          {
            "text": "Build ${{ github.run_number }} - ${{ steps.status.outputs.status }}",
            "blocks": [
              {
                "type": "section",
                "text": {
                  "type": "mrkdwn",
                  "text": "*Deployment Status*\nBuild: ${{ needs.build.result }}\nDeploy: ${{ needs.deploy.result }}"
                }
              }
            ]
          }
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
        SLACK_WEBHOOK_TYPE: INCOMING_WEBHOOK
```

### 4.3 GitHub Actions Matrix Builds

**Matrix Strategy:**
```yaml
name: Multi-Platform Build

on: [push]

jobs:
  build:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        include:
        - os: ubuntu-latest
          arch: linux/amd64
        - os: macos-latest
          arch: darwin/amd64
        - os: ubuntu-latest
          arch: linux/arm64
          qemu: true
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up QEMU
      if: matrix.qemu
      uses: docker/setup-qemu-action@v2
    
    - name: Build for ${{ matrix.arch }}
      run: |
        echo "Building for ${{ matrix.arch }}"
        make build ARCH=${{ matrix.arch }}
    
    - name: Upload artifacts
      uses: actions/upload-artifact@v3
      with:
        name: build-${{ matrix.arch }}
        path: dist/
```

### 4.4 GitHub Actions Secrets & Environment

**Store Secrets:**
```bash
# Via CLI
gh secret set SECRET_NAME --body "secret_value"

# Or via GitHub UI: Settings > Secrets and variables > Actions
```

**Use Secrets in Workflow:**
```yaml
steps:
- name: Deploy with credentials
  env:
    DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
    DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
  run: |
    echo "Using secret credentials"
    ./deploy.sh
```

**Environment Variables & Outputs:**
```yaml
jobs:
  job1:
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.vars.outputs.image-tag }}
    
    steps:
    - name: Set variables
      id: vars
      run: |
        echo "image-tag=$(date +%Y%m%d-%H%M%S)" >> $GITHUB_OUTPUT
        echo "BUILD_NUMBER=${{ github.run_number }}" >> $GITHUB_ENV
    
  job2:
    runs-on: ubuntu-latest
    needs: job1
    
    steps:
    - name: Use output from job1
      run: |
        echo "Image tag: ${{ needs.job1.outputs.image-tag }}"
```

---

## Part 5: Azure DevOps - Complete DevOps Platform

### 5.1 What is Azure DevOps?

**Definition:** Complete DevOps platform with Repos, Pipelines, Boards, Artifacts, and Tests.

**Components:**
```
┌─────────────────────────────────────┐
│      Azure DevOps Organization      │
├─────────────────────────────────────┤
│ • Azure Repos (Git repositories)    │
│ • Azure Pipelines (CI/CD)           │
│ • Azure Boards (Work tracking)      │
│ • Azure Artifacts (Package mgmt)    │
│ • Azure Test Plans (Testing)        │
└─────────────────────────────────────┘
```

### 5.2 Azure DevOps Pipelines

**YAML Pipeline Structure:**
```yaml
trigger:
  branches:
    include:
    - main
    - develop
  paths:
    include:
    - src/
    - pipelines/

pr:
  branches:
    include:
    - main

schedules:
- cron: "0 2 * * *"
  displayName: Daily build
  branches:
    include:
    - main

pool:
  vmImage: 'ubuntu-latest'

variables:
  buildConfiguration: 'Release'
  dockerRegistryServiceConnection: 'myregistry'
  imageRepository: 'myapp'
  containerRegistry: 'myregistry.azurecr.io'
  dockerfilePath: '$(Build.SourcesDirectory)/Dockerfile'

stages:
- stage: Build
  displayName: Build stage
  jobs:
  - job: Build
    displayName: Build Job
    steps:
    - task: UseDotNet@2
      inputs:
        version: '7.x'
        includePreviewVersions: false
    
    - task: DotNetCoreCLI@2
      displayName: 'dotnet build'
      inputs:
        command: 'build'
        arguments: '--configuration $(buildConfiguration)'
    
    - task: DotNetCoreCLI@2
      displayName: 'dotnet test'
      inputs:
        command: 'test'
        arguments: '--configuration $(buildConfiguration) --logger trx --collect:"XPlat Code Coverage"'
    
    - task: PublishCodeCoverageResults@1
      displayName: 'Publish code coverage'
      inputs:
        codeCoverageTool: Cobertura
        summaryFileLocation: '$(Agent.TempDirectory)/**/*coverage.cobertura.xml'
    
    - task: PublishTestResults@2
      displayName: 'Publish test results'
      inputs:
        testResultsFormat: 'VSTest'
        testResultsFiles: '**/*.trx'

- stage: BuildImage
  displayName: Build Docker Image
  dependsOn: Build
  condition: succeeded()
  
  jobs:
  - job: BuildAndPush
    displayName: Build and Push Image
    steps:
    - task: Docker@2
      displayName: 'Build and push image'
      inputs:
        command: 'buildAndPush'
        repository: '$(imageRepository)'
        dockerfile: '$(dockerfilePath)'
        containerRegistry: '$(dockerRegistryServiceConnection)'
        tags: |
          $(Build.BuildNumber)
          latest

- stage: Deploy
  displayName: Deploy to Kubernetes
  dependsOn: BuildImage
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
  
  jobs:
  - deployment: Deploy
    displayName: Deploy Job
    environment: 'production'
    
    strategy:
      runOnce:
        deploy:
          steps:
          - task: KubernetesManifest@0
            displayName: 'Deploy to Kubernetes'
            inputs:
              action: 'deploy'
              kubernetesServiceConnection: 'my-k8s-connection'
              namespace: 'production'
              manifests: |
                $(Pipeline.Workspace)/manifests/deployment.yaml
                $(Pipeline.Workspace)/manifests/service.yaml
          
          - task: Kubernetes@1
            displayName: 'Rollout status'
            inputs:
              connectionType: 'Kubernetes Service Connection'
              kubernetesServiceEndpoint: 'my-k8s-connection'
              command: 'rollout'
              arguments: 'status deployment/myapp -n production'
```

**Multi-Stage Pipeline:**
```yaml
stages:
- stage: Dev
  displayName: Deploy to Dev
  variables:
    environment: 'dev'
  jobs:
  - deployment: DeployDev
    environment: dev
    strategy:
      runOnce:
        deploy:
          steps:
          - script: echo Deploying to DEV
            displayName: Deploy to DEV

- stage: Staging
  displayName: Deploy to Staging
  dependsOn: Dev
  variables:
    environment: 'staging'
  jobs:
  - deployment: DeployStaging
    environment: staging
    strategy:
      runOnce:
        deploy:
          steps:
          - script: echo Deploying to STAGING
            displayName: Deploy to STAGING

- stage: Production
  displayName: Deploy to Production
  dependsOn: Staging
  variables:
    environment: 'production'
  jobs:
  - deployment: DeployProduction
    environment: production
    strategy:
      canary:
        increments: [25, 50, 75, 100]
        preDeploy:
          steps:
          - script: echo Pre-deploy validation
        deploy:
          steps:
          - script: echo Deploying to PRODUCTION
        postDeploy:
          steps:
          - script: echo Post-deploy verification
```

### 5.3 Azure Pipelines Templates

**Reusable Template:**
```yaml
# templates/build-and-push.yml
parameters:
  - name: dockerfilePath
    type: string
    default: 'Dockerfile'
  - name: imageRepository
    type: string
  - name: imageTags
    type: object
    default: ['latest']

jobs:
- job: BuildAndPush
  displayName: Build and Push Docker Image
  steps:
  - task: Docker@2
    displayName: 'Build and push'
    inputs:
      command: 'buildAndPush'
      Dockerfile: ${{ parameters.dockerfilePath }}
      repository: ${{ parameters.imageRepository }}
      tags: ${{ join(',', parameters.imageTags) }}
```

**Use Template:**
```yaml
stages:
- stage: Build
  jobs:
  - template: templates/build-and-push.yml
    parameters:
      dockerfilePath: 'Dockerfile'
      imageRepository: 'myapp'
      imageTags:
        - 'v1.0'
        - 'latest'
```

### 5.4 Azure Release Pipelines

**Classic Release Pipeline:**
```yaml
# Using YAML (recommended approach)
trigger:
  branches:
    include:
    - main

pool:
  vmImage: 'ubuntu-latest'

stages:
- stage: Release
  displayName: Create Release
  jobs:
  - job: CreateRelease
    displayName: Create GitHub Release
    steps:
    - task: GitHubRelease@1
      inputs:
        gitHubConnection: 'github-connection'
        repositoryName: 'myorg/myapp'
        action: 'create'
        target: '$(Build.SourceVersion)'
        tagSource: 'userSpecifiedTag'
        tag: 'v$(Build.BuildNumber)'
        title: 'Release v$(Build.BuildNumber)'
        releaseNotesSource: 'inline'
        releaseNotesInline: 'Release notes here'
        assets: '$(Build.ArtifactStagingDirectory)/*'
        isPreRelease: false
```

---

## Part 6: Integration Patterns

### 6.1 GitHub Actions + ArgoCD

**GitOps Deployment Flow:**
```yaml
# .github/workflows/deploy.yml
name: Push to GitOps Repo

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Build and push image
      # ... build steps ...
      run: |
        docker build -t myregistry.azurecr.io/myapp:${{ github.sha }} .
        docker push myregistry.azurecr.io/myapp:${{ github.sha }}
    
    - name: Update manifests in GitOps repo
      uses: actions/github-script@v6
      with:
        github-token: ${{ secrets.GITOPS_TOKEN }}
        script: |
          const fs = require('fs');
          const { Octokit } = require("@octokit/rest");
          
          const octokit = new Octokit({
            auth: process.env.GITHUB_TOKEN
          });
          
          // Update image in manifests
          const manifest = fs.readFileSync('k8s/deployment.yaml', 'utf8')
            .replace(/image: .*/g, `image: myregistry.azurecr.io/myapp:${{ github.sha }}`);
          
          // Push to gitops repo
          await octokit.repos.createOrUpdateFileContents({
            owner: 'myorg',
            repo: 'app-gitops',
            path: 'kubernetes/deployment.yaml',
            message: 'Update image to ${{ github.sha }}',
            content: Buffer.from(manifest).toString('base64'),
            branch: 'main'
          });
    
    # ArgoCD automatically picks up the change
```

### 6.2 Jenkins + ArgoCD

**Jenkins Job Triggering ArgoCD Sync:**
```groovy
pipeline {
  agent any
  
  stages {
    stage('Build & Test') {
      steps {
        sh '''
          ./gradlew build
          ./gradlew test
        '''
      }
    }
    
    stage('Build Image') {
      steps {
        sh '''
          docker build -t myregistry/myapp:${BUILD_NUMBER} .
          docker push myregistry/myapp:${BUILD_NUMBER}
        '''
      }
    }
    
    stage('Update GitOps Repo') {
      steps {
        sh '''
          git clone https://github.com/myorg/app-gitops.git
          cd app-gitops
          sed -i "s|image: .*|image: myregistry/myapp:${BUILD_NUMBER}|" k8s/deployment.yaml
          git add k8s/deployment.yaml
          git commit -m "Update image to ${BUILD_NUMBER}"
          git push
        '''
      }
    }
    
    stage('Sync ArgoCD') {
      steps {
        sh '''
          argocd app sync myapp --prune
          argocd app wait myapp
        '''
      }
    }
  }
}
```

### 6.3 Azure DevOps + Azure Container Registry

**Integrated Pipeline:**
```yaml
trigger:
  - main

pool:
  vmImage: 'ubuntu-latest'

variables:
  dockerRegistryServiceConnection: 'acr-connection'
  imageRepository: 'myapp'
  containerRegistry: 'myregistry.azurecr.io'

stages:
- stage: Build_And_Push
  displayName: Build and Push
  jobs:
  - job: Build
    displayName: Build Job
    steps:
    - task: Docker@2
      displayName: Build and push image
      inputs:
        command: buildAndPush
        repository: $(imageRepository)
        dockerfile: $(Build.SourcesDirectory)/Dockerfile
        containerRegistry: $(dockerRegistryServiceConnection)
        tags: |
          $(Build.BuildId)
          latest

- stage: Deploy_Dev
  displayName: Deploy to Dev
  dependsOn: Build_And_Push
  jobs:
  - deployment: Deploy
    displayName: Deploy to AKS Dev
    environment: 'dev'
    strategy:
      runOnce:
        deploy:
          steps:
          - task: KubernetesManifest@0
            displayName: Deploy to Kubernetes
            inputs:
              action: deploy
              kubernetesServiceConnection: 'dev-aks'
              namespace: dev
              manifests: |
                $(Pipeline.Workspace)/manifests/*.yaml
              imagePullSecrets: |
                acr-secret
              containers: |
                $(containerRegistry)/$(imageRepository):$(Build.BuildId)
```

---

## Part 7: Comparison & Best Practices

### Comparison Matrix

| Feature | ArgoCD | Jenkins | GitHub Actions | Azure DevOps |
|---------|--------|---------|-----------------|--------------|
| **Type** | GitOps Controller | CI/CD Server | Cloud CI/CD | Platform |
| **Architecture** | Declarative | Imperative | Cloud-native | Cloud-native |
| **Setup** | Easy (K8s native) | Complex | Very easy | Medium |
| **Cost** | Free | Free (OSS) | Free tier | Pay-per-agent |
| **Git Integration** | Native | Plugin | Native | Native |
| **Container Registry** | All | All | GitHub, Azure | Azure ACR |
| **Kubernetes Support** | Excellent | Via plugins | Via actions | Excellent |
| **Multi-cloud** | Excellent | Excellent | Limited | Excellent |
| **UI/UX** | Excellent | Good | Excellent | Excellent |
| **Audit Trail** | Full (Git-based) | Job logs | Full (Actions) | Full |
| **Self-healing** | Yes (native) | Via plugins | Manual | Manual |
| **Community** | Growing | Large | Large | Medium |
| **Best For** | GitOps, K8s | Complex builds | GitHub repos | Enterprise |

### Best Practices by Tool

**ArgoCD Best Practices:**
- Use separate repos for apps and infrastructure
- Implement ApplicationSets for multi-environment
- Use Project-based RBAC
- Enable notifications for deployment events
- Regular backup of ArgoCD state
- Use kustomize or Helm for templating
- Implement health checks in Application specs

**Jenkins Best Practices:**
- Use Jenkins Configuration as Code (JCasC)
- Declarative Pipelines over Scripted
- Use shared libraries for reusable logic
- Implement proper RBAC
- Use agents for isolation
- Monitor Jenkins performance
- Regular backups of configuration

**GitHub Actions Best Practices:**
- Use matrix builds for multi-platform
- Leverage reusable workflows
- Pin action versions (avoid @latest)
- Use environments for deployment controls
- Implement OIDC for cloud authentication
- Use branch protection rules
- Regular review of Action dependencies

**Azure DevOps Best Practices:**
- Use YAML pipelines (avoid classic)
- Implement multi-stage pipelines
- Use service connections securely
- Implement approval gates
- Use variable groups for secrets
- Regular pipeline audits
- Use templates for consistency

---

## Part 8: Interview Questions

### ArgoCD Questions

**Q1: How does ArgoCD differ from traditional CI/CD tools like Jenkins?**
A: ArgoCD is pull-based GitOps controller vs push-based CI/CD:
- Git is single source of truth
- Automatic drift detection and correction
- Declarative, version-controlled deployments
- No sensitive credentials in CI system
- Atomic rollback via git revert

**Q2: Explain ApplicationSet and when you'd use it.**
A: ApplicationSet generates multiple Applications from templates:
- Use cases: multi-environment, multi-cluster, multi-tenant
- Generators: list, matrix, git, cluster, pull request
- Reduces manifest duplication
- Enables GitOps at scale

**Q3: How would you implement Canary Deployment in ArgoCD?**
A: Use plugins or external controllers:
- Argo Rollouts: Progressive delivery with traffic shifting
- Define Rollout instead of Deployment
- Configure traffic split percentages
- Monitor metrics for health assessment
- Automatic promotion or manual approval

### Jenkins Questions

**Q4: Explain the difference between Declarative and Scripted Pipelines.**
A:
- **Declarative**: Structured, easier to read, limited flexibility
- **Scripted**: Full Groovy power, complex logic, steeper learning curve
- Declarative recommended for most use cases

**Q5: How do you handle sensitive credentials in Jenkins?**
A:
- Use Jenkins Credentials Plugin
- Store in Jenkins Secrets Manager
- Reference via `credentials()` function
- Inject as environment variables
- Never commit to Git

### GitHub Actions Questions

**Q6: How do you optimize GitHub Actions workflow performance?**
A:
- Use matrix builds for parallelization
- Cache dependencies (actions/cache)
- Use self-hosted runners for long jobs
- Parallelize independent jobs
- Use smaller base images
- Pin action versions for consistency

**Q7: Explain GitHub Actions Environments and their use.**
A: Environments allow:
- Environment-specific secrets
- Required reviewers for deployments
- Protection rules (branch restrictions)
- Deployment history tracking
- Separate OIDC subjects per environment

### Azure DevOps Questions

**Q8: What's the difference between Pipelines and Release Pipelines?**
A:
- **Pipelines (YAML)**: Modern, version-controlled, recommended
- **Release Pipelines (Classic)**: Legacy UI-based, being phased out
- Migrate to YAML for CI/CD as code

**Q9: How do you secure Azure DevOps Pipelines?**
A:
- Use service connections with minimal permissions
- Store secrets in Azure Key Vault
- Implement approval gates
- Use branch protection rules
- Regular audit of access
- Use managed identities for Azure resources

### General CI/CD Questions

**Q10: Design a complete CI/CD pipeline for a microservices application.**
A: Architecture:
1. **Code Stage**: Push to repo triggers pipeline
2. **Build Stage**: Compile, unit test, code quality scan
3. **Package Stage**: Build Docker image, scan vulnerabilities
4. **Registry Stage**: Push to container registry
5. **Deploy Dev**: Automated deployment to dev
6. **Deploy Staging**: Automated deployment with tests
7. **Deploy Prod**: Manual approval, canary deployment
8. **Monitoring**: Health checks, observability

**Q11: How would you implement GitOps for your infrastructure?**
A:
1. All infrastructure as code in Git
2. ArgoCD syncs from Git to cluster
3. Pull requests for changes
4. Automatic drift correction
5. Full audit trail
6. Easy rollback via git revert

**Q12: Explain your CI/CD strategy for handling multiple environments.**
A:
- Single pipeline definition, environment-specific variables
- Different promotion gates per environment
- Secrets management per environment
- Health checks per environment
- Rollback strategy per environment
- Monitoring and alerting per environment

---

## Summary Recommendation Matrix

**Choose ArgoCD if:**
- You want pure GitOps
- Kubernetes-first deployment
- Need declarative configuration
- Want automatic drift correction
- Multi-cluster management

**Choose Jenkins if:**
- Complex build logic needed
- Legacy system integration required
- Need full pipeline customization
- On-premises deployment
- Extensive plugin ecosystem needed

**Choose GitHub Actions if:**
- Repository hosted on GitHub
- Simple to medium complexity
- Cloud-native preference
- Want tight GitHub integration
- Cost-conscious

**Choose Azure DevOps if:**
- Enterprise environment
- Microsoft technology stack
- Need complete DevOps platform
- Azure cloud preference
- Want integrated work tracking

---

**Study Time Estimate:** 10-12 hours
**Key Takeaway:** Most organizations use multiple tools - GitOps (ArgoCD) for deployment, CI/CD (Jenkins/GitHub Actions/Azure DevOps) for building, and orchestration of these creates modern deployment pipeline.
