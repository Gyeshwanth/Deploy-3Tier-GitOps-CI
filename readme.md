
# DevSecOps Project - End-to-End CI/CD Setup Documentation

---

## 🏗️ Infrastructure Setup

**Tools and Components:**

* **GitHub** – Source code repository.
* **Jenkins** – CI/CD automation server.
* **SonarQube** – Static code analysis for quality and security.
* **Trivy** – Scanning dependencies and Docker images for vulnerabilities.
* **Docker** – Containerization of applications.
* **Kubernetes (K8s)** – Container orchestration and management platform.

---

| Language | Dependency File  |
| -------- | ---------------- |
| Java     | pom.xml          |
| Node.js  | package.json     |
| .NET     | .csproj          |
| Python   | requirements.txt |

---

## ⚙️ CI/CD Workflow Overview

**Pipeline Flow:**

```
GitHub → Compile → Gitleaks (Secrets Check) → Trivy (Dependency Vulnerability Scan) →
SonarQube (Static Code Analysis) → Quality Gate Check → Docker Build →
Trivy (Image Vulnerability Scan) → DockerHub Push → Kubernetes Deployment
```

---


## ☁️ Server Setup

**EC2 Configuration:**

* Create **3 EC2 instances**:

  * Jenkins Server
  * SonarQube Server
  * **EKS Master Node** (for running kubectl commands)
* Instance Type: `t2.medium`
* OS: **Ubuntu 24**
* Configure Security Group and Key Pair
* Connect to servers using **MobaXterm**

---



## 🧠 SonarQube Setup (with Docker)  Do this SonarQube-server(Ec2)

**1. Install Docker:**

```bash
sudo apt update
sudo apt install docker.io -y
sudo systemctl start docker
sudo systemctl enable docker
```

**2. Run SonarQube Container:**

```bash
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
```

**3. Access SonarQube:**

```
http://<SONARQUBE_PUBLIC_IP>:9000
```

Default credentials:

```
Username: admin
Password: admin
```

Change password → `java`

---

## ⚙️ Jenkins Setup  Do this Jenkins-server(Ec2)


**1. Install Java:**
Use **OpenJDK 21 Headless** (no GUI support)


**2. Install Jenkins:**

```bash
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update
sudo apt install jenkins -y
```

**3. Access Jenkins:**

```
http://<JENKINS_PUBLIC_IP>:8080
```

Retrieve initial admin password:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

**4. Configure Jenkins:**

* Install **Suggested Plugins**
* Create Admin User → Username: `yeshwanth`, Password: `java`

---


## 🔐 Gitleaks Setup (Secrets Scanning)

```bash
GITLEAKS_VERSION=$(curl -s "https://api.github.com/repos/gitleaks/gitleaks/releases/latest" | grep -Po '"tag_name": "v\K[0-9.]+')
wget -qO gitleaks.tar.gz https://github.com/gitleaks/gitleaks/releases/latest/download/gitleaks_${GITLEAKS_VERSION}_linux_x64.tar.gz
sudo tar xf gitleaks.tar.gz -C /usr/local/bin gitleaks
gitleaks version
rm -rf gitleaks.tar.gz
```

---

## 🧰 Trivy Setup (Vulnerability Scanning)

```bash
sudo apt-get install wget apt-transport-https gnupg lsb-release -y
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy -y

# Verify version
trivy --version
```

**Trivy scan results locations:**

* File system scan (FS): `fs-report.html` inside Jenkins workspace
* Docker image scan: `frontend-image-report.html` and `backend-image-report.html`

Access after pipeline run:

```bash
cd /var/lib/jenkins/workspace/<JOB_NAME>/
ls -l
```

---

## 🔌 Jenkins Plugins

* **NodeJS Plugin**
* **Pipeline Stage View**
* **SonarQube Scanner for Jenkins**
* **Docker Pipeline**
* **Kubernetes CLI**
* **Kubernetes**
* **Kubernetes Credentials**
* **Generic Webhook Trigger**
---

## 🛠️ Jenkins Configuration

### NodeJS Tool Setup

