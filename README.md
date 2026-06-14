# RideShare Pro 🚖

Production-grade ride-sharing platform deployed on Amazon EKS using Kubernetes, microservices architecture, automated secret management, persistent storage, ingress-based traffic routing, and autoscaling.

---

## Project Overview

RideShare Pro is a cloud-native ride-sharing application designed to demonstrate production-grade Kubernetes deployment patterns on Amazon Elastic Kubernetes Service (EKS).

The platform consists of multiple independently deployable microservices responsible for rider management, driver management, trip processing, ride matching, notifications, and frontend presentation.

The project showcases:

* Amazon EKS cluster administration
* Kubernetes StatefulSets and Deployments
* NGINX Ingress Controller
* External Secrets Operator
* AWS Secrets Manager integration
* Horizontal Pod Autoscaling (HPA)
* Persistent storage using EBS CSI Driver
* Redis Cluster deployment
* PostgreSQL deployment
* TLS termination with cert-manager
* High availability patterns
* Production troubleshooting workflows

---

## Architecture

### High-Level Architecture

```text
Internet User
      |
      v
     DNS
      |
      v
NGINX Ingress Controller
      |
      v
Amazon EKS Cluster
      |
      +-----------------------------------+
      |                                   |
      v                                   v

Frontend Service                Backend APIs

/api/rider     -> rider-service
/api/driver    -> driver-service
/api/trip      -> trip-service
/ws            -> matching-service

```
---
### Service Architecture

| Service          | Technology           | Purpose            |
| ---------------- | -------------------- | ------------------ |
| Frontend         | Next.js / TypeScript | User Interface     |
| Rider Service    | Node.js / TypeScript | Rider Management   |
| Driver Service   | Node.js / TypeScript | Driver Management  |
| Trip Service     | Python Flask         | Trip Lifecycle     |
| Matching Service | Go                   | Driver Matching    |
| Email Service    | Python               | Notifications      |
| PostgreSQL       | StatefulSet          | Persistent Storage |
| Redis Cluster    | StatefulSet          | Pub/Sub & Cache    |

---
## AWS Infrastructure

### Region

```text
us-east-2
```

### EKS Cluster

```text
Cluster Name: rideshare-cluster
Kubernetes Version: v1.34+
```

### Node Group

| Property         | Value     |
| ---------------- | --------- |
| Instance Type    | t3.large |
| Desired Capacity | 2       |
| Min Nodes        | 2         |
| Max Nodes        | 6        |

### Networking

* VPC CIDR: 192.168.0.0/16
* 3 Public Subnets
* 3 Private Subnets
* NAT Gateway
* Route Tables
* Security Groups
## Technology Stack

### Cloud

* AWS EKS
* AWS EBS
* AWS Secrets Manager
* IAM Roles for Service Accounts (IRSA)

### Kubernetes

* Deployments
* StatefulSets
* Services
* Ingress
* ConfigMaps
* Secrets
* HPA
* PDB

### Platform Components

* NGINX Ingress Controller
* External Secrets Operator
* cert-manager
* Metrics Server
* EBS CSI Driver

### Databases

* PostgreSQL
* Redis Cluster

---
## Repository Structure

```text
rideshare-pro/
│
├── aws/
│   ├── cluster.yaml
│ 
│
├── platform/
│   ├── ingress/
│   ├── cert-manager/
│   ├── secrets/
│   ├── autoscaling/
│   └── pdb/
│
├── databases/
│   ├── postgres/
│   └── redis/
│
├── services/
│   ├── frontend/
│   ├── rider/
│   ├── driver/
│   ├── trip/
│   ├── matching/
│   └── email/
│
├── docs/
│   ├── ARCHITECTURE.md
│   ├── DEPLOYMENT.md
│   ├── TROUBLESHOOTING.md
│   ├── OPERATIONS.md
│   ├── SECURITY.md
│   └── DISASTER_RECOVERY.md
│
└── README.md
```

---
# Deployment Guide

## Phase 1 – Cluster Setup

### Create EKS Cluster

```bash
eksctl create cluster -f aws/cluster.yaml
```
### Install EBS CSI Driver
EBS CSI driver is  is a Kubernetes component that allows Kubernetes to dynamically create, attach, mount, resize, snapshot, and delete AWS EBS volumes.It is implemented as a addon  as seen in the eks cluster manifest file

