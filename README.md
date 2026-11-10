# Kubernetes-Zero-to-Hero

Creating this repo with an intent to make Kubernetes easy for beginners. This is a work-in-progress repo.

## Kubernetes Installation Using KOPS on EC2

### Create an EC2 instance or use your personal laptop.

Dependencies required:

1. AWS CLI
2. kubectl
3. KOPS
4. curl

### Install dependencies

Update the package list and install the required dependencies:

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl apt-transport-https gnupg
```

### Install kubectl

Add the Kubernetes repository:

```bash
sudo mkdir -p /etc/apt/keyrings

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.36/deb/Release.key | \
sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

```bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.36/deb/ /' | \
sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Install kubectl:

```bash
sudo apt-get update
sudo apt-get install -y kubectl
```

Verify:

```bash
kubectl version --client
```

### Install AWS CLI

Install AWS CLI using Snap:

```bash
sudo snap install aws-cli --classic
```

Verify:

```bash
aws --version
```

### Install KOPS (our hero for today)

Download and install the latest KOPS release:

```bash
curl -LO https://github.com/kubernetes/kops/releases/download/$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)/kops-linux-amd64

chmod +x kops-linux-amd64

sudo mv kops-linux-amd64 /usr/local/bin/kops
```

Verify:

```bash
kops version
```

### Provide the below permissions to your IAM user

If you are using an admin user, the below permissions are available by default.

1. AmazonEC2FullAccess
2. AmazonS3FullAccess
3. IAMFullAccess
4. AmazonVPCFullAccess
5. AmazonSQSFullAccess
6. AmazonEventBridgeFullAccess
7. AmazonRoute53FullAccess (depending on DNS configuration)

These are broad permissions. For production environments, use a least-privilege IAM policy.

### Set up AWS CLI configuration on your EC2 Instance or Laptop

Run:

```bash
aws configure
```

Example:

```text
AWS Access Key ID: YOUR_ACCESS_KEY
AWS Secret Access Key: YOUR_SECRET_KEY
Default region name: us-east-1
Default output format: json
```

Verify your AWS identity:

```bash
aws sts get-caller-identity
```

## Kubernetes Cluster Installation

Please follow the steps carefully and read each command before executing.

### Create S3 bucket for storing the KOPS objects

KOPS uses an S3 bucket as its state store. It stores cluster configuration and state information there.

```bash
aws s3api create-bucket --bucket kops-abhi-storage --region us-east-1
```

Enable bucket versioning:

```bash
aws s3api put-bucket-versioning \
  --bucket kops-abhi-storage \
  --versioning-configuration Status=Enabled
```

### Create the cluster

```bash
kops create cluster \
  --name=demok8scluster.k8s.local \
  --state=s3://kops-abhi-storage \
  --zones=us-east-1a \
  --node-count=1 \
  --node-size=t3.small \
  --control-plane-size=t3.medium \
  --control-plane-volume-size=20 \
  --node-volume-size=20 \
  --dns=none
```

**Example configuration:**

- Cluster name: `demok8scluster.k8s.local`
- S3 state store: `s3://kops-abhi-storage`
- AWS region: `us-east-1`
- Availability zone: `us-east-1a`
- Worker nodes: 1
- Worker node instance type: `t3.small`
- Control plane instance type: `t3.medium`

Note: `kops create cluster` creates the cluster configuration in the S3 state store. It does not immediately create all AWS infrastructure.

### Important: Edit the configuration

Before deploying, review the cluster configuration because multiple AWS resources will be created and they may incur charges.

View the cluster:

```bash
kops get cluster --name=demok8scluster.k8s.local --state=s3://kops-abhi-storage
```

Edit the configuration:

```bash
kops edit cluster --name=demok8scluster.k8s.local --state=s3://kops-abhi-storage
```

You can also view the configuration in YAML:

```bash
kops get cluster --name=demok8scluster.k8s.local --state=s3://kops-abhi-storage -o yaml
```

For security, restrict SSH and Kubernetes API access to your own public IP instead of allowing access from everyone.

Example configuration:

```yaml
spec:
  sshAccess:
  - YOUR_PUBLIC_IP/32
  kubernetesApiAccess:
  - YOUR_PUBLIC_IP/32
```

Replace `YOUR_PUBLIC_IP` with your actual public IP address.

### Build the cluster

First, preview the changes:

```bash
kops update cluster demok8scluster.k8s.local --state=s3://kops-abhi-storage
```

Review the resources that KOPS plans to create.

To apply the changes and create the AWS infrastructure:

```bash
kops update cluster demok8scluster.k8s.local --yes --state=s3://kops-abhi-storage
```

This will take a few minutes to create the cluster.

### Export Kubernetes configuration

After the cluster is created, export its credentials and configuration:

```bash
kops export kubecfg demok8scluster.k8s.local --state=s3://kops-abhi-storage --admin
```

This allows kubectl to communicate with the Kubernetes cluster.

### Verify the cluster installation

Run:

```bash
kops validate cluster demok8scluster.k8s.local --state=s3://kops-abhi-storage
```

Check the nodes:

```bash
kubectl get nodes
```

Check all pods:

```bash
kubectl get pods -A
```

Check cluster information:

```bash
kubectl cluster-info
```

### Delete the cluster

When you no longer need the cluster, delete the AWS resources to avoid unnecessary charges.

Preview the deletion:

```bash
kops delete cluster demok8scluster.k8s.local --state=s3://kops-abhi-storage
```

Confirm deletion:

```bash
kops delete cluster demok8scluster.k8s.local --yes --state=s3://kops-abhi-storage
```

Note: Deleting the cluster does not automatically delete the S3 state-store bucket. Keep it if you want to retain the cluster state, or delete it separately when you no longer need it.
