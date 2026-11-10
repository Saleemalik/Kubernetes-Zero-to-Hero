# Kubernetes Zero to Hero

Creating this repository with the intention of making Kubernetes easy for beginners. This is a work in progress (WIP) repository containing practical examples, installation guides, architecture notes, and hands-on Kubernetes exercises.

# Kubernetes Installation Using KOPS on AWS EC2

## 1. Introduction

### What is Kubernetes?

Kubernetes is an open-source container orchestration platform used to deploy, manage, scale, and maintain containerized applications.

### What is KOPS?

KOPS (Kubernetes Operations) is a tool used to create, configure, update, and manage Kubernetes clusters.

KOPS automates the creation of AWS infrastructure and Kubernetes components.

### Why do we need an S3 bucket?

KOPS requires a persistent state store to save and retrieve cluster configuration and state.

Amazon S3 is used as the state store in this installation.

The S3 bucket stores:
- Kubernetes cluster configuration.
- Cluster state.
- Certificates and private keys.
- Etcd backup data and related metadata.

**Important:** S3 stores the KOPS state. It does not run Kubernetes or host application containers.

### Kubernetes cluster architecture

A Kubernetes cluster consists of two main components:

1. **Control Plane:** Manages and controls the Kubernetes cluster.
2. **Worker Nodes:** Run application workloads inside Pods.

```text
                  KOPS
                   |
          +--------+--------+
          |                 |
          v                 v
       Amazon S3         AWS APIs
     (State Store)          |
                            v
                    AWS Infrastructure
                            |
                    Kubernetes Cluster
                            |
                +-----------+-----------+
                |                       |
                v                       v
           Control Plane            Worker Node
                |                       |
          API Server                  Kubelet
          Scheduler                   Kube-proxy
          Controller                  Container Runtime
          Manager                        |
          etcd                           v
                                       Pods
                                         |
                                     Containers
```

## 2. Prerequisites

You can install and manage KOPS from:
- An Ubuntu EC2 instance.
- Your personal Ubuntu/Linux machine.

### Required tools

1. AWS CLI v2
2. kubectl
3. KOPS
4. curl
5. unzip
6. GPG

Python3 is not required for these three CLI tools.

## 3. Install Dependencies

### Step 1: Update Ubuntu

```bash
sudo apt update
sudo apt install -y curl unzip ca-certificates gnupg
```

### Step 2: Install AWS CLI v2

Download the official AWS CLI v2 installer:

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
```

Extract the downloaded file:

```bash
unzip awscliv2.zip
```

Install:

```bash
sudo ./aws/install
```

Verify:

```bash
aws --version
which aws
```

Expected version format:

```text
aws-cli/2.x.x Python/3.x.x Linux/...
```

Note: These commands use the Linux x86_64 installer. Use the ARM64 installer for ARM-based machines.

### Step 3: Install kubectl

kubectl is the command-line tool used to communicate with and manage Kubernetes clusters.

Create the APT keyring directory:

```bash
sudo mkdir -p -m 755 /etc/apt/keyrings
```

Choose a Kubernetes minor version supported by your KOPS release. The following uses v1.36 as an example:

```bash
K8S_MINOR=v1.36
```

Download the repository signing key:

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/${K8S_MINOR}/deb/Release.key | \
sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

Set permissions:

```bash
sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

Add the Kubernetes repository:

