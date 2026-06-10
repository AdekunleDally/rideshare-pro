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
