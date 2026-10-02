# EKS Cluster GitOps Bootstrap Repository

> **Repository 3 in the Standard 3-Repo Production Model:**
> 1. `App Repo`: Source code + CI pipeline (Build, Unit Test, Trivy, Push OCI Image).
> 2. `Infra Repo (IaC)`: Terraform VPC, EKS Cluster, Node Groups, IAM Roles for Service Accounts (IRSA).
> 3. **`GitOps Config Repo` (This Repo)**: ArgoCD App-of-Apps, Sync Waves, Observability, Policy-as-Code, and Application Delivery.

---

## 1. Directory Structure

```text
.
├── apps/                               # ArgoCD Application manifests (App-of-Apps pattern)
│   ├── root-app.yaml                   # [Wave -10] Master App syncing the apps/ directory
│   ├── wave--2-storage.yaml            # [Wave -2]  Default gp3 StorageClass (EBS CSI)
│   ├── wave--1-essentials.yaml         # [Wave -1]  Metrics Server (HPA & kubectl top)
│   ├── wave-0-kyverno.yaml             # [Wave 0]   Kyverno Admission Controller
│   ├── wave-0-kyverno-policies.yaml    # [Wave 0]   Kyverno Security & Capacity Policies
│   ├── wave-1-monitoring.yaml          # [Wave 1]   kube-prometheus-stack (Prometheus + Grafana)
│   ├── wave-1-logging.yaml             # [Wave 1]   loki-stack (Loki + Promtail DaemonSet)
│   └── wave-2-app.yaml                 # [Wave 2]   MyBlogSpot Application Helm Workload
│
├── bootstrap/                          # Manifests & tuned values organized by Wave
│   ├── wave--2-storage/
│   │   └── default-storageclass.yaml   # gp3 default StorageClass definition
│   ├── wave--1-essentials/
│   │   └── metrics-server.yaml         # Metrics Server official deployment
│   ├── wave-0-policy/
│   │   ├── cluster-policies.yaml       # disallow-root, require-limits, disallow-latest-tag
│   │   └── kyverno-values.yaml         # Controller resource limits & 1 replica tuning
│   ├── wave-1-monitoring/
│   │   └── prometheus-values.yaml      # Prometheus, Grafana, Alertmanager tuned values
│   └── wave-1-logging/
│       └── loki-values.yaml            # Loki SingleBinary gp3 PVC + Promtail values
│
├── helm/
│   └── myblogspot/                     # MyBlogSpot Helm Chart (Backend, Frontend, Ingress)
│       ├── Chart.yaml
│       ├── values.yaml
│       ├── values-staging.yaml
│       └── values-prod.yaml
│
└── legacy/                             # Preserved initial manifests for reference
```

---

## 2. ArgoCD Sync Wave Sequence

ArgoCD Sync Waves guarantee that dependencies are healthy before dependent resources are provisioned.

| Wave | Component | Namespace | Purpose / Dependency Requirement |
| :---: | :--- | :--- | :--- |
| **`-10`** | `root-bootstrap-app` | `argocd` | **App-of-Apps Master Controller**. Synchronizes the `apps/` directory to deploy all child apps. |
| **`-2`** | `wave--2-storage` | `kube-system` | **gp3 StorageClass**. Must exist first so any subsequent StatefulSets (Prometheus, Loki, Alertmanager, Grafana) can claim EBS volumes. |
| **`-1`** | `wave--1-essentials` | `kube-system` | **Metrics Server**. Enables `metrics.k8s.io` API for Horizontal Pod Autoscalers (HPA) and `kubectl top nodes/pods`. |
| **`0`** | `wave-0-kyverno` | `kyverno` | **Admission Controller**. Mutating and Validating Webhooks intercepting Kubernetes API requests. Tuned to 1 replica to save memory. |
| **`0`** | `wave-0-kyverno-policies` | `kyverno` | **Security Guardrails**. Enforces `runAsNonRoot: true`, container resource limits, and forbids `:latest` image tags. Configured in `Audit` mode with system namespaces excluded. |
| **`1`** | `wave-1-monitoring` | `monitoring` | **Prometheus + Grafana + Alertmanager**. Scrapes cluster metrics, stores 7 days on 10Gi gp3 PVC. Automatically links Loki as datasource. |
| **`1`** | `wave-1-logging` | `monitoring` | **Grafana Loki + Promtail DaemonSet**. Promtail scrapes node `/var/log/pods`, ships to Loki SingleBinary stored on 10Gi gp3 PVC. |
| **`2`** | `wave-2-myblogspot` | `staging` | **Application Delivery**. Deploys the business workload (Backend Go API, Frontend React, Ingress) protected by Kyverno and monitored by Prometheus/Loki. |

---

## 3. Tool Selection & Architecture Rationale

### Why Grafana Loki over AWS CloudWatch Logs?
1. **Cost Efficiency (85%+ Savings)**:
   - *AWS CloudWatch*: Ingestion is \$0.50/GB + storage \$0.03/GB + query fees. For a small cluster, CloudWatch log ingestion can easily exceed \$50–\$100/month.
   - *Grafana Loki*: Ingestion is free in-cluster. Storage resides on AWS EBS `gp3` (\$0.08/GB) or S3 (\$0.023/GB), costing mere pennies per month.
2. **Unified Observability (Metrics + Logs Correlation)**:
   - In Grafana, developers can seamlessly split-screen view: jump directly from a Prometheus CPU/Error-Rate spike to the exact Loki log lines at that timestamp using LogQL.
3. **Optimized Footprint**:
   - Loki runs in `SingleBinary` mode with a 7-day retention period (`168h`), requiring only ~150MiB RAM and 10Gi storage.

### Excluded Redundancies (Keeping the Cluster Minimal & Clean)
To operate efficiently within a 2-node `t3.medium` cluster (8GB total RAM) without OOM crashes, redundant tools from typical bloated boilerplates were deliberately excluded:
- ❌ **Traefik Ingress**: Excluded. Redundant with AWS Load Balancer Controller (ALB) and native Ingress.
- ❌ **VictoriaMetrics**: Excluded. Redundant with Prometheus Operator (`kube-prometheus-stack`), which is the industry standard and natively integrated with Grafana dashboards.
- ❌ **Sealed Secrets**: Excluded. Modern Kubernetes standard uses **External Secrets Operator (ESO)** synchronizing with AWS Secrets Manager or KMS.
- ❌ **Velero**: Excluded. EBS CSI volume snapshots and GitOps repo state provide full cluster reproducibility without needing full-cluster backup daemons in staging.

---

## 4. Quickstart Guide

### Step 1: Set Your Remote Git Repository
When you are ready to push this repo to GitHub/GitLab:
```bash
# Add your remote
git remote add origin git@github.com:<YOUR_USERNAME>/<YOUR_REPO_NAME>.git
git branch -M main
git push -u origin main
```

### Step 2: Update the repoURL in App Manifests
Update the `repoURL` in `apps/*.yaml` to point to your new remote URL:
```bash
# Replace with your actual repository URL
sed -i 's|https://github.com/gryffin-uit-alpha/cluster-bootstrap.git|git@github.com:<YOUR_USERNAME>/<YOUR_REPO_NAME>.git|g' apps/*.yaml
```

### Step 3: Apply the Master Root App
Deploy the root application into your EKS cluster:
```bash
kubectl apply -f apps/root-app.yaml
```
ArgoCD will automatically discover all applications in `apps/` and orchestrate the bootstrap sequence in strict Sync Wave order!