```bash
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/${K8S_MINOR}/deb/ /" | \
sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Update the package index:

```bash
sudo apt update
```

Install kubectl:

```bash
sudo apt install -y kubectl
```

Verify:

```bash
kubectl version --client
```

**Note:** Keep kubectl within one minor version of the Kubernetes API server. Check KOPS and Kubernetes version compatibility before choosing a version.

### Step 4: Install KOPS

Download the latest stable KOPS Linux AMD64 binary:

```bash
curl -Lo kops https://github.com/kubernetes/kops/releases/download/$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)/kops-linux-amd64
```

Make it executable:

```bash
chmod +x kops
```

Move it into the system executable directory:

```bash
sudo mv kops /usr/local/bin/kops
```

Verify:

```bash
kops version
```

## 4. AWS IAM Permissions

KOPS requires AWS permissions to create and manage infrastructure.

Create a dedicated IAM user or role for KOPS. Do not use AWS root account credentials.

The official KOPS AWS setup guide lists these AWS managed policies for its standard AWS workflow:

1. AmazonEC2FullAccess
2. AmazonRoute53FullAccess
3. AmazonS3FullAccess
4. IAMFullAccess
5. AmazonVPCFullAccess
6. AmazonSQSFullAccess
7. AmazonEventBridgeFullAccess

These are broad permissions intended for the documented setup. Use a carefully scoped least-privilege policy for production environments. Route53 permissions depend on the DNS configuration.

## 5. Configure AWS CLI

Configure your AWS credentials:

```bash
aws configure
```

Provide:

```text
AWS Access Key ID
AWS Secret Access Key
Default region name: ap-south-1
Default output format: json
```

If running on an EC2 instance, prefer an appropriately scoped IAM instance role instead of storing long-lived access keys.

Verify AWS authentication:

```bash
aws sts get-caller-identity
```

Check the configured region:

```bash
aws configure get region
```

## 6. Kubernetes Cluster Installation Using KOPS

### Step 1: Define environment variables

Set the AWS region:

```bash
export AWS_REGION=ap-south-1
export AWS_DEFAULT_REGION=ap-south-1
```

Define the cluster name:

```bash
export KOPS_CLUSTER_NAME=demok8scluster.k8s.local
```

Define the S3 bucket:

```bash
export KOPS_STATE_STORE=s3://kops-saleem-storage
```

Define the availability zone:

```bash
export AWS_ZONE=ap-south-1a
```

These variables avoid repeating the same values in every KOPS command.

### Step 2: Create an S3 bucket for KOPS state storage

The S3 bucket name must be globally unique.

For regions other than us-east-1, specify the location constraint.

```bash
aws s3api create-bucket \
  --bucket kops-saleem-storage \
  --region ap-south-1 \
  --create-bucket-configuration LocationConstraint=ap-south-1
```

Expected output:

```json
{
    "Location": "http://kops-saleem-storage.s3.amazonaws.com/",
    "BucketArn": "arn:aws:s3:::kops-saleem-storage"
}
```

If the bucket already exists and belongs to your AWS account, do not create it again.

Enable S3 bucket versioning:

```bash
aws s3api put-bucket-versioning \
  --bucket kops-saleem-storage \
  --versioning-configuration Status=Enabled
```

Verify:

```bash
aws s3api get-bucket-versioning \
  --bucket kops-saleem-storage
```

Expected:

```json
{
    "Status": "Enabled"
}
```

**Why enable versioning?**

Versioning helps recover previous versions of KOPS state objects if they are accidentally overwritten or deleted.

Keep the state bucket private and do not make it publicly accessible.

### Step 3: Create the Kubernetes cluster configuration

Use the following command:

```bash
kops create cluster \
  --name=${KOPS_CLUSTER_NAME} \
  --state=${KOPS_STATE_STORE} \
  --zones=${AWS_ZONE} \
  --node-count=1 \
  --node-size=t3.small \
  --control-plane-size=t3.medium \
  --control-plane-volume-size=20 \
  --node-volume-size=20 \
  --dns=none
```

Explanation:

| Parameter | Purpose |
|---|---|
| --name | Kubernetes cluster name |
| --state | S3 state store location |
| --zones | AWS availability zone |
| --node-count | Number of worker nodes |
| --node-size | Worker EC2 instance type |
| --control-plane-size | Control-plane EC2 instance type |
| --control-plane-volume-size | Control-plane root volume |
| --node-volume-size | Worker root volume |
| --dns=none | Disables gossip DNS |

**Note:** The instance sizes are illustrative small lab settings, not production recommendations. Verify resource requirements and AWS pricing before deployment.

### Step 4: Understand the cluster configuration

The `kops create cluster` command creates the cluster configuration and stores it in the state store.

It does not mean the complete AWS infrastructure is already running.

Check the cluster:

```bash
kops get cluster \
  --name=${KOPS_CLUSTER_NAME} \
  --state=${KOPS_STATE_STORE}
```

View the configuration:

```bash
kops get cluster \
  --name=${KOPS_CLUSTER_NAME} \
  --state=${KOPS_STATE_STORE} \
  -o yaml
```

Edit the configuration:

```bash
kops edit cluster \
  --name=${KOPS_CLUSTER_NAME} \
  --state=${KOPS_STATE_STORE}
```

### Step 5: Configure SSH and Kubernetes API access

Restrict access to your own public IP address instead of allowing connections from everywhere.

Inside the cluster configuration, set:

```yaml
spec:
  sshAccess:
  - YOUR_PUBLIC_IP/32

  kubernetesApiAccess:
  - YOUR_PUBLIC_IP/32
```

Replace `YOUR_PUBLIC_IP` with your actual public IP address.

These settings restrict SSH access and Kubernetes API access through the relevant AWS security groups.

### Step 6: Preview infrastructure changes

Run:

```bash
kops update cluster \
  --name=${KOPS_CLUSTER_NAME} \
  --state=${KOPS_STATE_STORE}
