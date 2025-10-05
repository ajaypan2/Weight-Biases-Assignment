Weight & Biases Assignment

Deploying a Self-Managed Weights & Biases Server on AWS EKS Using Helm Charts

This Readme file walks you through the steps for Deploying a Weights & Biases Server on AWS EKS Using Helm Charts.

First this First.

What is Weight & Biases

Weight & Biases is the AI Developer Platform with tools for training models, fine tuning models and leveraging foundational models.
It has 4 main core components - Core, Models, Weave and Inference.
It also has a Platform (Weight & Biases Platform) which is the kind of foundational infrastructure, tooling, governance scaffolding which support the Weight & biases Products like Core, Weave and Model.

Weight & Biases Platform is available in 3 different deployment options.

1.Multi-Teanant (SAAS) - Fully managed service deployed in Weight & Biases Cloud Infrastructure which provides seamless access to Weight & Biases Products at a desired scale,
Cost efficient for pricing and continuous updates for latest features and functionalities.

2.Dedicated Cloud - Single Tenant. Fully managed service deployed in Weight & Biases Cloud Infrastructure. This is good for if you have a strong compliance and data residency requirements.

3.Customer Managed - Customers are responsible for provisioning and managing the infrastructure and deploying the Weight & Biases server. They also need to take care of maintenance, backups along with scaling, security and Performance characteristics.

Ways to Deploy the Weight & Server.

Weight & Biases offer 3 Options to deploy the Weight & Biases Server.

1.Terraform 

2.Helm Chart 

3.Operators.

Have explored all 3 options but decided to pick Helm Charts for this Assignments and plan to try the operators option next.

So here we go with the actual Steps.

Step 1. Prerequisites 

AWS Account with AdminPrivileges to Provision the AWS Resources (VPC, EKS etc.)
AWS CLI - Access AWS resources Programmatically.
Eksctl - Create the EKS cluster.
kubectl - Interact with the  cluster.
Helm  -  Deploy W &B Server.
Python - Python script for log runs.

Step 2 : Create EKS Cluster 

eksctl create cluster --name wandb --region us-east-1

This provisions:
VPC with public & private subnets
Managed Node Group (2 nodes, autoscaling enabled)
Control Plane
AWS Managed Add-ons (VPC CNI, CoreDNS, Kube-Proxy, Metrics Server)

Step 3: Install EBS CSI Driver

W&B uses MySQL for metadata, which requires persistent storage.

 1. Associate OIDC Provider:
  eksctl utils associate-iam-oidc-provider \
  --region us-east-1 \
  --cluster wandb \
  --approve

 2. Create IAM Service Account:
  eksctl create iamserviceaccount \
  --cluster wandb \
  --namespace kube-system \
  --name ebs-csi-controller-sa \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve

 3. Install CSI Driver:

helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
helm repo update

helm install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver \
  -n kube-system \
  --set controller.serviceAccount.create=false \
  --set controller.serviceAccount.name=ebs-csi-controller-sa

 4. Verify installation:

kubectl get pods -n kube-system -l "app.kubernetes.io/name=aws-ebs-csi-driver


Step 4: Configure Storage Class & PVC

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

Step 5: Generate W&B License

Obtain a temporary license key from the W&B website.

Step 6: Deploy W&B Server using Helm

 1. Add repo:

helm repo add wandb https://wandb.github.io/helm-charts/
helm repo update

 2. Create values.yaml:


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

 3. Install:


helm install wandb wandb/wandb \
  -f values.yaml \
  --namespace wandb --create-namespace

Step 7: Verify & Access W&B

kubectl get svc -n wandb

Access W&B via the LoadBalancer URL.
 Create a user and log in.

Step 8: Run Experiments

1. Install SDK:

pip install wandb

2. Python script (run.py):

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

3. Authenticate & run:


wandb login --relogin --host=http://<LB-DNS>:8080
python3 run.py

Check runs in the W&B Console → Project: my-awesome-project.

******************************************************************************************************************************************************************************

Note: I could have used any other Deployment Options (Terraform Modules, Operators) for faster setup  but picked up Helm Charts for this assignmnets because it allowed me to explore more on the infrastructure setup and helped with Trobuleshooting.

Object storage (S3/MinIO) can also be configured for artifacts but skipped to keep things simple.

I do ran into multiple issue duing the set up. Please check out the Challenges file for the details related to the issue and fix.

Also checkout the Weight & Biases Set up file for more details along with the Screenshots.

Next up i will the Try the Operators option and will share feedback. Thank You.








