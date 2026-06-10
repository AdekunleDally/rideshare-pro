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
│   ├── external-secrets/
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
NGINX Ingress Controller is a Kubernetes component that routes external HTTP/HTTPS traffic into your cluster using NGINX as a reverse proxy. Nginx Ingress Controller acts as the ApI gateway. It sits in front of your backend services and handles:

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