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
├── stateful/
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
---
### Install Cluster Autoscaler on the EKS
Cluster Autoscaler  automatically adjusts the number of worker nodes in the cluster by either scaling out (adding nodes) or scaling in (removing nodes)

#### Create IAM Policy
```
curl -O https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/iam-policy.json

aws iam create-policy \
  --policy-name AmazonEKSClusterAutoscalerPolicy \
  --policy-document file://iam-policy.json
```
#### Create IAM role for Service Account
```
eksctl create iamserviceaccount \
  --cluster=lukman-rideshare-cluster \
  --namespace=kube-system \
  --name=cluster-autoscaler \
  --attach-policy-arn=<POLICY_ARN> \
  --approve

  NB: you get the POLICY_ARN after creating the iam-policy
```

#### Deploy Cluster Autoscaler
##### Download official manifest:

```
curl -O https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml
```
##### Apply the manifest

```
kubectl apply -f cluster-autoscaler-autodiscover.yaml
```
---
### PostgreSQL High Availability on Amazon EKS using CloudNativePG

#### Implementation of a highly available PostgreSQL database on Amazon EKS using CloudNativePG (CNPG).

The solution provides:

* PostgreSQL 16
* 1 Primary instance
* 2 Read Replica instances
* Automated replication management
* Automated failover
* Persistent storage using Amazon EBS
* Secret management using External Secrets Operator (ESO)
* Integration with AWS Secrets Manager

---

#### Architecture

```text
                        AWS Secrets Manager
                                │
                                │
                                ▼
                    External Secrets Operator
                                │
                                │
                                ▼
                     Kubernetes Secret
                       postgres-secret
                                │
                                │
                                ▼
                      CloudNativePG Cluster
                                │
          ┌─────────────────────┼─────────────────────┐
          │                     │                     │
          ▼                     ▼                     ▼
    postgres-1            postgres-2            postgres-3
     Primary               Replica               Replica
          │                     │                     │
          └────────── Streaming Replication ─────────┘

                                │
                                ▼
                          Amazon EBS
                        Persistent Storage
```

---

#### Why CloudNativePG?

Traditionally, PostgreSQL replication requires:

* StatefulSets
* Replication configuration
* Replication users
* WAL management
* Failover scripts
* Promotion logic

CloudNativePG automates all these operations through a Kubernetes Operator.

Benefits include:

* Automated cluster creation
* Automated replication
* Automated failover
* Backup support
* Recovery support
* Rolling upgrades
* Kubernetes-native management

---

# Prerequisites

## EKS Cluster

A running EKS cluster with worker nodes.

Verify:

```bash
kubectl get nodes
```

Expected:

```text
NAME                                            STATUS
ip-192-168-129-187.us-east-2.compute.internal   Ready
ip-192-168-191-123.us-east-2.compute.internal   Ready
```

---

## StorageClass

A StorageClass must exist.

Verify:

```bash
kubectl get storageclass
```

Example:

```text
NAME   PROVISIONER
gp2     kubernetes.io/aws-ebs 
```

---

## AWS EBS CSI Driver

CloudNativePG relies on Persistent Volumes.

Verify:

```bash
kubectl get csidriver
```

Expected:

```text
ebs.csi.aws.com
```

---

## External Secrets Operator

ESO synchronizes secrets from AWS Secrets Manager into Kubernetes.

Verify:

```bash
kubectl get pods -n external-secrets
```

---

#### Secret Management

##### AWS Secrets Manager Secret

A secret named:

```text
postgres-secret
```

contains:

```json
{
  "POSTGRES_USER": "rideshare",
  "POSTGRES_PASSWORD": "ridesharepass"
}
```

---

## ClusterSecretStore

```yaml
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-2
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets
            namespace: external-secrets
```

This allows ESO to authenticate to AWS Secrets Manager.

---

## ExternalSecret

File: `externalsecret.yaml`

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret

metadata:
  name: postgres-secret-es

spec:
  refreshInterval: 1h

  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore

  target:
    name: postgres-secret
    creationPolicy: Owner

  data:
    - secretKey: username
      remoteRef:
        key: postgres-secret
        property: POSTGRES_USER

    - secretKey: password
      remoteRef:
        key: postgres-secret
        property: POSTGRES_PASSWORD
```

Apply:

```bash
kubectl apply -f externalsecret.yaml
```

---

## Verify Secret

```bash
kubectl get secret postgres-secret
```

Expected:

```text
NAME              TYPE
postgres-secret   Opaque
```

Inspect:

```bash
kubectl get secret postgres-secret -o yaml
```

Expected keys:

```yaml
username:
password:
```

---

### PostgreSQL on Amazon EKS with CloudNativePG

#### The Rideshare platform uses **CloudNativePG (CNPG)** to deploy and manage a highly available PostgreSQL cluster on Amazon EKS. The solution provides automated replication, failover, storage management, and Kubernetes-native database operations.

#### Deployment

#### Install CloudNativePG

```bash
helm repo add cnpg https://cloudnative-pg.github.io/charts

helm install cnpg cnpg/cloudnative-pg \
  -n cnpg-system \
  --create-namespace
```
#### NB: The postgres manifest file has a postgres-secret that was pulled from the AWS secrets manager using the clustersecretstore and externalsecret.
Verify installation:

```bash
kubectl get pods -n cnpg-system
```

### Deploy PostgreSQL Cluster

```bash
kubectl apply -f postgres-cluster.yaml
```

Cluster configuration:

* PostgreSQL 16
* 3 Instances (1 Primary, 2 Replicas)
* 20Gi gp3 EBS volume per instance
* Streaming replication enabled
* Credentials sourced from Kubernetes Secrets (integrated with AWS Secrets Manager via External Secrets Operator)

## Validation

Verify cluster health:

```bash
kubectl get cluster
```

Verify database pods:

```bash
kubectl get pods
```

## Database Access

CloudNativePG automatically creates service endpoints:

| Service | Purpose                      |
| ------- | ---------------------------- |
| `*-rw`  | Read/Write access to Primary |
| `*-ro`  | Read-only access to Replicas |
| `*-r`   | Read access to all instances |

Applications should use the **read-write service (`-rw`)** for transactional workloads.

## Connectivity Test

```bash
kubectl exec -it <postgres-pod> -- \
psql -U rideshare -d postgres
```

```sql
SELECT version();
```

## High Availability

CloudNativePG automatically handles:

* Primary/Replica replication
* Automatic failover
* Replica replacement
* Cluster self-healing

Failover can be tested by deleting the current primary pod:

```bash
kubectl delete pod <primary-pod>
```

The operator promotes a replica and restores the cluster to a healthy state without manual intervention.

## Troubleshooting

### Pods Stuck in Pending

```bash
kubectl describe pod <pod-name>
```

### PVC Binding Issues

```bash
kubectl describe pvc <pvc-name>
```

Ensure the `gp2` or `gp3`  StorageClass exists and is configured correctly.

### Secret Issues

```bash
kubectl get secret postgres-secret
```

Required keys:

```yaml
username
password
```

## Benefits

* Highly Available PostgreSQL
* Automated Replication and Failover
* Kubernetes-Native Operations
* Dynamic EBS Provisioning
* AWS Secrets Manager Integration
* Production-Ready Architecture
* Minimal Operational Overhead

