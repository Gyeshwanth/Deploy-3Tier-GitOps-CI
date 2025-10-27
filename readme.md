
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
* **ArgoCD** – GitOps-based continuous deployment for Kubernetes
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
or 

```bash
sudo apt install gitleaks
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
        SCANNER_HOME   = tool 'sonar-scanner'
        IMAGE_TAG      = "1.0.${BUILD_NUMBER}"
        FRONTEND_IMAGE = "yeshwanthgosi/frontend"
        BACKEND_IMAGE  = "yeshwanthgosi/backend"
        CD_REPO_URL    = "https://github.com/Gyeshwanth/Deploy-3Tier-GitOps-CD.git"
      GIT_USER_NAME  = "Gyeshwanth"
        GIT_REPO_NAME  = "Deploy-3Tier-GitOps-CD"
	}

    stages {
        stage('git checkout') {
            steps {
                git branch: 'main',
                    credentialsId: 'git-cred',
                    url: 'https://github.com/Gyeshwanth/Deploy-3Tier-GitOps-CI.git'
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
                    sh ''' $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=NodeJS-Project \
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

       
        stage('Build, Tag & Push Backend Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        dir('api') {
                            sh """
                                echo " Building backend image..."
                                docker build -t ${BACKEND_IMAGE}:latest -t ${BACKEND_IMAGE}:${IMAGE_TAG} .
                                echo " Scanning image with Trivy..."
                                trivy image --format table -o backend-image-report.html ${BACKEND_IMAGE}:${IMAGE_TAG} || true
                                echo " Pushing images..."
                                docker push ${BACKEND_IMAGE}:latest
                                docker push ${BACKEND_IMAGE}:${IMAGE_TAG}
                            """
                        }
                    }
                }
            }
        }

        stage('Build, Tag & Push Frontend Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        dir('client') {
                            sh """
                                echo " Building frontend image..."
                                docker build -t ${FRONTEND_IMAGE}:latest -t ${FRONTEND_IMAGE}:${IMAGE_TAG} .
                                echo " Scanning image with Trivy..."
                                trivy image --format table -o frontend-image-report.html ${FRONTEND_IMAGE}:${IMAGE_TAG} || true
                                echo " Pushing images..."
                                docker push ${FRONTEND_IMAGE}:latest
                                docker push ${FRONTEND_IMAGE}:${IMAGE_TAG}
                            """
                        }
                    }
                }
            }
        }

       stage('Update CD Repo with New Image Tags') {
    steps {
        script {
            withCredentials([usernamePassword(credentialsId: 'git-cred', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
                sh '''
                    echo "🌀 Cloning CD repo..."
                    rm -rf cd
                    git clone https://$GIT_USERNAME:$GIT_PASSWORD@github.com/$GIT_USER_NAME/$GIT_REPO_NAME.git cd
                    cd cd
                    git checkout main || git checkout -b main

                    echo " Installing yq..."
                    curl -sL https://github.com/mikefarah/yq/releases/download/v4.44.3/yq_linux_amd64 -o yq
                    chmod +x yq
                    mv yq /tmp/yq
                    export PATH=$PATH:/tmp

                    echo " Updating image tags..."
                    echo "📝 Updating backend image tag..."
                    yq -i '
                      (. | select(.kind=="Deployment")
                         | .spec.template.spec.containers[]
                         | select(.name=="backend")
                         | .image)
                      = "yeshwanthgosi/backend:" + strenv(IMAGE_TAG)
                    ' k8s-prod/backend.yaml

                    yq -i '
                      (. | select(.kind=="Deployment")
                         | .spec.template.spec.containers[]
                         | select(.name=="frontend")
                         | .image)
                      = "yeshwanthgosi/frontend:" + strenv(IMAGE_TAG)
                    ' k8s-prod/frontend.yaml

                    git config user.email "yeshwanth@example.com"
                    git config user.name "yeshwanth"
                    git add -A
                    git commit -m "Update image tags to ${IMAGE_TAG}" || echo "No changes to commit."
                    git pull --rebase origin main || true

                    echo " Pushing updated files..."
                    git push https://$GIT_USERNAME:$GIT_PASSWORD@github.com/$GIT_USER_NAME/$GIT_REPO_NAME.git HEAD:main || echo "⚠️ Push failed or no new changes."
                '''
            }
        }
    }
}


    }

    post {
        success {
            echo "✅ CI + CD pipeline completed successfully. Images updated to tag: ${IMAGE_TAG}"
        }
        failure {
            echo "❌ Pipeline failed. Check Jenkins logs for details."
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

This document outlines the steps to set up an EC2 instance as an EKS Master Node to run `kubectl` commands and integrate with Jenkins pipelines for Kubernetes automation.

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


---

# ArgoCD Setup

### Install ArgoCD on EKS Cluster

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

**Verify Installation:**

```bash
kubectl get all -n argocd
```

---

### Expose ArgoCD

By default, ArgoCD server is internal(CluserIP) . Expose via LoadBalancer:

```bash
kubectl edit svc argocd-server -n argocd
```

Change service type → `LoadBalancer`

**Access UI:** `http://<PUBLIC_IP:NodePort>`