* Navigate to: **Manage Jenkins → Tools → NodeJS installations**
* Add a NodeJS version (e.g., `nodejs23`)
* Reference inside pipeline:

```groovy
tools { nodejs 'nodejs23' }
```

### SonarQube Scanner Setup

1. **Manage Jenkins → Tools → SonarQube Scanner → Add Installation**

   * Name: `sonar-scanner`
2. **Generate and Add SonarQube Token**

   * SonarQube → **Administration → Security → Tokens → Generate Token**
   * Jenkins → **Manage Credentials → Global → Add Credentials** → Secret Text → ID: `sonar-token`
3. **Add SonarQube Server in Jenkins**

   * Name: `sonar`
   * URL: `http://<SONARQUBE_IP>:9000`
   * Authentication Token: `sonar-token`

### Webhook Configuration (For Quality Gate)

* SonarQube → **Administration → Configuration → Webhooks → Create New**

```
Name: Jenkins-Webhook
URL: http://<JENKINS_IP>:8080/sonarqube-webhook/
```

---



## 🐳 Docker Setup for CI/CD 
* Install Docker and Docker-compose*

```bash
docker 
sudo apt install docker.io
sudo usermod -aG docker jenkins
sudo systemctl restart docker  or  exit

```
```bash

sudo  curl -SL https://github.com/docker/compose/releases/download/v2.40.1/docker-compose-linux-x86_64 -o /usr/local/bin/docker-compose

sudo chmod +x /usr/local/bin/docker-compose

```

### Dockerfile Recommendations (Multi-stage Build)

* **Frontend and Backend Dockerfiles** should use multi-stage builds to optimize image size.
* For when to use the Java application, use JDK for compilation in the intermediate stage and JRE/base image for runtime.
* Example for Node.js:

```
# Stage 1: Build
FROM node:18-alpine as builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# Stage 2: Production
FROM nginx:alpine
COPY --from=builder /app/build /usr/share/nginx/html
```

### Docker-Compose (Root Project Directory)

```yaml
version: '3.8'

services:
  mysql:
    image: mysql:8
    container_name: mysql
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: java
      MYSQL_DATABASE: crud_app
    volumes:
      - mysql-data:/var/lib/mysql
      - ./mysql-init:/docker-entrypoint-initdb.d
    ports:
      - "3306:3306"

  backend:
    build: ./api
    container_name: backend-api
    environment:
      DB_HOST: mysql
      DB_USER: root
      DB_PASSWORD: java
      DB_NAME: crud_app
      JWT_SECRET: SuperSecretKey
      RESET_ADMIN_PASS: 'true'
    depends_on:
      - mysql
    ports:
      - "5000:5000"

  frontend:
    build: ./client
    container_name: frontend-react
    ports:
      - "3000:80"
    depends_on:
      - backend

volumes:
  mysql-data:
```

### Docker Commands for Checking & Cleanup

* **List running containers:** `docker ps`
* **List all containers:** `docker ps -a`
* **Container logs:** `docker logs <container_id>`
* **Stop all containers:** `docker stop $(docker ps -aq)`
* **Remove all containers:** `docker rm $(docker ps -aq)`
* **Remove all images:** `docker rmi -f $(docker images -aq)`

---

## 🧪 Jenkins Pipeline Script (Full DevSecOps Flow)