```bash
eksctl create addon --config-file aws/cluster.yaml

```
When you run the command above, eksctl  automatically:

- Creates an IAM Role.
- Attaches the required EBS CSI permissions.
- Creates the IRSA trust relationship.
- Associates the role with the EBS CSI ServiceAccount.
- Installs the AWS-managed EBS CSI addon.

NB: No manual IAM policy creation is needed.

Verify:

```bash
kubectl get sc
```

---
## Phase 2 – Platform Components

### Install NGINX Ingress Controller
NGINX Ingress Controller is a Kubernetes component that routes external HTTP/HTTPS traffic into a Kubernetes cluster by automatically configuring NGINX, which acts as a reverse proxy and can be used as an API gateway.

As an API gateway, NGINX sits in front of backend services and can provide

- Routing requests to the right service
- Authentication / authorization
- Rate limiting
- TLS termination (HTTPS)
- Request/response filtering
- Logging and observability

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
kubectl create namespace ingress-nginx

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --set controller.service.type=LoadBalancer \
  --set controller.service.externalTrafficPolicy=Local \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-type"="nlb"

```
---
### Install Cert-Manager on the EKS

Cert-manager is a Kubernetes controller that automates the lifecycle of TLS certificates. It uses a Certificate object as the trigger, and a ClusterIssuer object as the configuration source to determine how and where to contact an external Certificate Authority like Let’s Encrypt.

#### Step 1: Add Helm repo
```
helm repo add jetstack https://charts.jetstack.io
helm repo update
```
#### Step 2: Create namespace
```
kubectl create namespace cert-manager
```

#### Step 3: Install cert-manager CRDs (IMPORTANT)

This is mandatory.
```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.crds.yaml
```

#### Step 4: Install cert-manager via Helm
```
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v1.14.4 \
  --set installCRDs=false
```

#### Step 5: Verify installation
```
kubectl get pods -n cert-manager
```
#### Step 6: Create a ClusterIssuer (Let’s Encrypt)
A ClusterIssuer is a cluster-wide configuration resource that tells cert-manager how to communicate with an external certificate authority such as Let's Encrypt, how to prove domain ownership, and how to obtain and renew certificates on behalf of Kubernetes workloads.
```
k apply -f platform/cert-manager/cluster-issuer.yaml
```
---
### Install External Secrets Operator
External Secrets Operator is a Kubernetes operator that synchronizes secrets from external secret stores such as AWS Secrets Manager into Kubernetes Secrets. As a result you dont have to store your secrets in your repo. Rather, you store it in a single source of truth. It uses clustersecretstore/secretstore resource(which define how you connect to the AWS secrets manager) and externalsecrets resource(which defines how each the particular secrets is retrieved from aws secrets manager into kubernetes cluster) to acieve this.

#### Install ESO

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

helm install external-secrets \
  external-secrets/external-secrets \
  -n external-secrets \
  --create-namespace \
  --set installCRDs=true

```
Upon installing the helm chart above, ESO gets installed as a deployment
#### Give ESO Permission to AWS Secrets Manager
```bash
. Create IAM Policy
. Get the OIDC provider ID
. Use the OIDC provider ID to create trust policy json file
. Use the trust policy file  to create the IAM role called ExternalSecretsRole
. Assign the created IAM policy to the IAM role
. Create service account external-secrets-sa

. Annotate the external-secrets-sa serviceaccount so that whenever a pod uses this ServiceAccount,a temporary AWS credentials for ExternalSecretsRole is given to it.

```
#### Make  ESO Use the external-secrets-sa ServiceAccount 
Ugrade the helm installation to use this service account so that the External Secrets Operator Pods run with the correct Kubernetes identity that is mapped via IRSA to an IAM Role. This allows them to securely assume AWS permissions and access AWS Secrets Manager without using node-level credentials or overly permissive default service accounts.

```bash
helm upgrade external-secrets \
external-secrets/external-secrets \
-n external-secrets \
--set serviceAccount.create=false \
--set serviceAccount.name=external-secrets-sa
```

#### Restart ESO
```bash
kubectl rollout restart deployment external-secrets \
-n external-secrets
```
#### Create ClusterSecretStore
```bash
 k apply -f secrets/eso/clustersecretstore.yaml
```

