# Amazon EKS Deployment from Scratch (via EC2 Bastion)

Step-by-step guide to provisioning an EC2 management host, installing essential Kubernetes tools, creating an EKS cluster with `eksctl`, fixing CNI permissions, and verifying cluster health.

---

## Step 1: Launch Management EC2 Instance (Bastion Host)

Run this via your local machine, AWS CLI, or launch directly in the AWS Console:

```bash
# 1. Create a key pair (if you don't have one)
aws ec2 create-key-pair \
  --key-name eks-bastion-key \
  --region ap-south-1 \
  --query 'KeyMaterial' \
  --output text > eks-bastion-key.pem

chmod 400 eks-bastion-key.pem

# 2. Get latest Amazon Linux 2023 AMI ID
AMI_ID=$(aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=al2023-ami-2023.*-x86_64" "Name=state,Values=available" \
  --region ap-south-1 \
  --query 'sort_by(Images, &CreationDate)[-1].ImageId' \
  --output text)

# 3. Launch EC2 Instance (t3.micro / t2.micro)
INSTANCE_ID=$(aws ec2 run-instances \
  --image-id "$AMI_ID" \
  --instance-type t3.micro \
  --key-name eks-bastion-key \
  --region ap-south-1 \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=eks-management-host}]' \
  --query 'Instances[0].InstanceId' \
  --output text)

echo "Launched Instance ID: $INSTANCE_ID"

# 4. Get Public IP
PUBLIC_IP=$(aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --region ap-south-1 \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text)

echo "SSH into your instance:"
echo "ssh -i eks-bastion-key.pem ec2-user@$PUBLIC_IP"
```

---

## Step 2: Install CLI Tools (`aws`, `kubectl`, `eksctl`)

SSH into your EC2 instance and run:

```bash
# Update packages and install prerequisites
sudo yum update -y
sudo yum install -y unzip curl tar git

# 1. Install AWS CLI v2
curl -s "[https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip](https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip)" -o "awscliv2.zip"
unzip -q awscliv2.zip
sudo ./aws/install
rm -rf awscliv2.zip aws

# 2. Install kubectl
curl -sLO "[https://dl.k8s.io/release/$(curl](https://dl.k8s.io/release/$(curl) -L -s [https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl](https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl)"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm -f kubectl

# 3. Install eksctl
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH
curl -sLO "[https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz](https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz)"
tar -xzf "eksctl_$PLATFORM.tar.gz" -C /tmp && rm -f "eksctl_$PLATFORM.tar.gz"
sudo mv /tmp/eksctl /usr/local/bin/eksctl

# Verify installations
aws --version
kubectl version --client
eksctl version
```

---

## Step 3: Configure AWS Credentials

Ensure your machine is authenticated with your AWS account:

```bash
aws configure
```

*(Enter your `AWS Access Key ID`, `AWS Secret Access Key`, default region: `ap-south-1`, and output format: `json`)*

Verify your identity:

```bash
aws sts get-caller-identity
```

---

## Step 4: Provision the EKS Cluster

Create the cluster using `t3.micro` worker nodes to comply with Free Tier restrictions:

```bash
eksctl create cluster \
  --name my-eks-cluster \
  --region ap-south-1 \
  --version 1.30 \
  --nodegroup-name free-tier-workers \
  --node-type t3.micro \
  --nodes 2 \
  --managed
```

> **Note:** Cluster provisioning typically takes 12–15 minutes.

---

## Step 5: Update Local Kubeconfig

Configure `kubectl` to connect to your newly created EKS cluster:

```bash
aws eks update-kubeconfig --region ap-south-1 --name my-eks-cluster
```

Verify connection:

```bash
kubectl config current-context
```

---

## Step 6: Fix CNI Permissions & Enable Pod Networking

### A. Find the Node IAM Role Name

```bash
aws cloudformation describe-stack-resources \
  --stack-name eksctl-my-eks-cluster-nodegroup-free-tier-workers \
  --region ap-south-1 \
  --query 'StackResources[?LogicalResourceId==`NodeInstanceRole`].PhysicalResourceId' \
  --output text
```

*(Copy the role name displayed on your terminal, e.g., `eksctl-my-eks-cluster-nodegroup-NodeInstanceRole-XXXXX`)*

### B. Attach the CNI Policy

Replace `<COPIED_ROLE_NAME>` with the role name obtained from the previous command:

```bash
aws iam attach-role-policy \
  --role-name <COPIED_ROLE_NAME> \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
```

### C. Deploy the VPC CNI Manifest

```bash
kubectl apply -f [https://raw.githubusercontent.com/aws/amazon-vpc-cni-k8s/master/config/master/aws-k8s-cni.yaml](https://raw.githubusercontent.com/aws/amazon-vpc-cni-k8s/master/config/master/aws-k8s-cni.yaml)
```

---

## Step 7: Verify Cluster & Nodes

Wait 30–60 seconds, then inspect the nodes and system pods:

```bash
# Check worker nodes (both should display Ready)
kubectl get nodes -o wide

# Check core system pods (CoreDNS, kube-proxy, aws-node should display Running)
kubectl get pods -n kube-system
```

---

## Step 8: Teardown (Clean Up Resources)

When you are done practicing, delete the cluster to prevent ongoing AWS billing:

```bash
eksctl delete cluster --name my-eks-cluster --region ap-south-1
```
