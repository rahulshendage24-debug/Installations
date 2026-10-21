## Step - 1 : Create EKS Management Host in AWS ##

1) Launch new Ubuntu VM using AWS Ec2 ( t2.micro )	  
2) Connect to machine and install kubectl using below commands  
```
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin
kubectl version --short --client
```
3) Install AWS CLI latest version using below commands 
```
sudo apt install unzip
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

4) Install eksctl using below commands
```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```
### Step - 2: Create IAM User & Configure AWS CLI on EC2

#### Part A: Create IAM User & Keys (AWS Console)
1. Go to **IAM** > **Users** > **Create user**.
2. Name: `eks-admin-user` (leave console access unchecked) > **Next**.
3. Select **Attach policies directly** > add `AdministratorAccess` > **Create user**.
4. Open `eks-admin-user` > **Security credentials** tab > **Create access key**.
5. Select **CLI** > acknowledge > **Create access key** > copy both **Access Key** and **Secret Access Key**.

---

#### Part B: Configure EC2 Instance
1. Connect to EC2 and run:
   ```bash
   aws configure

## Step - 3 : Create EKS Cluster using eksctl ## 
**Syntax:** 

eksctl create cluster --name cluster-name  \
--region region-name \
--node-type instance-type \
--nodes-min 2 \
--nodes-max 2 \ 
--zones <AZ-1>,<AZ-2>

## N. Virgina: <br/>
```
eksctl create cluster --name ashokit-cluster4 --region us-east-1 --node-type t2.medium  --zones us-east-1a,us-east-1b
```
## Mumbai: <br/>
```
eksctl create cluster \
  --name my-eks-cluster \
  --region ap-south-1 \
  --version 1.31 \
  --nodegroup-name free-tier-workers \
  --node-type t3.micro \
  --nodes 2 \
  --managed```

## Note: Cluster creation will take 5 to 10 mins of time (we have to wait). After cluster created we can check nodes using below command.

```
 kubectl get nodes  
```

### Note: We should be able to see EKS cluster nodes here. ##

### We are done with our Setup ###


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

## Step - 4 : After your practise, delete Cluster and other resources we have used in AWS Cloud to avoid billing ##

```
eksctl delete cluster --name ashokit-cluster4 --region ap-south-1
```
