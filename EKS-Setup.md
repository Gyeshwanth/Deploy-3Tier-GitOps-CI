# 📘 DevOps Tooling & EKS Setup Guide on Linux

This guide provides a comprehensive step-by-step process to install DevOps tools on Linux, set up an EKS cluster, configure IAM roles for service accounts, and deploy essential Kubernetes add-ons (EBS CSI Driver, NGINX Ingress Controller, cert-manager). The guide is fully corrected, structured, and formatted for clarity.

---

## 🖥️ EC2 Instance Requirements

* Instance Type: **t2.medium**


---

## 1️⃣ Install AWS CLI

AWS CLI is used to interact with AWS services from the terminal.

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip -y
unzip awscliv2.zip
sudo ./aws/install
aws --version
aws configure  # provide AWS Access Key, Secret Key, region, and output format
```

---
# 🔑 How to Generate AWS Access Key and Secret Access Key

This guide walks you through creating AWS Access Keys for CLI, SDK, or Terraform usage.

---

## 1️⃣ Sign in to AWS Management Console
- Go to [https://console.aws.amazon.com/](https://console.aws.amazon.com/)
- Log in using your **IAM user credentials** (do **not** use root account).

---

## 2️⃣ Open IAM (Identity and Access Management)
- In the AWS search bar, type **IAM** → click **IAM** service.

---

## 3️⃣ Select Your IAM User
- From the left menu → **Users**
- Click the **username** for which you want to create keys.

---

## 4️⃣ Create Access Keys
- Go to the **Security credentials** tab.  
- Scroll to **Access keys** section.  
- Click **Create access key**.

---

## 5️⃣ Choose Key Type
- Select **Application running outside AWS** (for CLI, SDK, Terraform).  
- Click **Next**, then **Create access key**.

---

## 6️⃣ Copy and Save Keys
After creation, AWS will show:  
- **Access Key ID**  
- **Secret Access Key**

> ⚠️ Important: Secret key is **shown only once**. Save it securely (password manager or `~/.aws/credentials`).

---

## 2️⃣ Install kubectl

`kubectl` is the Kubernetes command-line tool to manage clusters.

```bash
curl -LO "https://dl.k8s.io/release/$(curl -sL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
kubectl version --client --short
```

---

## 3️⃣ Install eksctl

`eksctl` is used to create and manage EKS clusters.

```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```

---

## 4️⃣ Install Terraform

Terraform is used to provision AWS infrastructure as code.

```bash
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common curl
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt-get update && sudo apt-get install terraform -y
terraform --version
```

---
##  Running Terraform

1. Clone the repository:

```bash
git clone https://github.com/Gyeshwanth/Terraform-devops-project.git
cd Terraform-devops-project
```

2. Update `variables.tf` with your AWS key-pair and other configuration.
3. Update `main.tf` to set the correct region and settings.
4. Initialize and apply Terraform:

**Purpose:** Creates EKS cluster, VPC, and required AWS resources automatically.

```bash
terraform init
terraform plan
terraform apply --auto-approve

terraform destroy
```

# Terraform Destroy Guide

## Purpose

* Deletes all AWS resources (EC2, EKS, VPCs, S3, IAM roles, etc.) created by Terraform.
* Cleans up the infrastructure to avoid ongoing costs.
* Useful in development, testing, or when you want to completely remove a deployment.
* **Single-command simplicity:** Removing all resources manually from AWS can be complex and error-prone due to dependencies between services; `terraform destroy` handles this automatically in the correct order.



### Optional Flag

```bash
terraform destroy --auto-approve
```

* Skips the confirmation prompt and removes all resources immediately.

## Notes

* **Destructive:** Permanently deletes resources.
* **Production Caution:** Only use if you intend to remove all live infrastructure.
* **State File Dependency:** Terraform uses its state file to know which resources to delete. If the state is missing or corrupted, the command may fail or behave unpredictably.



---



## 5️⃣ Configure kubeconfig for EKS

Connect `kubectl` to your EKS cluster.

```bash
aws eks --region <region> update-kubeconfig --name <cluster-name>
```

---

## 6️⃣ Associate IAM OIDC Provider

Allows Kubernetes pods to assume AWS IAM roles.

```bash
eksctl utils associate-iam-oidc-provider \
--region <region> \
--cluster <cluster-name> \
--approve
```

**Purpose:** Required when pods need to access AWS services like S3, DynamoDB, or EBS.

---

## 7️⃣ Create IAM Service Account for EBS CSI Driver

```bash
eksctl create iamserviceaccount \
--region <region> \
--name ebs-csi-controller-sa \
--namespace kube-system \
--cluster <cluster-name> \
--attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
--approve \
--override-existing-serviceaccounts
```

**Purpose:** Grants EKS permissions to dynamically provision EBS volumes for pods when using storage that is managed by Kubernetes.

**Use Case:**

* **Dynamic storage required:** If your application uses Kubernetes-managed persistent volumes (PV) for storing data, this setup is required.
* **Not needed:** If the application stores data in a fully managed cloud service (e.g., RDS, DynamoDB) and does not require direct block storage, EBS CSI is not required.

**Flow for Dynamic Provisioning:**

```
PersistentVolume (PV) > PersistentVolumeClaim (PVC) > StorageClass (SC) > CSI Driver > AWS EBS Volume
```

---

## 8️⃣ Deploy Kubernetes Add-ons

### a) EBS CSI Driver (Storage)

```bash
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/ecr/?ref=release-1.11"
```

**Purpose:** Dynamic provisioning of persistent volumes for pods using AWS EBS.

### b) NGINX Ingress Controller (Traffic Management)

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
```

