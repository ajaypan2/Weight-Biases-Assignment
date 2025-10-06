 Weights & Biases Assignment

 Deploying a Self-Managed Weights & Biases Server on AWS EKS Using Helm Charts

 This README walks you through the steps for deploying a Weights & Biases (W&B)server on AWS EKSusing Helm Charts.


What is Weights & Biases?

Weights & Biases (W&B) is an AI Developer Platform that provides tools for:
- Training models  
- Fine-tuning models  
- Leveraging foundational models  

It has four main core components:
- Core
- Models
- Weave
- Inference

Additionally, it includes the Weights & Biases Platform the foundational infrastructure that supports products like Core, Weave, and Models with tooling, governance, and scalability.

---

Deployment Options

The W&B Platform is available in three deployment modes:

1. Multi-Tenant (SaaS) 
   Fully managed by W&B in their cloud environment.  
   Ideal for most users, cost-efficient and continuously updated.

2. Dedicated Cloud
   Single-tenant managed deployment hosted by W&B.  
   Recommended for strong compliance or data residency needs.

3. Customer Managed 
   You provision and manage infrastructure, scaling, backups, and security.

---

Ways to Deploy the W&B Server

W&B supports three deployment methods:

1. Terraform
2. Helm Charts
3. Kubernetes Operators

I explored all three but selected Helm Charts for this assignment to better understand the underlying infrastructure setup and troubleshooting.

---

Deployment Steps

Step 1: Prerequisites

Ensure you have:
- AWS Account (with admin privileges)
- AWS CLI – to access AWS programmatically
- eksctl – to create the EKS cluster
- kubectl – to interact with Kubernetes
- Helm – to deploy the W&B Server
- Python – for running experiment scripts

---

Step 2: Create an EKS Cluster

eksctl create cluster --name wandb --region us-east-1

This provisions:

A VPC with public and private subnets

Managed Node Group (2 nodes, autoscaling enabled)

Control Plane

AWS Managed Add-ons (VPC CNI, CoreDNS, Kube-Proxy, Metrics Server)

Step 3: Install EBS CSI Driver

W&B uses MySQL for metadata, which requires persistent storage.

Associate OIDC Provider

eksctl utils associate-iam-oidc-provider \
  --region us-east-1 \
  --cluster wandb \
  --approve
  
Create IAM Service Account

eksctl create iamserviceaccount \
  --cluster wandb \
  --namespace kube-system \
  --name ebs-csi-controller-sa \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve
  
Install CSI Driver

helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver

helm repo update

helm install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver \
  -n kube-system \
  --set controller.serviceAccount.create=false \
  --set controller.serviceAccount.name=ebs-csi-controller-sa
  
Verify Installation

kubectl get pods -n kube-system -l "app.kubernetes.io/name=aws-ebs-csi-driver"

Step 4: Configure Storage Class & PVC

Create a file named ebs_csi.yaml and apply it:


apiVersion: storage.k8s.io/v1

kind: StorageClass

metadata:

  name: ebs-sc
  
provisioner: ebs.csi.aws.com

parameters:

  type: gp3
  
  fsType: ext4
  
reclaimPolicy: Delete

volumeBindingMode: WaitForFirstConsumer

---
apiVersion: v1

kind: PersistentVolumeClaim

metadata:

  name: ebs-pvc
  
spec:

  accessModes:
  
    - ReadWriteOnce
    
  resources:
  
    requests:
    
      storage: 10Gi
      
  storageClassName: ebs-sc
  
Apply it:

kubectl apply -f ebs_csi.yaml

Step 5: Generate W&B License

Obtain a temporary license key from the W&B website (https://deploy.wandb.ai/deploy)

Step 6: Deploy W&B Server Using Helm

Add Helm Repo

helm repo add wandb https://wandb.github.io/helm-charts/

helm repo update

Create values.yaml

license: XXXXXXXXXXXXXXXXXXXXXX

service:
  type: LoadBalancer

ingress:
  enabled: false

mysql:

  persistence:
  
    enabled: true
    
    size: "10Gi"
    
    storageClass: "ebs-sc"

resources:

  limits:
  
    cpu: 2
    
    memory: 4Gi
    
  requests:
  
    cpu: 1
    
    memory: 2Gi
    
Install the Chart

helm install wandb wandb/wandb \
  -f values.yaml \
  --namespace wandb --create-namespace
  
Step 7: Verify & Access W&B

Check the service:

kubectl get svc -n wandb

Access W&B via the LoadBalancer URL, create a user, and log in.

Step 8: Run Experiments

Install SDK

pip install wandb

Create Python Script (run.py)

import wandb, random

wandb.init(
    project="my-awesome-project",
    config={
        "learning_rate": 0.02,
        "architecture": "CNN",
        "dataset": "CIFAR-100",
        "epochs": 10,
    }
)

epochs = 10
offset = random.random() / 5
for epoch in range(2, epochs):
    acc = 1 - 2 ** -epoch - random.random() / epoch - offset
    loss = 2 ** -epoch + random.random() / epoch + offset
    wandb.log({"acc": acc, "loss": loss})

wandb.finish()


Authenticate and Run


wandb login --relogin --host=httphttp://a5d414f87d2544d19b49d06579dce6a0-1024552980.us-east-1.elb.amazonaws.com:8080

python3 run.py

Check runs in the W&B Console under the project “my-awesome-project”.



******************************************************************************************************************

Note: I could have used any other Deployment Options (Terraform Modules, Operators) for faster setup  but picked up Helm Charts for this assignmnets because it allowed me to explore more on the infrastructure setup and helped with Trobuleshooting.

Object storage (S3/MinIO) can also be configured for artifacts but skipped to keep things simple.

I do ran into multiple issue duing the set up. Please check out the Challenges file for the details related to the issue and fix.

Also checkout the Installation & Deployment doc for more details along with the Screenshots.

Next up i will the Try the Operators option and will share feedback. Thank You.
