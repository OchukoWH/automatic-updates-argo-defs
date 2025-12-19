# VProfile Application - GitOps Kubernetes Definitions

![Kubernetes](https://img.shields.io/badge/kubernetes-v1.28-326CE5?style=flat&logo=kubernetes&logoColor=white)
![ArgoCD](https://img.shields.io/badge/argocd-gitops-EF7B4D?style=flat&logo=argo&logoColor=white)

This repository contains Kubernetes manifests for deploying the **vprofile** multi-tier Java application using **GitOps** with ArgoCD.

## 🎯 Project Overview

This project demonstrates GitOps best practices for automated Kubernetes deployments. The application runs on infrastructure provisioned by the [3-Node Production Grade Kubernetes Cluster with Vagrant & Ansible](https://github.com/OchukoWH/3-Node-Production-Grade-Kubernetes-Cluster-with-Vagrant-Ansible.git) project.

**GitOps Benefits:**
- Version-controlled infrastructure
- Automated deployments from Git
- Easy rollback via Git history
- Complete audit trail

## 🏗️ Application Architecture

```
┌─────────────┐
│   Ingress   │  (NGINX - SSL enabled)
└──────┬──────┘
       │
┌──────▼──────────┐
│  VProfile Web   │  (NGINX Reverse Proxy - Port 80)
└──────┬──────────┘
       │
┌──────▼──────────┐
│  VProfile App   │  (Java/Tomcat - Port 8080)
└────┬───┬───┬────┘
     │   │   │
┌────▼─────┐  ┌───────▼────┐  ┌──────────┐
│  MySQL   │  │ Memcached  │  │ RabbitMQ │
│ Database │  │   Cache    │  │  Queue   │
└──────────┘  └────────────┘  └──────────┘
```

**Components:**
- **VProfile Web**: NGINX reverse proxy with auto-scaling (HPA)
- **VProfile App**: Java application with auto-scaling (HPA)
- **MySQL**: Persistent database with 3Gi storage
- **Memcached**: Application caching layer
- **RabbitMQ**: Message queue for async processing

## 📋 Prerequisites

- Kubernetes cluster (v1.19+) - [Setup guide](https://github.com/OchukoWH/3-Node-Production-Grade-Kubernetes-Cluster-with-Vagrant-Ansible.git)
- ArgoCD installed
- NGINX Ingress Controller
- Metrics Server (for HPA)

## 🚀 Quick Deployment

### 1. Apply ArgoCD Manifests

```bash
# Create the Project and Application
kubectl apply -f argo-project/argocd-project.yaml
kubectl apply -f argo-project/argocd-application.yaml

# Watch deployment
kubectl get applications -n argocd -w
```

### 2. Monitor Deployment

```bash
# Check pods
kubectl get pods -n vprofile

# Check services
kubectl get svc,ingress -n vprofile

# View logs
kubectl logs -n vprofile -l app=vproapp -f
```

### 3. Access Application

**URL**: Update your domain in `vprofile/appingress.yaml`

**Login Credentials**:
- Username: `admin_vp`
- Password: `admin_vp`

## 📁 Repository Structure

```
argo-project-defs/
├── argo-project/              # ArgoCD configurations
│   ├── argocd-project.yaml    # Project definition
│   └── argocd-application.yaml # Application with auto-sync
└── vprofile/                  # Kubernetes manifests
    ├── appdeploy.yaml         # Application deployment
    ├── appservice.yaml
    ├── webdeploy.yaml         # Web/NGINX reverse proxy deployment
    ├── webservice.yaml
    ├── appingress.yaml        # NGINX ingress with SSL
    ├── dbdeploy.yaml          # MySQL deployment
    ├── dbservice.yaml
    ├── dbpvc.yaml            # 3Gi persistent storage
    ├── mcdep.yaml            # Memcached
    ├── mcservice.yaml
    ├── rmqdeploy.yaml        # RabbitMQ
    ├── rmqservice.yaml
    ├── secret.yaml           # DB and RabbitMQ passwords
    ├── hpa-all.yaml          # Auto-scaling for app, cache, queue
    ├── resource-quota.yaml   # Namespace resource limits
    └── limit-range.yaml      # Default pod limits
```

## ⚙️ Configuration

### Update Domain

Edit `vprofile/appingress.yaml`:
```yaml
spec:
  rules:
  - host: your-domain.com  # Change this
```

### Update Secrets (Optional)

Default credentials in `vprofile/secret.yaml`:
- Database password: `vprodbpass`
- RabbitMQ password: `guest`

To change:
```bash
echo -n 'new-password' | base64
# Update secret.yaml with the encoded value
```

## 🔄 GitOps Workflow

1. **Make changes** to manifests in `vprofile/` directory
2. **Commit & push** to Git
3. **ArgoCD auto-syncs** changes to cluster (prune + self-heal enabled)
4. **Rollback** by reverting Git commit

## 🔄 Automatic Image Updates

This repository automatically updates image tags in deployments through an integrated CI/CD pipeline.

### How It Works

1. **Docker Image Build & Push**: The [docker-build-push-and-update-argo-defs](https://github.com/OchukoWH/docker-build-push-and-update-argo-defs.git) repository builds new Docker images and pushes them to Docker Hub
2. **Automatic Manifest Updates**: The same repository automatically updates this repository with the new image tags
3. **ArgoCD Sync**: ArgoCD detects the changes in this repository and automatically applies them to the Kubernetes cluster

### Required Permissions & Access

#### GitHub Repository Permissions

The [docker-build-push-and-update-argo-defs](https://github.com/OchukoWH/docker-build-push-and-update-argo-defs.git) repository requires **write permission** to this repository to update the image tags in the deployment manifests.

#### Docker Hub Access

ArgoCD requires access to Docker Hub to pull images from private repositories. This access is configured using **Sealed Secrets** for secure credential management.

### Secret Management

All access credentials and secrets are managed using **Sealed Secrets**:
- Docker Hub credentials for ArgoCD
- GitHub tokens for repository access
- Other sensitive configuration data

Sealed Secrets provide encrypted secrets that can be safely stored in Git and decrypted only by the Sealed Secrets controller in your cluster.

## 🔍 Monitoring

```bash
# Application status
kubectl get pods,svc,ingress -n vprofile

# HPA status
kubectl get hpa -n vprofile

# Resource usage
kubectl top pods -n vprofile

# ArgoCD status
kubectl get application vprofile-app -n argocd
```

## 🛠️ Troubleshooting

### Pods not starting
```bash
kubectl describe pod <pod-name> -n vprofile
kubectl logs <pod-name> -n vprofile
```

### Database issues
Check if init containers completed and PVC is bound:
```bash
kubectl get pvc -n vprofile
kubectl describe deployment vprodb -n vprofile
```

### Ingress not working
Ensure NGINX Ingress Controller is installed and DNS points to your cluster.

## 🏗️ Infrastructure Setup

This GitOps project deploys on Kubernetes infrastructure from:
**[3-Node Production Grade Kubernetes Cluster with Vagrant & Ansible](https://github.com/OchukoWH/3-Node-Production-Grade-Kubernetes-Cluster-with-Vagrant-Ansible.git)**

The infrastructure project provides:
- 3-node Kubernetes cluster (1 master, 2 workers)
- Automated setup with Vagrant + Ansible
- Pre-configured networking and storage
- Calico CNI, NGINX Ingress, Metrics Server

## 📚 Resources

- [Infrastructure Setup Guide](https://github.com/OchukoWH/3-Node-Production-Grade-Kubernetes-Cluster-with-Vagrant-Ansible.git)

## 👥 Credits

- **Infrastructure**: [3-Node Production Grade Kubernetes Cluster with Vagrant & Ansible](https://github.com/OchukoWH/3-Node-Production-Grade-Kubernetes-Cluster-with-Vagrant-Ansible.git)
- **GitOps**: ArgoCD-based automated deployment