```groovy
pipeline {
    agent any

    tools {
        nodejs 'nodejs23'
    }

   environment {
       SCANNER_HOME = tool 'sonar-scanner'
   }


    stages {
        stage('git checkout') {
            steps {
                git branch: 'main', credentialsId: 'gitcred', url: 'https://github.com/Gyeshwanth/devops-project.git'
            }
        }
        
         stage('frontend compile') {
            steps {
                 dir('client') {
                      sh 'find . -name "*.js" -exec node --check {} +'
                 }
            }
        }
        
        stage('backend compile') {
            steps {
                 dir('api') {
                      sh 'find . -name "*.js" -exec node --check {} +'
                 }
            }
        }
        
        stage('git leaks') {
            steps {
                sh 'gitleaks detect --source ./client --exit-code 1'
                sh 'gitleaks detect --source ./api --exit-code 1' 
            }
        }
        
         stage('SonarQube Analysis') {
            steps {
               withSonarQubeEnv('sonar') {
          sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=NodeJS-Project \
                            -Dsonar.projectKey=NodeJS-Project '''
             }
            }
        }
        
         stage('Quality Gate Check') {
            steps {
             timeout(time: 1, unit: 'HOURS') {
               waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
             }
                  }
         }
        
        
         stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs --format table -o fs-report.html .'
            }
        }
        
         stage('Build-Tag & Push Backend Docker Image') {
            steps {
              script {
        withDockerRegistry(credentialsId: 'docker-cred') {
          
           dir('api') {
               sh 'docker build -t yeshwanthgosi/backend:latest .'
                sh 'trivy image --format table -o backend-image-report.html yeshwanthgosi/backend:latest '
               sh 'docker push yeshwanthgosi/backend:latest'
           }
              }
            }
        }
    }
         stage('Build-Tag & Push Frontend Docker Image') {
            steps {
              script {
        withDockerRegistry(credentialsId: 'docker-cred') {
          
           dir('client') {
               sh 'docker build -t yeshwanthgosi/frontend:latest .'
               sh 'trivy image --format table -o frontend-image-report.html yeshwanthgosi/frontend:latest '
               sh 'docker push yeshwanthgosi/frontend:latest'
           }
              }
            }
        }
    }
      
      
       stage('k8s-deploy') {
        steps {
             script {
                  
    withKubeConfig(caCertificate: '', clusterName: ' yesh-cluster', contextName: '', credentialsId: 'k8s-token', namespace: 'prod', restrictKubeConfigAccess: false, serverUrl: 'https://7CEBD932F89B3BD349EDE73AD44A8264.sk1.ap-south-1.eks.amazonaws.com') {
                        sh 'kubectl apply -f k8s-prod/sc.yaml'
                        sleep 20
                        sh 'kubectl apply -f k8s-prod/mysql.yaml -n prod'
                        sh 'kubectl apply -f k8s-prod/backend.yaml -n prod'
                        sh 'kubectl apply -f k8s-prod/frontend.yaml -n prod'
                        sh 'kubectl apply -f k8s-prod/ci.yaml'
                        sh 'kubectl apply -f k8s-prod/ingress.yaml -n prod'
                        sleep 30
        }
             }
        }
    }
    
   
  stage('verify-k8s-deploy') {
        steps {
             script {
                  
    withKubeConfig(caCertificate: '', clusterName: ' yesh-cluster', contextName: '', credentialsId: 'k8s-token', namespace: 'prod', restrictKubeConfigAccess: false, serverUrl: 'https://7CEBD932F89B3BD349EDE73AD44A8264.sk1.ap-south-1.eks.amazonaws.com') {
                       sh 'kubectl get pods -n prod'
                        sleep 20
                         sh 'kubectl get ingress -n prod'
       
        }
             }
        }
    }
 


        
    }
}

```


---

## 🧾 Viewing Reports

Navigate to Jenkins workspace:

```bash
cd /var/lib/jenkins/workspace/<JOB_NAME>/
ls -l
```

Look for:

```
fs-report.html
backend-image-report.html
frontend-image-report.html
```

Open in Jenkins or browser to review vulnerabilities.

---

# EKS Master Node Setup for Jenkins Integration

This document describes how to set up an EC2 instance as an EKS Master Node to run `kubectl` commands and integrate with Jenkins pipelines for Kubernetes automation.

---

## 1. Launch EC2 Instance

**Instance Type:** `t2.medium`

**Steps:**

1. Login to AWS Management Console.
2. Navigate to **EC2 → Launch Instance**.
3. Choose Ubuntu  (or preferred Linux distro).
4. Select instance type `t2.medium`.
5. Configure security group (allow SSH and necessary ports).
6. Launch instance and connect via SSH:

```bash
ssh -i your-key.pem ubuntu@<EC2_PUBLIC_IP>
```

---

## 2. Install Required Tools & Plugins