**Purpose:** Handles external HTTP/HTTPS traffic and routes it to services.

### c) cert-manager (SSL Certificates)

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.12.0/cert-manager.yaml
```

**Purpose:** Automates issuance and renewal of SSL/TLS certificates for secure HTTPS traffic.

---


## 🔗 Connecting Pods to AWS Services

Pods can access AWS services using IAM roles associated with service accounts.

```bash
eksctl utils associate-iam-oidc-provider --cluster <your-cluster-name> --approve
```

**Use case:** Pods can securely communicate with S3, EBS, DynamoDB without embedding credentials.

---

### ✅ Summary of Add-ons & Purpose

| Add-on         | Purpose                              | Required When                                                                                  |
| -------------- | ------------------------------------ | ---------------------------------------------------------------------------------------------- |
| EBS CSI Driver | Dynamic volume provisioning for pods | Pods need persistent storage on AWS EBS; not required for fully managed cloud storage services |
| NGINX Ingress  | HTTP/HTTPS traffic routing           | External access to services                                                                    |
| cert-manager   | SSL/TLS automation                   | HTTPS security for services                                                                    |
| IAM OIDC       | Associate IAM roles with pods        | Pods access AWS services                                                                       |

---

### ⚠️ Note About EBS CSI Driver During Terraform Apply

When running:

```bash
terraform apply --auto-approve
```

* You might encounter errors while deploying the **EBS CSI Driver** add-on automatically.
* This is **expected for the very first EKS cluster** because the IAM service account for the CSI driver may not exist yet or the OIDC provider association might not be ready.

**Recommended Steps:**

1. **Remove any automated EBS CSI Driver creation code** from your Terraform configuration temporarily.
2. **Run `terraform apply`** to create the cluster and core resources first.
3. **Manually create the IAM service account** for EBS CSI Driver as per this guide:

```bash
eksctl create iamserviceaccount \
--region <region> \
--name ebs-csi-controller-sa \
--namespace kube-system \
--cluster <cluster-name> \
--attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
--approve \
--override-existing-serviceaccounts
```

4. **Then deploy the EBS CSI Driver** manually:

```bash
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/ecr/?ref=release-1.11"
```

5. After this, you can **re-add or update your Terraform code** for the CSI driver if you want to automate it for future clusters.

> ✅ Summary: First create the cluster, then set up the IAM service account for EBS CSI, then deploy the CSI driver. This prevents Terraform apply from failing due to missing IAM roles or OIDC association.

---

### 💡 Alternate AWS Console Method

1. Go to **AWS Console → EKS → Cluster → Add-ons**.
2. Select **Amazon EBS CSI Driver **, and delete this.
3. If you see existing IAM service account conflicts, go to **CloudFormation**, delete the existing `ebs-csi-controller-sa` stack.
4. Then run:

```bash
eksctl create iamserviceaccount \
--region ap-south-1 \
--name ebs-csi-controller-sa \
--namespace kube-system \
--cluster yesh-cluster \
--attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
--approve \
--override-existing-serviceaccounts
```

5. Finally, deploy with:

```bash
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/ecr/?ref=release-1.11"
```

Then re-run Terraform:

```bash
terraform init
terraform apply --auto-approve
```