---

### Get ArgoCD Admin Password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

Login → Username: `admin`

Update password → via ArgoCD UI under **User Info**

---

### Connect GitHub Repo to ArgoCD

1. Go to **Settings → Repositories → Connect Repo**
2. Choose connection type **HTTPS**
3. Fill:

   * Repository URL
   * Username
   * Password or GitHub Token

---

### Create Application in ArgoCD

* Provide repo URL
* Specify path (e.g., `k8s-prod/`)
* Set **Cluster URL** and **Namespace (prod)**
* Enable checkboxes:

  * Auto-create namespace
  * Auto-sync

---

### Setup GitHub Webhook for Auto-Sync

```bash
SECRET="yeshwanth7896"
kubectl -n argocd patch secret argocd-secret \
  --type merge \
  -p "{\"stringData\": {\"webhook.github.secret\": \"${SECRET}\"}}"
```

Verify webhook:

```bash
kubectl -n argocd get secret argocd-secret -o yaml | grep webhook.github.secret
```


----
##  Configure GitHub Webhook

1. Go to **GitHub Repository** → **Settings** → **Webhooks**.
2. Click **Add webhook**.
3. Set **Payload URL** <ARGOCD_URL>/api/webhook
4. Set **Content type** to `application/json`.
5.  **Secret:** yeshwanth7896
6. Select the specific events (e.g., **Push**) to trigger the pipeline.
7. Verify webhook by checking **Recent Deliveries**.

This enables ArgoCD to **auto-sync** whenever a GitHub commit occurs.

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
---

## 🧾 Troubleshoot Related Certificate In  Kubernetes (TLS)

If your certificate is not created or has issues(After sometime also still Ready False), you can manually delete it to trigger automatic recreation by **cert-manager**:

```bash
kubectl delete certificate yeshwanth-co-tls -n prod
```

After deletion, wait a few minutes — cert-manager will reissue it automatically.

Verify the status:

```bash
kubectl get certificate yeshwanth-co-tls -n prod
```

If `READY` is `True`, your certificate is successfully recreated.

---


* Verify HTTPS access in browser for secure endpoints.


---
## 🔒 Summary

* **CI** handled by Jenkins
* **CD** handled by ArgoCD (GitOps model)
* **Auto-sync** triggered via GitHub webhook
* **Secure & automated** deployment pipeline

<img width="1920" height="1080" alt="Screenshot (261)" src="https://github.com/user-attachments/assets/c83bb112-aceb-4b0d-bd33-c131bc0079f8" />

<img width="1920" height="1080" alt="Screenshot (262)" src="https://github.com/user-attachments/assets/63373c30-741e-422e-b332-834725610b97" />


