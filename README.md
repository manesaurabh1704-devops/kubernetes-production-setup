# ☸️ Kubernetes Production Setup

> Production-grade Kubernetes manifests for StudentSphere application.
> Part of the [multi-cloud-devops-studentsphere](https://github.com/manesaurabh1704-devops/multi-cloud-devops-studentsphere) project.

---

📁 **Repository Structure**

```bash
kubernetes-production-setup/

├── common/                         # Shared Kubernetes resources
│   ├── namespace.yaml              # Application namespace
│   └── secrets.yaml                # Database credentials (K8s Secrets)
│
├── aws/                            # AWS EKS Kubernetes manifests
│   ├── database/
│   │   ├── mariadb-statefulset.yaml   # MariaDB StatefulSet + PVC
│   │   └── mariadb-service.yaml       # MariaDB ClusterIP Service
│   │
│   ├── backend/
│   │   ├── backend-deployment.yaml    # Spring Boot (replicas: 2)
│   │   └── backend-service.yaml       # Backend ClusterIP Service
│   │
│   ├── frontend/
│   │   ├── frontend-deployment.yaml   # React + Nginx (replicas: 2)
│   │   └── frontend-service.yaml      # Frontend LoadBalancer Service
│
├── azure/                          # Azure AKS manifests (Phase 2)
│   └── [To be implemented]
│
├── gcp/                            # Google GKE manifests (Phase 3)
│   └── [To be implemented]
│
└── README.md                       # Project documentation
```

---

## 🏗️ Architecture
Internet
↓
AWS LoadBalancer
↓
Frontend Pods (Nginx x2)
↓
Backend Pods (Spring Boot x2)
↓
MariaDB StatefulSet (x1)
↓
EBS Persistent Volume (5Gi)

---

## 📋 Kubernetes Resources

| Resource | Type | Replicas | Description |
|---|---|---|---|
| mariadb | StatefulSet | 1 | Persistent database |
| backend | Deployment | 2 | REST API server |
| frontend | Deployment | 2 | React + Nginx |
| frontend-service | LoadBalancer | - | Public access |
| backend-service | ClusterIP | - | Internal only |
| mariadb-service | Headless | - | StatefulSet DNS |
| db-secret | Secret | - | DB credentials |

---

## ⚡ How to Deploy on AWS EKS

### Prerequisites
```bash
# Tools required
aws --version       # AWS CLI
kubectl version     # kubectl
eksctl version      # eksctl
```

### Step 1 — Create EKS Cluster
```bash
eksctl create cluster \
  --name studentsphere-cluster \
  --region ap-south-1 \
  --nodegroup-name studentsphere-nodes \
  --node-type t3.small \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3 \
  --managed
```

### Step 2 — Install EBS CSI Driver
```bash
eksctl utils associate-iam-oidc-provider \
  --region ap-south-1 \
  --cluster studentsphere-cluster \
  --approve

eksctl create addon \
  --name aws-ebs-csi-driver \
  --cluster studentsphere-cluster \
  --region ap-south-1 \
  --force
```

### Step 3 — Deploy All Resources
```bash
kubectl apply -f aws/namespace.yaml
kubectl apply -f aws/secrets.yaml
kubectl apply -f aws/mariadb-deployment.yaml
kubectl apply -f aws/mariadb-service.yaml
kubectl apply -f aws/backend-deployment.yaml
kubectl apply -f aws/backend-service.yaml
kubectl apply -f aws/frontend-deployment.yaml
kubectl apply -f aws/frontend-service.yaml
```

### Step 4 — Verify All Pods Running
```bash
kubectl get all -n studentsphere
```

Expected output:
NAME                            READY   STATUS    RESTARTS
pod/backend-xxxx                1/1     Running   0
pod/backend-xxxx                1/1     Running   0
pod/frontend-xxxx               1/1     Running   0
pod/frontend-xxxx               1/1     Running   0
pod/mariadb-0                   1/1     Running   0
NAME                       TYPE           EXTERNAL-IP
service/frontend-service   LoadBalancer   xxxx.ap-south-1.elb.amazonaws.com
service/backend-service    ClusterIP      <none>
service/mariadb-service    ClusterIP      None

### Step 5 — Get App URL
```bash
kubectl get svc frontend-service -n studentsphere \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

---

## 🐛 Troubleshooting

### Problem 1 — MariaDB Pod Pending
```bash
# Check error
kubectl describe pod mariadb-0 -n studentsphere | grep -A 5 Events

# Fix — Install EBS CSI Driver
eksctl create addon --name aws-ebs-csi-driver \
  --cluster studentsphere-cluster --region ap-south-1 --force
```

### Problem 2 — Frontend CrashLoopBackOff
```bash
# Check logs
kubectl logs -n studentsphere -l app=frontend

# Error: host not found in upstream "backend"
# Fix — nginx.conf mein use backend-service instead of backend
proxy_pass http://backend-service:8080/api/;
```

### Problem 3 — t3.medium Launch Failed
Error: instance type not eligible
Fix: use t3.small
--node-type t3.small

---

## 🔗 Main Project

This repository is part of the main project:
👉 [multi-cloud-devops-studentsphere](https://github.com/manesaurabh1704-devops/multi-cloud-devops-studentsphere)

| Repository | Purpose |
|---|---|
| [multi-cloud-devops-studentsphere](https://github.com/manesaurabh1704-devops/multi-cloud-devops-studentsphere) | Main project — Full DevOps system |
| [terraform-multi-cloud-infra](https://github.com/manesaurabh1704-devops/terraform-multi-cloud-infra) | Infrastructure as Code |
| [ci-cd-devops-pipelines](https://github.com/manesaurabh1704-devops/ci-cd-devops-pipelines) | Jenkins CI/CD pipelines |
| [monitoring-observability-stack](https://github.com/manesaurabh1704-devops/monitoring-observability-stack) | Prometheus + Grafana |
| [devops-security-secrets](https://github.com/manesaurabh1704-devops/devops-security-secrets) | RBAC + Security |

---

## 👨‍💻 Author
**Saurabh Mane** — DevOps Engineer
- GitHub: [@manesaurabh1704-devops](https://github.com/manesaurabh1704-devops)

---

> ⭐ Star this repo if you find it helpful!