```

This previews the AWS resources KOPS intends to create or update.

Review the output before applying changes.

### Step 7: Apply infrastructure changes

If the preview is correct:

```bash
kops update cluster \
  --name=${KOPS_CLUSTER_NAME} \
  --state=${KOPS_STATE_STORE} \
  --yes
```

This applies the configuration and provisions the required AWS infrastructure.

It may take several minutes for the resources and Kubernetes components to become ready.

## 7. Validate the Kubernetes Cluster

Export Kubernetes credentials:

```bash
kops export kubecfg \
  --name=${KOPS_CLUSTER_NAME} \
  --state=${KOPS_STATE_STORE} \
  --admin
```

Validate the cluster:

```bash
kops validate cluster \
  --name=${KOPS_CLUSTER_NAME} \
  --state=${KOPS_STATE_STORE}
```

Check Kubernetes nodes:

```bash
kubectl get nodes -o wide
```

Check all Pods:

```bash
kubectl get pods -A
```

Check cluster information:

```bash
kubectl cluster-info
```

A successful validation indicates that KOPS can communicate with the cluster and the required Kubernetes components are operational.

## 8. Basic Kubernetes Commands

List nodes:

```bash
kubectl get nodes
```

List namespaces:

```bash
kubectl get namespaces
```

List Pods in all namespaces:

```bash
kubectl get pods -A
```

List Deployments:

```bash
kubectl get deployments -A
```

List Services:

```bash
kubectl get services -A
```

Describe a node:

```bash
kubectl describe node NODE_NAME
```

View cluster events:

```bash
kubectl get events -A
```

## 9. Important AWS Cost Considerations

Creating a Kubernetes cluster on AWS can generate charges.

Resources that may incur costs include:

- EC2 instances.
- EBS volumes.
- Network Load Balancers.
- Public IPv4 addresses.
- Data transfer.
- Other networking and storage resources.

Even small instances and volumes can incur charges.

Always review the infrastructure preview and AWS pricing before applying changes.

## 10. Delete the Kubernetes Cluster

When the cluster is no longer required, delete it to avoid unnecessary infrastructure costs.

Preview deletion:

```bash
kops delete cluster \
  --name=${KOPS_CLUSTER_NAME} \
  --state=${KOPS_STATE_STORE}
```

If the resources shown belong to the intended cluster, execute:

```bash
kops delete cluster \
  --name=${KOPS_CLUSTER_NAME} \
  --state=${KOPS_STATE_STORE} \
  --yes
```

Verify that the cluster's AWS resources have been removed.

**Important:** The S3 state bucket is a separate resource. Do not delete it if you intend to reuse it or retain cluster state and backups. Check its contents and versioning before any bucket cleanup.

---

## 11. Troubleshooting

### Error: IllegalLocationConstraintException

Cause: S3 bucket creation in a region other than us-east-1 without a location constraint.

Solution:

```bash
aws s3api create-bucket \
  --bucket YOUR_BUCKET_NAME \
  --region ap-south-1 \
  --create-bucket-configuration LocationConstraint=ap-south-1
```

### Error: AccessDenied

Cause: The IAM identity does not have the required AWS API permission.

Solution:
- Identify the denied API action.
- Review the IAM policy.
- Add the required least-privilege permission.
- Retry the operation.

### Error: Cluster already exists

Cause: A cluster configuration with the same name already exists in the KOPS state store.

Solution:

```bash
kops get cluster \
  --name=${KOPS_CLUSTER_NAME} \
  --state=${KOPS_STATE_STORE}
```

Use `kops update cluster` to apply changes to an existing configuration.

### Error: Kubernetes cluster validation failed

Check:

```bash
kops validate cluster \
  --name=${KOPS_CLUSTER_NAME} \
  --state=${KOPS_STATE_STORE}
```

Check EC2 instance status, security groups, networking, and control-plane logs.

---

## 12. Official Documentation

- Kubernetes: https://kubernetes.io/docs/
- kubectl installation: https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/
- AWS CLI installation: https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html
- KOPS installation: https://kops.sigs.k8s.io/getting_started/install/
- KOPS AWS installation: https://kops.sigs.k8s.io/getting_started/aws/
- KOPS state store: https://kops.sigs.k8s.io/state/
- KOPS cluster commands: https://kops.sigs.k8s.io/cli/kops_create_cluster/

---

**Repository Status:** Work in Progress (WIP)

The purpose of this repository is to document Kubernetes concepts and practical implementations in a simple, beginner-friendly way.
