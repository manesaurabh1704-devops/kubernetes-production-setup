# ☸️ Kubernetes Production Setup

> Production-grade Kubernetes manifests for StudentSphere application.
> AWS EKS + Azure AKS + GCP GKE — all three clouds with same K8s manifests.
> Part of the [multi-cloud-devops-studentsphere](https://github.com/manesaurabh1704-devops/multi-cloud-devops-studentsphere) project.

---

## 📁 Repository Structure

```
kubernetes-production-setup/
├── aws/                             # AWS EKS Kubernetes manifests (Phase 2 & 5)
│   ├── namespace.yaml               # Kubernetes namespace
│   ├── secrets.yaml                 # DB credentials as K8s Secrets
│   ├── mariadb-deployment.yaml      # MariaDB StatefulSet + PVC (gp2)
│   ├── mariadb-service.yaml         # MariaDB Headless Service
│   ├── backend-deployment.yaml      # Spring Boot Deployment x2
│   ├── backend-service.yaml         # Backend ClusterIP Service
│   ├── frontend-deployment.yaml     # React + Nginx Deployment x2
│   ├── frontend-service.yaml        # Frontend LoadBalancer Service
│   ├── backend-hpa.yaml             # HPA — CPU 70% Memory 95% min2 max5
│   ├── frontend-hpa.yaml            # HPA — CPU 70% Memory 80% min2 max5
│   ├── backend-canary.yaml          # Canary deployment — 33% traffic
│   ├── backend-blue-green.yaml      # Blue-Green — zero downtime switch
│   ├── rbac.yaml                    # ServiceAccounts + Roles + RoleBindings
│   ├── network-policy.yaml          # Default deny + selective allow rules
│   └── argocd-app.yaml              # ArgoCD GitOps application
├── azure/                           # Azure AKS manifests (Phase 9) ✅
│   ├── namespace.yaml               # Kubernetes namespace
│   ├── secrets.yaml                 # DB credentials as K8s Secrets
│   ├── mariadb-deployment.yaml      # MariaDB StatefulSet + PVC (managed-csi)
│   ├── mariadb-service.yaml         # MariaDB Headless Service
│   ├── backend-deployment.yaml      # Spring Boot Deployment x2
│   ├── backend-service.yaml         # Backend ClusterIP Service
│   ├── frontend-deployment.yaml     # React + Nginx Deployment x2 (v3 image)
│   ├── frontend-service.yaml        # Frontend LoadBalancer Service
│   ├── backend-hpa.yaml             # HPA — Auto-scale backend
│   ├── frontend-hpa.yaml            # HPA — Auto-scale frontend
│   ├── backend-canary.yaml          # Canary deployment
│   ├── backend-blue-green.yaml      # Blue-Green deployment
│   ├── rbac.yaml                    # RBAC — ServiceAccounts + Roles
│   ├── network-policy.yaml          # Zero Trust Network Policies
│   └── argocd-app.yaml              # ArgoCD GitOps application
├── gcp/                             # GCP GKE manifests (Phase 10) ✅
│   ├── namespace.yaml               # Kubernetes namespace
│   ├── secrets.yaml                 # DB credentials as K8s Secrets
│   ├── mariadb-deployment.yaml      # MariaDB StatefulSet + PVC (standard)
│   ├── mariadb-service.yaml         # MariaDB Headless Service
│   ├── backend-deployment.yaml      # Spring Boot Deployment x2
│   ├── backend-service.yaml         # Backend ClusterIP Service
│   ├── frontend-deployment.yaml     # React + Nginx Deployment x2 (v3 image)
│   └── frontend-service.yaml        # Frontend LoadBalancer Service
├── screenshots/                     # Proof of deployment — all 3 clouds
└── README.md
```

---

## 🔄 Why Same Manifests for All Clouds?

```
Cloud-Agnostic Design:
  AWS EKS → kubectl apply -f aws/
  Azure AKS → kubectl apply -f azure/
  GCP GKE → kubectl apply -f gcp/

Only difference per cloud:
  storageClassName: gp2          (AWS EBS)
  storageClassName: managed-csi  (Azure CSI)
  storageClassName: standard     (GCP standard)

Everything else is IDENTICAL — write once, deploy anywhere!
```

---

## ☁️ Cloud Coverage

| Cloud | Cluster | K8s Version | Storage Class | Status |
|---|---|---|---|---|
| AWS EKS | studentsphere-cluster | v1.34 | gp2 (EBS) | ✅ Phase 2 Complete |
| Azure AKS | studentsphere-aks | v1.35.1 | managed-csi | ✅ Phase 9 Complete |
| GCP GKE | studentsphere-gke | v1.35.1-gke | standard | ✅ Phase 10 Complete |

---

## 🏗️ Architecture

```
Internet
    ↓
Cloud LoadBalancer (AWS ELB / Azure LB / GCP LB)
    ↓
Frontend Pods (React + Nginx x2)
    ↓  proxy_pass /api/ → backend-service:8080
Backend Pods (Spring Boot x2)
    ↓  JDBC → mariadb-service:3306
MariaDB StatefulSet (x1)
    ↓
Persistent Volume (5Gi — cloud specific storage)
```

---

## 📋 Kubernetes Resources

| Resource | Type | Replicas | Description |
|---|---|---|---|
| mariadb | StatefulSet | 1 | Persistent database with cloud volume |
| backend | Deployment | 2 | Spring Boot REST API |
| frontend | Deployment | 2 | React + Nginx (v3 image) |
| frontend-service | LoadBalancer | - | Public internet access |
| backend-service | ClusterIP | - | Internal cluster only |
| mariadb-service | Headless | - | StatefulSet DNS resolution |
| db-secret | Secret | - | Database credentials |
| backend-hpa | HPA | 2-5 | Auto-scale on CPU 70% / Memory 95% |
| frontend-hpa | HPA | 2-5 | Auto-scale on CPU 70% / Memory 80% |
| backend-canary | Deployment | 1 | Canary — 33% traffic to new version |
| backend-blue | Deployment | 2 | Blue environment (stable) |
| backend-green | Deployment | 2 | Green environment (new version) |
| backend-sa | ServiceAccount | - | RBAC identity for backend |
| frontend-sa | ServiceAccount | - | RBAC identity for frontend |

---

## ⚡ Phase 2 — Deploy on AWS EKS

### Prerequisites
```bash
aws --version
kubectl version --client
eksctl version
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

Expected output:
```
✔ EKS cluster "studentsphere-cluster" in "ap-south-1" region is ready
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
```
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
```

### Step 5 — Get App URL
```bash
kubectl get svc frontend-service -n studentsphere \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

### Output / Proof

#### All Kubernetes Resources Running
![kubectl get all](screenshots/01-kubectl-get-all.png)

#### Nodes Ready
![kubectl get nodes](screenshots/02-kubectl-get-nodes.png)

#### App Live on AWS EKS
![App on EKS](screenshots/03-app-on-eks.png)

#### Student Registered on EKS
![Student Registered](screenshots/04-student-registered-eks.png)

---

## ⚡ Phase 9 — Deploy on Azure AKS

### Prerequisites
```bash
# AKS cluster already created via Terraform
az aks get-credentials \
  --resource-group studentsphere-rg \
  --name studentsphere-aks \
  --overwrite-existing

kubectl get nodes
```

Expected output:
```
NAME                              STATUS   ROLES    AGE   VERSION
aks-default-xxxx-vmss000000       Ready    <none>   5m    v1.35.1
aks-default-xxxx-vmss000001       Ready    <none>   5m    v1.35.1
```

### Step 1 — Deploy All Resources
```bash
kubectl apply -f azure/namespace.yaml
kubectl apply -f azure/secrets.yaml
kubectl apply -f azure/mariadb-deployment.yaml
kubectl apply -f azure/mariadb-service.yaml
kubectl apply -f azure/backend-deployment.yaml
kubectl apply -f azure/backend-service.yaml
kubectl apply -f azure/frontend-deployment.yaml
kubectl apply -f azure/frontend-service.yaml
```

### Step 2 — Verify Pods Running
```bash
kubectl get pods -n studentsphere
```

Expected output:
```
NAME                        READY   STATUS    RESTARTS
backend-xxxx                1/1     Running   0
backend-xxxx                1/1     Running   0
frontend-xxxx               1/1     Running   0
frontend-xxxx               1/1     Running   0
mariadb-0                   1/1     Running   0
```

### Step 3 — Get External IP
```bash
kubectl get svc frontend-service -n studentsphere
```

Expected output:
```
NAME               TYPE           EXTERNAL-IP    PORT(S)
frontend-service   LoadBalancer   4.246.43.99    80:31969/TCP
```

### Output / Proof

#### Azure AKS Nodes Ready
![Azure Nodes](screenshots/05-azure-aks-nodes.png)

#### All Pods Running on Azure
![Azure Pods](screenshots/06-azure-aks-pods.png)

#### App Live on Azure AKS
![Azure App](screenshots/07-azure-app-running.png)

#### Student Registered on Azure
![Azure Student](screenshots/08-azure-student-registered.png)

---

## ⚡ Phase 10 — Deploy on GCP GKE

### Prerequisites
```bash
# GKE cluster already created via Terraform
gcloud container clusters get-credentials studentsphere-gke \
  --zone us-central1-a \
  --project YOUR_PROJECT_ID

kubectl get nodes
```

Expected output:
```
NAME                                                  STATUS   ROLES    AGE   VERSION
gke-studentsphere-gk-studentsphere-no-xxxx-xxxx       Ready    <none>   5m    v1.35.1-gke
gke-studentsphere-gk-studentsphere-no-xxxx-xxxx       Ready    <none>   5m    v1.35.1-gke
```

### Step 1 — Deploy All Resources
```bash
kubectl apply -f gcp/namespace.yaml
kubectl apply -f gcp/secrets.yaml
kubectl apply -f gcp/mariadb-deployment.yaml
kubectl apply -f gcp/mariadb-service.yaml
kubectl apply -f gcp/backend-deployment.yaml
kubectl apply -f gcp/backend-service.yaml
kubectl apply -f gcp/frontend-deployment.yaml
kubectl apply -f gcp/frontend-service.yaml
```

### Step 2 — Verify Pods Running
```bash
kubectl get pods -n studentsphere
```

Expected output:
```
NAME                        READY   STATUS    RESTARTS
backend-xxxx                1/1     Running   0
frontend-xxxx               1/1     Running   0
mariadb-0                   1/1     Running   0
```

> **Note:** On e2-medium nodes, scale to 1 replica if CPU insufficient:
> ```bash
> kubectl scale deployment backend --replicas=1 -n studentsphere
> kubectl scale deployment frontend --replicas=1 -n studentsphere
> ```

### Step 3 — Get External IP
```bash
kubectl get svc frontend-service -n studentsphere
```

Expected output:
```
NAME               TYPE           EXTERNAL-IP     PORT(S)
frontend-service   LoadBalancer   35.188.33.201   80:30385/TCP
```

### Output / Proof

#### GCP GKE Nodes Ready
![GCP Nodes](screenshots/09-gcp-gke-nodes.png)

#### All Pods Running on GCP
![GCP Pods](screenshots/10-gcp-gke-pods.png)

#### App Live on GCP GKE
![GCP App](screenshots/11-gcp-app-running.png)

#### Student Registered on GCP
![GCP Student](screenshots/12-gcp-student-registered.png)

---

## 🚀 Phase 5 — Advanced Kubernetes Features

### Feature 1 — HPA (Horizontal Pod Autoscaler)

#### What
Automatically scales pods up/down based on CPU and memory usage.

#### Why
```
Without HPA: Fixed 2 pods — cannot handle traffic spikes
With HPA:    Load increases → 2→5 pods auto-scale
             Load decreases → 5→2 pods auto-scale down
```

#### How
```bash
kubectl apply -f aws/backend-hpa.yaml
kubectl apply -f aws/frontend-hpa.yaml
kubectl get hpa -n studentsphere
```

Expected output:
```
NAME           REFERENCE             TARGETS                        MINPODS   MAXPODS   REPLICAS
backend-hpa    Deployment/backend    cpu: 0%/70%, memory: 78%/95%   2         5         2
frontend-hpa   Deployment/frontend   cpu: 1%/70%, memory: 4%/80%    2         5         2
```

---

### Feature 2 — Canary Deployment

#### What
Deploy new version to 33% of traffic — test before full rollout.

#### Why
```
Without Canary: v1 → v2 (100% users affected if bug exists)
With Canary:    v1 (2 pods, 67%) + v2 canary (1 pod, 33%) → test → full rollout
```

#### How
```bash
kubectl apply -f aws/backend-canary.yaml
kubectl get pods -n studentsphere --show-labels | grep canary
```

---

### Feature 3 — Blue-Green Deployment

#### What
Two identical environments — instant traffic switch with zero downtime.

#### Why
```
Blue  = Current stable version (receiving traffic)
Green = New version (tested in parallel)
Switch = kubectl patch → zero downtime!
Rollback = kubectl patch back → instant!
```

#### How
```bash
# Apply blue-green
kubectl apply -f aws/backend-blue-green.yaml

# Switch Blue → Green
kubectl patch svc backend-bg-service -n studentsphere \
  -p '{"spec":{"selector":{"app":"backend","version":"green"}}}'

# Rollback Green → Blue
kubectl patch svc backend-bg-service -n studentsphere \
  -p '{"spec":{"selector":{"app":"backend","version":"blue"}}}'
```

### Output / Proof

#### HPA Working
![HPA Working](screenshots/phase5/01-hpa-working.png)

#### Canary Deployment
![Canary Deployment](screenshots/phase5/02-canary-deployment.png)

#### Blue-Green Switch
![Blue-Green Switch](screenshots/phase5/02-blue-green-switch.png)

#### All Deployments Running
![All Deployments](screenshots/phase5/03-all-deployments.png)

---

## 🆚 Deployment Strategy Comparison

| Strategy | Downtime | Risk | Use Case |
|---|---|---|---|
| Rolling Update | Zero | Medium | Normal updates |
| Canary | Zero | Low | Test new version on 33% traffic |
| Blue-Green | Zero | Very Low | Instant switch with rollback |
| Recreate | Yes | High | Dev/test environments only |

---

## 🐛 Troubleshooting

### AWS Problems
```
Error: InvalidParameterCombination - instance type not eligible
Fix:   --node-type t3.small

Error: MariaDB Pod Pending — PVC not bound
Fix:   Install EBS CSI Driver + storageClassName: gp2

Error: Frontend CrashLoopBackOff — host not found
Fix:   nginx.conf proxy_pass → use K8s service name
       http://backend-service:8080 (not Docker Compose name)

Error: HPA shows <unknown> metrics
Fix:   Wait 2-3 minutes for metrics server
       kubectl get pods -n kube-system | grep metrics
```

### Azure Problems
```
Error: LoadBalancer External IP pending
Fix:   Assign Network Contributor role to AKS identity

Error: CORS error — old AWS IP in frontend
Fix:   Use frontend:v3 image (VITE_API_URL=/api)
       kubectl set image deployment/frontend frontend=studentsphere-frontend:v3

Error: mariadb-storage PVC pending
Fix:   storageClassName: managed-csi (not gp2)
```

### GCP Problems
```
Error: gke-gcloud-auth-plugin not found
Fix:   gcloud components install gke-gcloud-auth-plugin

Error: Insufficient CPU — pod pending
Fix:   kubectl scale deployment backend --replicas=1 -n studentsphere

Error: mariadb-storage PVC pending
Fix:   storageClassName: standard (not gp2)
```

---

## 🔗 Related Repositories

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
