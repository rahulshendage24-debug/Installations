# Amazon EKS Installation & Automated Setup Guide

A complete, end-to-end guide and single-file automated script for deploying, configuring, and troubleshooting an Amazon EKS cluster on AWS using `eksctl`, `awscli`, and `kubectl`. This configuration is tailored for `ap-south-1` and includes fixes for common AWS Free Tier instance constraints and VPC CNI (`aws-node`) networking issues.

---

## 📑 Table of Contents

- [Overview & Architecture](#overview--architecture)
- [Single-Run Automation Script](#single-run-automation-script)
- [Step-by-Step Manual Breakdown](#step-by-step-manual-breakdown)
  - [1. CLI Prerequisites Installation](#1-cli-prerequisites-installation)
  - [2. Cluster Provisioning](#2-cluster-provisioning)
  - [3. Kubeconfig Configuration](#3-kubeconfig-configuration)
  - [4. CNI & IAM Permissions Fix](#4-cni--iam-permissions-fix)
  - [5. Verification Workload Deployment](#5-verification-workload-deployment)
- [Common Errors & Troubleshooting](#common-errors--troubleshooting)
  - [Free Tier Restriction (`InvalidParameterCombination`)](#free-tier-restriction-invalidparametercombination)
  - [Nodes Stuck in `NotReady` & CoreDNS `Pending`](#nodes-stuck-in-notready--coredns-pending)
- [Teardown & Cost Management](#teardown--cost-management)

---

## Overview & Architecture

When provisioning an EKS cluster with managed nodegroups, AWS creates two CloudFormation stacks:
1. **Control Plane Stack (`eksctl-<name>-cluster`)**: Provisions the VPC, subnets, internet gateways, NAT gateways, security groups, and the Kubernetes API server control plane.
2. **Nodegroup Stack (`eksctl-<name>-nodegroup-<name>`)**: Provisions the Auto Scaling Group, IAM instance profile, and EC2 worker instances.

```
+-----------------------------------------------------------------------------------+
| AWS Cloud (Region: ap-south-1)                                                   |
|                                                                                   |
|  +-----------------------------------------------------------------------------+  |
|  | VPC (Public & Private Subnets, Route Tables, NAT GW)                        |  |
|  |                                                                             |  |
|  |  +-----------------------+         +-------------------------------------+  |  |
|  |  | EKS Control Plane     | <-----> | Managed Nodegroup (EC2 Instances)   |  |  |
|  |  | (API, etcd, CoreDNS)  |         | - kube-proxy (HostPort)             |  |  |
|  |  +-----------------------+         | - aws-node (VPC CNI Plugin)         |  |  |
|  |                                    | - AmazonEKS_CNI_Policy              |  |  |
|  |                                    +-------------------------------------+  |  |
|  +-----------------------------------------------------------------------------+  |
+-----------------------------------------------------------------------------------+
```

---

## Single-Run Automation Script

Save this script as `deploy-eks.sh`, grant execution permissions, and run it directly on your management host (such as an EC2 bastion host or local terminal with configured AWS credentials):

```bash
cat << 'EOF' > deploy-eks.sh
#!/usr/bin/env bash
set -eo pipefail

# ==============================================================================
# Amazon EKS End-to-End Setup & Auto-Troubleshooting Script
# Region: ap-south-1 | Compatible with Free-Tier accounts & handles VPC CNI issues
# ==============================================================================

CLUSTER_NAME="my-eks-cluster"
REGION="ap-south-1"
NODEGROUP_NAME="free-tier-workers"
INSTANCE_TYPE="t3.micro"  # Use t3.micro for Free Tier; use t3.medium for standard

echo "=============================================================================="
echo " [1/6] Installing CLI Prerequisites (AWS CLI, kubectl, eksctl)..."
echo "=============================================================================="

# 1. AWS CLI
if ! command -v aws &> /dev/null; then
  echo "--> Installing AWS CLI v2..."
  curl -s "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
  unzip -q awscliv2.zip && sudo ./aws/install && rm -rf awscliv2.zip aws
fi

# 2. kubectl
if ! command -v kubectl &> /dev/null; then
  echo "--> Installing kubectl..."
  curl -sLO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
  sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl && rm -f kubectl
fi

# 3. eksctl
if ! command -v eksctl &> /dev/null; then
  echo "--> Installing eksctl..."
  ARCH=amd64
  PLATFORM=$(uname -s)_$ARCH
  curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"
  tar -xzf "eksctl_$PLATFORM.tar.gz" -C /tmp && rm -f "eksctl_$PLATFORM.tar.gz"
  sudo mv /tmp/eksctl /usr/local/bin/eksctl
fi

echo "--> CLI Versions:"
aws --version
kubectl version --client --output=yaml | grep gitVersion || true
eksctl version

echo "=============================================================================="
echo " [2/6] Provisioning EKS Cluster via eksctl..."
echo "=============================================================================="

if ! aws eks describe-cluster --name "$CLUSTER_NAME" --region "$REGION" &> /dev/null; then
  echo "--> Creating cluster $CLUSTER_NAME in $REGION (approx. 12-15 minutes)..."
  eksctl create cluster \
    --name "$CLUSTER_NAME" \
    --region "$REGION" \
    --version 1.30 \
    --nodegroup-name "$NODEGROUP_NAME" \
    --node-type "$INSTANCE_TYPE" \
    --nodes 2 \
    --nodes-min 1 \
    --nodes-max 2 \
    --managed
else
  echo "--> Cluster $CLUSTER_NAME already exists. Skipping control plane creation."
fi

echo "=============================================================================="
echo " [3/6] Configuring local kubeconfig..."
echo "=============================================================================="
aws eks update-kubeconfig --region "$REGION" --name "$CLUSTER_NAME"

echo "=============================================================================="
echo " [4/6] Applying VPC CNI IAM Policy & Networking Fix..."
echo "=============================================================================="
echo "--> Resolving Node Instance IAM Role from CloudFormation..."
STACK_NAME=$(aws cloudformation list-stacks --region "$REGION" \
  --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE \
  --query "StackSummaries[?starts_with(StackName, 'eksctl-$CLUSTER_NAME-nodegroup')].StackName" \
  --output text | awk '{print $1}')

if [ -n "$STACK_NAME" ]; then
  NODE_ROLE=$(aws cloudformation describe-stack-resources \
    --stack-name "$STACK_NAME" \
    --region "$REGION" \
    --query 'StackResources[?LogicalResourceId==`NodeInstanceRole`].PhysicalResourceId' \
    --output text)

  if [ -n "$NODE_ROLE" ]; then
    echo "--> Attaching AmazonEKS_CNI_Policy to IAM Role: $NODE_ROLE"
    aws iam attach-role-policy \
      --role-name "$NODE_ROLE" \
      --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy || true
  fi
fi

echo "--> Applying AWS VPC CNI daemonset manifest..."
kubectl apply -f https://raw.githubusercontent.com/aws/amazon-vpc-cni-k8s/master/config/master/aws-k8s-cni.yaml

echo "--> Waiting up to 180 seconds for nodes to transition to Ready..."
kubectl wait --for=condition=Ready node --all --timeout=180s || true

echo "=============================================================================="
echo " [5/6] Deploying Verification Test Workload (Nginx)..."
echo "=============================================================================="
kubectl create deployment test-nginx --image=nginx:alpine --replicas=2 --dry-run=client -o yaml | kubectl apply -f -
kubectl expose deployment test-nginx --port=80 --type=NodePort --dry-run=client -o yaml | kubectl apply -f -

echo "=============================================================================="
echo " [6/6] Final Cluster Status Verification..."
echo "=============================================================================="
echo "--- Nodes ---"
kubectl get nodes -o wide

echo "--- kube-system Pods ---"
kubectl get pods -n kube-system

echo "--- Application Workloads ---"
kubectl get pods,svc -l app=test-nginx

echo "=============================================================================="
echo "Cluster deployment successful!"
echo "To destroy the cluster later and stop billing, run:"
echo "  eksctl delete cluster --name $CLUSTER_NAME --region $REGION"
echo "=============================================================================="
EOF

chmod +x deploy-eks.sh
./deploy-eks.sh
```

---

## Step-by-Step Manual Breakdown

### 1. CLI Prerequisites Installation

Before initiating any AWS resources, ensure your client environment has the required CLI binaries installed:

```bash
# AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip -q awscliv2.zip
sudo ./aws/install
rm -rf awscliv2.zip aws

# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm kubectl

# eksctl
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH
curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"
tar -xzf "eksctl_$PLATFORM.tar.gz" -C /tmp && rm "eksctl_$PLATFORM.tar.gz"
sudo mv /tmp/eksctl /usr/local/bin/eksctl
```

Verify your authentication context:
```bash
aws sts get-caller-identity
```

---

### 2. Cluster Provisioning

Run the cluster creation command. In regions like `ap-south-1` under AWS Free Tier limitations, specify `t3.micro`:

```bash
eksctl create cluster \
  --name my-eks-cluster \
  --region ap-south-1 \
  --version 1.30 \
  --nodegroup-name free-tier-workers \
  --node-type t3.micro \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 2 \
  --managed
```

---

### 3. Kubeconfig Configuration

Point your local `kubectl` context to the newly created EKS API endpoint:

```bash
aws eks update-kubeconfig --region ap-south-1 --name my-eks-cluster
```

Verify reachability:
```bash
kubectl cluster-info
```

---

### 4. CNI & IAM Permissions Fix

If your nodes show `NotReady` due to uninitialized pod networking:

1. **Locate the Node Instance Role:**
   ```bash
   NODE_ROLE=$(aws cloudformation describe-stack-resources \
     --stack-name eksctl-my-eks-cluster-nodegroup-free-tier-workers \
     --region ap-south-1 \
     --query 'StackResources[?LogicalResourceId==`NodeInstanceRole`].PhysicalResourceId' \
     --output text)
   ```

2. **Attach the AWS CNI Policy:**
   ```bash
   aws iam attach-role-policy \
     --role-name "$NODE_ROLE" \
     --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
   ```

3. **Deploy the VPC CNI Plugin:**
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/aws/amazon-vpc-cni-k8s/master/config/master/aws-k8s-cni.yaml
   ```

4. **Verify the DaemonSet:**
   ```bash
   kubectl get daemonset -n kube-system aws-node
   ```

---

### 5. Verification Workload Deployment

Deploy a lightweight test container to verify pod scheduling, CNI IP assignment, and node port routing:

```bash
# Deploy Nginx
kubectl create deployment test-nginx --image=nginx:alpine --replicas=2

# Expose service
kubectl expose deployment test-nginx --port=80 --type=NodePort

# Check status
kubectl get pods -o wide
kubectl get svc test-nginx
```

---

## Common Errors & Troubleshooting

### Free Tier Restriction (`InvalidParameterCombination`)

* **Symptom:** The nodegroup stack hangs in `CREATE_IN_PROGRESS` for 20 minutes before rolling back.
* **Error Output:**
  ```text
  Could not launch On-Demand Instances. InvalidParameterCombination - 
  The specified instance type is not eligible for Free Tier.
  ```
* **Root Cause:** The default instance type `t3.medium` is blocked on accounts subject to Free Tier enforcement policies.
* **Fix:** Delete the failed nodegroup and redeploy with `t3.micro`:
  ```bash
  eksctl delete nodegroup --cluster my-eks-cluster --name standard-workers --region ap-south-1
  eksctl create nodegroup --cluster my-eks-cluster --region ap-south-1 --name free-tier-workers --node-type t3.micro --nodes 2 --managed
  ```

---

### Nodes Stuck in `NotReady` & CoreDNS `Pending`

* **Symptom:** `kubectl get nodes` displays `NotReady`, and `kubectl get pods -n kube-system` shows `coredns` stuck in `Pending` with no `aws-node` pods.
* **Root Cause:** The AWS VPC CNI DaemonSet (`aws-node`) is either missing from the cluster or lacks IAM permissions (`AmazonEKS_CNI_Policy`) on the EC2 role, preventing container network interfaces (ENIs) from attaching.
* **Fix:** Follow the steps in [Section 4](#4-cni--iam-permissions-fix) to attach `AmazonEKS_CNI_Policy` and reapply `aws-k8s-cni.yaml`.

---

## Teardown & Cost Management

EKS cluster control planes incur an ongoing AWS cost (~$0.10/hour). When you finish your testing, clean up all provisioned resources:

```bash
eksctl delete cluster --name my-eks-cluster --region ap-south-1
```

Confirm all stacks have been removed from CloudFormation:
```bash
aws cloudformation list-stacks \
  --region ap-south-1 \
  --stack-status-filter DELETE_IN_PROGRESS CREATE_COMPLETE \
  --query "StackSummaries[?starts_with(StackName, 'eksctl-my-eks-cluster')].StackName" \
  --output table
```