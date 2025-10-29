# Kubernetes Deployment for Gogs using Helm and Terraform

## Problem Statement

Build a production-grade, self-hosted Git service using Gogs on Amazon EKS, ensuring:

- Terraform provisions the entire AWS infrastructure — including VPC, IAM, and an EKS cluster with node groups.
- Helm deploys Gogs with:
  - PostgreSQL backend
  - Persistent storage via EBS
  - Ingress controller for external access
- The deployment must be:
  - Fully reproducible
  - Secure and scalable
  - Require minimal manual intervention

### Goal

Deliver an automated, version-controlled deployment pipeline that spins up a complete Git hosting platform in AWS using one command for infrastructure and one for application deployment.

---

## Prerequisites & Setup

### 1. DevOps Control Server

Ensure you have a dedicated server (local or EC2) for running Terraform and Helm commands.

### 2. Install AWS CLI

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```
### 3. Configure AWS Credentials
```bash
export AWS_ACCESS_KEY_ID=<your-access-key-id>
export AWS_SECRET_ACCESS_KEY=<your-secret-access-key>
aws eks --region us-east-1 update-kubeconfig --name gogs-prod-cluster
```
### 3. Install kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

## AWS Infrastructure

### Create Terraform Backend
```bash
aws s3api create-bucket --bucket gogs-terraform --region us-east-1

aws s3api put-bucket-versioning \
  --bucket gogs-terraform \
  --region us-east-1 \
  --versioning-configuration Status=Enabled

aws dynamodb create-table \
  --table-name terraform-locks \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST

```
### Terraform Installation & Usage
```bash
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
sudo apt-add-repository \
  "deb [arch=$(dpkg --print-architecture)] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
sudo apt update && sudo apt install terraform

```
### Terraform Workflow
```bash
terraform init
terraform validate
terraform plan
terraform apply
terraform output
```
## Gogs Kubernetes Deployment

### 1. Create Namespace
```bash
kubectl create namespace gogs
```
### 2. EBS CSI Driver
```bash
aws iam attach-role-policy \
  --role-name <your-node-group-role-name> \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy

aws eks create-addon \
  --cluster-name gogs-prod-cluster \
  --addon-name aws-ebs-csi-driver \
  --region us-east-1

aws eks describe-addon \
  --cluster-name gogs-prod-cluster \
  --addon-name aws-ebs-csi-driver \
  --region us-east-1
```

**For instructions to enable service mesh observability, check out the service-mesh folder.**