**Refer URL Below for EKS-Step**

https://github.com/Gyeshwanth/devops-project/blob/main/EKS-Setup.md

**Plugins Required for Jenkins:**

* Kubernetes CLI
* Kubernetes
* Kubernetes Credentials
---

## 6. Apply Kubernetes RBAC YAMLs for Jenkins Automation

Reference RBAC YAML files: [RBAC YAMLs](https://github.com/Gyeshwanth/Terraform-devops-project/blob/main/RBAC/rbac.md)

**Steps:**

1. Create namespace `prod`.
2. Create secret YAML `secret.yaml`:

```yaml
apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
  name: mysecretname
  annotations:
    kubernetes.io/service-account.name: jenkins
```

3. Apply secret:

```bash
kubectl apply -f secret.yaml -n prod
```

4. Retrieve token:

```bash
kubectl describe secret mysecretname -n prod
```

*Copy token → Jenkins → Manage → Credentials → Secret Text → ID: `k8s-token`*

---

## 7. Configure Jenkins for EKS

1. Copy the **EKS cluster endpoint** from AWS → EKS → Cluster.
2. Install `kubectl` on Jenkins EC2 if not installed.
3. Configure Jenkins pipeline to use `k8s-token` and cluster endpoint for deploying resources.

---

This completes the setup of an EC2 instance as the EKS Master Node for running `kubectl` commands and integrating Kubernetes automation with Jenkins.

---


# 📘 Jenkins Webhook & CD Setup Guide

## Automation with Webhook

You can automate your Jenkins pipelines using either the **GitHub plugin** or **Generic Webhook Trigger**.

* **Generic Webhook Trigger**: Flexible, supports multiple integration sources with Jenkins automation.
* **GitHub Plugin**: Only supports GitHub repositories.

---

## Step 1: Install Plugin Generic Webhook Trigger

1. Go to your Jenkins **Pipeline** → **Current Build** → **Configure**.
2. Under **Triggers**, select **Generic Webhook Trigger**.

## Step 2: Configure Webhook Parameters

1. In the **Post Parameter** section, add:

   * **Variable**: `ref`
   * **Expression (JSONPath)**: `$.ref`

## Step 3: Token Setup

1. Provide any string as the **token**.
2. You can make the token hidden by selecting the **Credentials** section and specifying the token.

## Step 4: Optional Filter

1. In the **Optional Filter** section, configure:

   * **Expression**: `refs/heads/branch_name`
   * **Text**: `$ref`
2. Apply and save.
3. Copy the webhook URL:

   ```
   http://jenkins_url/generic-webhook-trigger/invoke?token=yourtoken
   ```

## Step 5: Configure GitHub Webhook

1. Go to **GitHub Repository** → **Settings** → **Webhooks**.
2. Click **Add webhook**.
3. Set **Payload URL** to the copied Jenkins URL.
4. Set **Content type** to `application/json`.
5. Select the specific events (e.g., **Push**) to trigger the pipeline.
6. Verify webhook by checking **Recent Deliveries**.

---

## Continuous Delivery vs Continuous Deployment

* **Continuous Delivery (CD)**: Requires manual permission to deploy.
* **Continuous Deployment (CD)**: Deployment happens automatically without manual approval.

### To enable manual approval in pipeline:

```groovy
stage('Manual Approval for Production') {
    steps {
        timeout(time: 1, unit: 'HOURS') {
            input message: 'Approve deployment to PRODUCTION?', ok: 'Deploy'
        }
    }
}
```

* If not approved within 1 hour, deployment triggers automatically.

---

## Kubernetes Resources Verification

* Check all resources under **prod namespace**:

```bash
kubectl get all -n prod
```

* Check **Ingress**:

```bash
kubectl get ingress -n prod
```

* Copy the **ADDRESS URL**.
* Run `nslookup` to get IP address and details:

```bash
nslookup <copied_address>
```

---

## Domain Mapping (GoDaddy)

1. Buy domain on GoDaddy → **Profile** → **Products**.
2. Configure DNS:

   * **Type A** → `@` → Value: Non-authoritative IP address from `nslookup`.
   * **Type CNAME** → `www` → URL: Non-authoritative URL from `nslookup`.
3. Verify DNS mapping on [https://www.whatsmydns.net](https://www.whatsmydns.net) using **A** and **CNAME**.

---

## SSL/TLS Verification

* Check certificate:

```bash
kubectl get certificate -n prod
```

* If `READY=True`, SSL is active.
* Describe certificate details:

```bash
kubectl describe certificate <secretName> -n prod
kubectl describe certificate yeshwanth-co-tls -n prod
```

* Verify HTTPS access in browser for secure endpoints.

---

This setup ensures:

* Automated Jenkins pipeline triggers via GitHub or other sources.
* Controlled or automatic deployments with Continuous Delivery/Deployment.
* Secure, DNS-mapped endpoints with SSL/TLS verification.

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/535c14fd-bbdf-4a51-b150-308948707566" />
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/972b9a17-3ecd-48b3-8988-90baf9b79a06" />


# EKS Monitoring Setup with Prometheus and Grafana

This guide provides steps to set up a complete monitoring stack on EKS EC2 instances using Prometheus, Grafana, Node Exporter, Kube-State-Metrics, and Alertmanager.

---

##

* EC2 instances running  (EKS cluster)
* Kubectl configured to access the cluster
*  download Helm and chart repositories

---

## 1. Install Helm

Helm is a package manager for Kubernetes that simplifies application deployment.

```bash
sudo apt update && sudo apt upgrade -y
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

Verify installation:

```bash
helm version
```

---

## 2. Add Prometheus Helm repository

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

---

## 3. Create a directory for monitoring configs

```bash
mkdir monitoring && cd monitoring
```

Create a custom `values.yaml`:

```yaml
alertmanager:
  enabled: false
prometheus:
  prometheusSpec:
    service:
      type: LoadBalancer
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: ebs-sc
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 5Gi
grafana:
  enabled: true
  service:
    type: LoadBalancer
  adminUser: admin
  adminPassword: admin123
nodeExporter:
  service:
    type: LoadBalancer
kubeStateMetrics:
  enabled: true
  service:
    type: LoadBalancer
additionalScrapeConfigs:
  - job_name: node-exporter
    static_configs:
      - targets:
          - node-exporter:9100
  - job_name: kube-state-metrics
    static_configs:
      - targets:
          - kube-state-metrics:8080
```

> **Note:** This configuration exposes Grafana, Prometheus, Node Exporter, and Kube-State-Metrics via LoadBalancer for external access and uses EBS volumes for persistence.

---

## 4. Deploy the monitoring stack

```bash
helm upgrade --install monitoring prometheus-community/kube-prometheus-stack -f values.yaml -n monitoring --create-namespace
```

Verify deployment:

```bash
kubectl get all -n monitoring
```

---

## 5. Patch services for LoadBalancer access (if needed)

```bash
kubectl patch svc monitoring-kube-prometheus-prometheus -n monitoring -p '{"spec": {"type": "LoadBalancer"}}'
kubectl patch svc monitoring-kube-state-metrics -n monitoring -p '{"spec": {"type": "LoadBalancer"}}'
kubectl patch svc monitoring-prometheus-node-exporter -n monitoring -p '{"spec": {"type": "LoadBalancer"}}'
```

> This ensures Prometheus, Node Exporter, and Kube-State-Metrics are accessible externally.

---

## 6. Components Installed and Their Purpose

| Component                       | Purpose                                                                               |
| ------------------------------- | ------------------------------------------------------------------------------------- |
| **Prometheus**                  | Scrapes cluster and node metrics; stores time-series data                             |
| **Grafana**                     | Preconfigured dashboards for Kubernetes, Nodes, Pods; visualization layer             |
| **Node Exporter**               | Metrics for CPU, Memory, Disk, Network per node                                       |
| **Kube-State-Metrics**          | Metrics for cluster object health (Pods, Deployments, PVCs, Jobs)                     |
| **Alertmanager**                | Preconfigured alert rules for common cluster issues (disabled in current values.yaml) |
| **Custom Resource Definitions** | ServiceMonitors and PrometheusRules for dynamic metric discovery and alerting         |
| **RBAC & ServiceAccounts**      | Proper permissions for all components to access metrics and resources                 |

---

## 7. Current Configuration Summary (Based on `values.yaml`)

* **Alertmanager:** Disabled → no alerts will be sent
* **Prometheus:** LoadBalancer service, 5Gi EBS storage
* **Grafana:** LoadBalancer service, admin credentials set to `admin/admin123`
* **Node Exporter:** LoadBalancer service (exposed externally, typically should be ClusterIP)
* **Kube-State-Metrics:** LoadBalancer service (exposed externally, typically should be ClusterIP)
* **Additional Scrape Configs:** Node Exporter and Kube-State-Metrics manually added (redundant; ServiceMonitors handle this automatically)

> ⚠️ **Note:** Exposing Node Exporter and Kube-State-Metrics externally is generally not recommended in production; internal ClusterIP is safer.

---

## 8. Next Steps / Recommendations

* Enable **Alertmanager** for production alerting.
* Use **Kubernetes Secrets** for Grafana credentials instead of hardcoding.
* Consider keeping **Node Exporter and Kube-State-Metrics as ClusterIP** and rely on Prometheus for scraping internally.
* Import official Grafana dashboards for Kubernetes, Node Exporter, and Prometheus.
* Add ServiceMonitors for your application and Jenkins to monitor custom metrics.

---

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/77f1251e-98fe-4394-b6a2-cf5dc956ab61" />

-------------

# Notification with Slack

## Create Slack Account

1. Go to Slack and create an account.
2. Click **Create a workspace**.
3. Enter a **workspace name**, click **Next**, and skip the trial/free steps.

---

## Complete Jenkins-Slack Integration via Webhook

### **PART 1: Create Slack App and Webhook URL**

#### Step 1: Create Slack App

1. Navigate to [https://api.slack.com/apps](https://api.slack.com/apps)
2. Click **Create New App**
3. Choose **From scratch**
4. Fill in the following details:

   * **App Name:** any name of your choice
   * **Workspace:** your workspace name
5. Click **Create App**

#### Step 2: Enable Webhooks

1. In the left sidebar, click **Incoming Webhooks**
2. Toggle **Activate Incoming Webhooks** to enable

#### Step 3: Add Webhook to a Channel

1. Scroll down and click **Add New Webhook to Workspace**
2. Select a channel, e.g., `#general`
3. Click **Allow**
4. Copy the generated Webhook URL (example below):

   ```
   https://hooks.slack.com/services/T0XXXX/B0YYYY/ZZZZZ
   ```

---

### **PART 2: OAuth Scopes Required (MANDATORY)**

If you plan to use bot integration via OAuth token (not just webhooks), add the following scopes:

| OAuth Scope       | Description                                 |
| ----------------- | ------------------------------------------- |
| chat:write        | Send messages as the bot                    |
| chat:write.public | Send messages to public channels not joined |
| channels:read     | Read public channels                        |
| groups:read       | Read private channels                       |
| users:read        | Read user info (optional for mentions)      |

> These are not all required for webhook-only use but essential for token-based plugins.

To add:

* Go to **OAuth & Permissions** → Scroll to **Scopes**
* Click **Add an OAuth Scope** → Add all scopes above
* Return to the top and click **Reinstall to Workspace**

---

### **PART 3: Configure Jenkins**

1. Install **Slack Notification Plugin** in Jenkins.

2. Go to **Jenkins > Credentials**:

   * Add both **Webhook URL** and **OAuth token** as separate credentials.
   * For OAuth token → choose **Secret Text**, ID = `slack-token`
   * For Webhook URL → choose **Secret Text**, ID = `slack-webhook`

3. Go to **Jenkins > Manage Jenkins > System**:

   * Under **Slack**, configure:

     * Workspace Name
     * Slack Token Credential ID = `slack-token`
     * Channel Name (e.g., `#general`)
     * Check **Custom Slack Bot User**
   * Click **Test Connection** and verify Slack response.

---

### **PART 4: Jenkinsfile Configuration**

Below is a fully working Jenkinsfile for Slack notifications:

```groovy
pipeline {
    agent any

    environment {
        PROJECT_NAME = '🧰 My-App'
        ENVIRONMENT = '🚀 Production'
    }

    stages {
        stage('Compile') {
            steps {
                echo "🏗️ Compiling..."
            }
        }
        stage('Test') {
            steps {
                echo "🧪 Running tests..."
            }
        }
        stage('Build') {
            steps {
                echo "🧪 Building..."
            }
        }
        stage('Security') {
            steps {
                echo "🧪 Security..."
            }
        }
        stage('Deploy') {
            steps {
                echo "🚀 Deploying..."
            }
        }
    }

    post {
        success {
            withCredentials([string(credentialsId: 'slack-webhook', variable: 'SLACK_URL')]) {
                script {
                    def message = """{
                        \"text\": \"*✅ ${PROJECT_NAME} Build Successful!*\",
                        \"attachments\": [
                            {
                                \"color\": \"#36a64f\",
                                \"fields\": [
                                    { \"title\": \"Job\", \"value\": \"${env.JOB_NAME}\", \"short\": true },
                                    { \"title\": \"Build\", \"value\": \"#${env.BUILD_NUMBER}\", \"short\": true },
                                    { \"title\": \"Environment\", \"value\": \"${ENVIRONMENT}\", \"short\": true }
                                ],
                                \"footer\": \"Jenkins CI\",
                                \"footer_icon\": \"https://www.jenkins.io/images/logos/jenkins/jenkins.png\",
                                \"ts\": ${System.currentTimeMillis() / 1000},
                                \"actions\": [
                                    {
                                        \"type\": \"button\",
                                        \"text\": \"View Build\",
                                        \"url\": \"${env.BUILD_URL}\",
                                        \"style\": \"primary\"
                                    }
                                ]
                            }
                        ]
                    }"""
                    sh """curl -X POST -H 'Content-type: application/json' --data '${message}' $SLACK_URL"""
                }
            }
        }

        failure {
            withCredentials([string(credentialsId: 'slack-webhook', variable: 'SLACK_URL')]) {
                script {
                    def message = """{
                        \"text\": \"<!here> *❌ ${PROJECT_NAME} Build Failed!*\",
                        \"attachments\": [
                            {
                                \"color\": \"#FF0000\",
                                \"fields\": [
                                    { \"title\": \"Job\", \"value\": \"${env.JOB_NAME}\", \"short\": true },
                                    { \"title\": \"Build\", \"value\": \"#${env.BUILD_NUMBER}\", \"short\": true },
                                    { \"title\": \"Environment\", \"value\": \"${ENVIRONMENT}\", \"short\": true }
                                ],
                                \"footer\": \"Jenkins CI\",
                                \"footer_icon\": \"https://www.jenkins.io/images/logos/jenkins/jenkins.png\",
                                \"ts\": ${System.currentTimeMillis() / 1000},
                                \"actions\": [
                                    {
                                        \"type\": \"button\",
                                        \"text\": \"View Build Logs\",
                                        \"url\": \"${env.BUILD_URL}\",
                                        \"style\": \"danger\"
                                    }
                                ]
                            }
                        ]
                    }"""
                    sh """curl -X POST -H 'Content-type: application/json' --data '${message}' $SLACK_URL"""
                }
            }
        }

        always {
            echo "🎯 Post-build notification sent"
        }
    }
}
```

---

### **PART 5: Run and Verify**

#### Step 1: Trigger a Jenkins Build

* Run your pipeline manually or via Git webhook.

#### Step 2: Verify Slack Notification

* Check your Slack channel to see messages:

  * ✅ Build Passed
  * ❌ Build Failed

> Congratulations! You’ve successfully configured Jenkins to send Slack build notifications 🚀

<img width="1920" height="1080" alt="Screenshot (259)" src="https://github.com/user-attachments/assets/6f64b8c2-4334-470f-aa6c-c0912ef8fd26" />






