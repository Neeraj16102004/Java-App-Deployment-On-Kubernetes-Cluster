# Java Application Deployment on Kubernetes Cluster
This repository contains the Kubernetes manifests and automation scripts required to deploy a multi-service Java web application. The deployment covers a robust stack, including a front-end application server, various backend supporting services, and persistent storage on a cloud environment.

# 🏗️ Architecture Overview
The system architecture, as detailed in the image below, is designed for scalability and high availability. It integrates several key Kubernetes components and cloud services to manage external traffic, provide secure communication, and handle stateful data.

```text
[ External Web Traffic ]
│
▼
┌────────────────────────────────────────────────────────────────────────┐
│                        AWS Cloud Environment                           │
│                                                                        │
│   ┌───────────────────────────┐                                        │
│   │ Application Load Balancer │                                        │
│   └─────────────┬─────────────┘                                        │
│                 │                                                      │
│                 ▼                                                      │
│   ┌────────────────────────────────────────────────────────────────┐   │
│   │                       Kubernetes Cluster                       │   │
│   │                                                                │   │
│   │     ┌─────────┐                                                │   │
│   │     │ Ingress │                                                │   │
│   │     └────┬────┘                                                │   │
│   │          │                                                     │   │
│   │          ▼                                                     │   │
│   │   ┌───────────────┐     ┌──────────────┐                       │   │
│   │   │ TomcatService ├────►│  Tomcat Pod  │                       │   │
│   │   └───────────────┘     └──────┬───────┘                       │   │
│   │                                │                               │   │
│   │         ┌──────────────────────┼──────────────────────┐        │   │
│   │         │                      │                      │        │   │
│   │         ▼                      ▼                      ▼        │   │
│   │  ┌────────────┐         ┌────────────┐         ┌───────────┐   │   │
│   │  │ RMQService │         │ MCService  │         │ DBService │   │   │
│   │  └─────┬──────┘         └─────┬──────┘         └─────┬─────┘   │   │
│   │        │                      │                      │         │   │
│   │        ▼                      ▼                      ▼         │   │
│   │  ┌────────────┐         ┌────────────┐         ┌───────────┐   │   │
│   │  │ RabbitMQ   │         │  Memcache  │         │  DB Pod   │   │   │
│   │  └─────┬──────┘         └────────────┘         └─────┬─────┘   │   │
│   │        │                               (/var/lib/mysql)│       │   │
│   │        └──────────────┐      ┌───────────────────────┘         │   │
│   │                       │      │                                 │   │
│   │                       ▼      ▼                                 │   │
│   │                  ┌──────────────┐                              │   │
│   │                  │    Secret    │                              │   │
│   │                  └──────────────┘                              │   │
│   │                              │                                 │   │
│   │                              ▼                                 │   │
│   │                 ┌──────────────────────────┐                   │   │
│   │                 │  PersistentVolumeClaim   │                   │   │
│   │                 └────────────┬─────────────┘                   │   │
│   │                              │                                 │   │
│   │                              ▼                                 │   │
│   │                     ┌─────────────────┐                        │   │
│   │                     │  StorageClass   │                        │   │
│   │                     └────────┬────────┘                        │   │
│   └──────────────────────────────┼─────────────────────────────────┘   │
│                                  │                                     │
│                                  ▼                                     │
│                         ┌─────────────────┐                            │
│                         │   Amazon EBS    │                            │
│                         └─────────────────┘                            │
└────────────────────────────────────────────────────────────────────────┘
```
---

## 🧩 Architectural Components & Workflow

### 1. External Ingress & Traffic Routing
* **Application Load Balancer (ALB):** Acts as the public entry point provisioned via the AWS Load Balancer Controller.
* **Kubernetes Ingress:** Receives traffic from the ALB and routes HTTP/HTTPS requests to internal services based on configured host/path rules.

### 2. Application Tier (Frontend / Middleware)
* **Tomcat Service (`TomcatService`):** Exposes the deployment internally using a `ClusterIP` abstraction.
* **Tomcat Pods:** Houses the core Java web application binaries (WAR/JAR) executing inside a Apache Tomcat container.

### 3. Backend Support Services
* **RabbitMQ (`RMQService` + `RabbitMQ` Pod):** Asynchronous message broker facilitating decoupled background task queues.
* **Memcache (`MCService` + `Memcache` Pod):** In-memory key-value cache layer reducing database read bottlenecks.
* **Database (`DBService` + `DB Pod`):** Relational database (MySQL/MariaDB) persisting domain data mounted to `/var/lib/mysql`.

### 4. Configuration & Security
* **Secrets (`Secret`):** Securely injects base64-encoded environment parameters (e.g., database credentials, RabbitMQ auth keys) into the RabbitMQ and DB pods without hardcoding inside image manifests.

### 5. Persistent Storage Engine
* **PersistentVolumeClaim (PVC):** Requests storage allocation dynamically for the database mount point `/var/lib/mysql`.
* **StorageClass (`sc`):** Configured with the AWS EBS CSI driver (`ebs.csi.aws.com`) to enable dynamic volume provisioning.
* **Amazon EBS:** Block storage volume provisioned in AWS to preserve database state beyond pod lifecycle terminations.

---

## 📂 Repository Directory Structure

```text
.
├── kubedefs/
│   ├── secret.yaml                # Encrypted credentials & environment configurations
│   ├── dbpvc.yaml                 # PersistentVolumeClaim for DB state
│   ├── dbdeploy.yaml              # Database deployment & pod specification
│   ├── dbservice.yaml             # Internal service endpoint for MySQL
│   ├── rmqdeploy.yaml             # RabbitMQ message broker deployment
│   ├── rmqservice.yaml            # Internal service endpoint for RabbitMQ
│   ├── mcdeploy.yaml              # Memcached deployment
│   ├── mcservice.yaml             # Internal service endpoint for Memcache
│   ├── appdeploy.yaml             # Main Java application deployment
│   ├── appservice.yaml            # Internal service endpoint for Java App
│   └── appingress.yaml            # Ingress rules mapping to AWS ALB
└── README.md
```
---
# 📋 Prerequisites
Before deploying the manifests, verify that your environment satisfies the following dependencies:

## 1. Kubernetes Cluster: v1.24+ (AWS EKS, Minikube, or custom cluster)

## 2. kubectl: Configured with administrative access (cluster-admin)

## 3. AWS EBS CSI Driver: Installed on the target EKS cluster to enable EBS dynamic provisioning

## 4. AWS Load Balancer Controller: Installed if provisioning the AWS ALB via Kubernetes Ingress annotations
---
