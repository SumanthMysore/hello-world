# Hello World — Bootcamp Project

A fully containerised Node.js application deployed to Amazon EKS using Helm, with infrastructure provisioned by Terraform and monitored by Prometheus and Grafana. A GitHub Actions pipeline automates the build-and-deploy cycle on every push.

---

## Repository Structure

```
.
├── hello-world/                  # Node.js application
│   ├── index.js                  # Express server
│   ├── package.json
│   ├── Dockerfile
│   └── .dockerignore
├── Helm/
│   └── hello-world/              # Helm chart
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│           ├── deployment.yaml
│           ├── service.yaml
│           ├── servicemonitor.yaml   # Prometheus scrape config
│           ├── serviceaccount.yaml
│           ├── ingress.yaml
│           ├── hpa.yaml
│           └── tests/
│               └── test-connection.yaml
├── IaC/                          # Terraform — VPC lookup + EKS cluster
│   ├── versions.tf
│   ├── variables.tf
│   ├── vpc.tf
│   ├── vpc.variables.tf
│   ├── eks.tf
│   ├── eks.variables.tf
│   └── iam.tf
└── .github/
    └── workflows/
        ├── hello-world-release.yaml      # Build + deploy pipeline
        ├── hello-world-deployment.yaml   # Alternate deploy workflow
        └── monitoring-setup.yaml         # One-time Prometheus + Grafana install
```

---

## Prerequisites

| Tool | Version |
|------|---------|
| Terraform | >= 1.5 |
| AWS CLI | >= 2 |
| kubectl | >= 1.28 |
| Helm | >= 3.12 |
| Node.js | >= 20 (local dev only) |
| Docker | any recent version |

AWS credentials must be configured (`aws configure` or environment variables).

---

## 1. Infrastructure (IaC)

Terraform provisions an EKS cluster and managed node group into an **existing** VPC. The VPC is looked up by name tag; it is not created by this code.

### Architecture

```
Existing VPC  ──►  Public subnets (tagged kubernetes.io/role/elb=1)
                        │
                   EKS Cluster  (Kubernetes 1.33)
                        │
                   Managed Node Group  (AL2023, auto-scaling 1–3)
```

### Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `region` | AWS region | _(required)_ |
| `vpc_name` | Name tag of the existing VPC | _(required)_ |
| `cluster_name` | EKS cluster name | _(required)_ |
| `cluster_version` | Kubernetes version | `1.33` |
| `node_group_name` | Node group name | `default` |
| `node_instance_type` | EC2 instance type | _(required)_ |
| `node_ami_type` | AMI type | `AL2023_x86_64_STANDARD` |
| `node_desired_size` | Desired node count | `2` |
| `node_min_size` | Minimum node count | `1` |
| `node_max_size` | Maximum node count | `3` |

### Usage

```bash
cd IaC

# Create a tfvars file with your values
cat > terraform.tfvars <<EOF
region             = "us-east-1"
vpc_name           = "my-vpc"
cluster_name       = "my-eks-cluster"
node_instance_type = "t3.medium"
EOF

terraform init
terraform plan
terraform apply
```

After apply, configure kubectl:

```bash
aws eks update-kubeconfig --region <region> --name <cluster_name>
kubectl get nodes
```

---

## 2. Application

A minimal Node.js Express server at `hello-world/`.

### Endpoints

| Endpoint | Description |
|----------|-------------|
| `GET /` | Returns `{ message: "Hello World!", version: "1.0.0" }` |
| `GET /health` | Returns `{ status: "ok" }` — used by Kubernetes liveness and readiness probes |
| `GET /metrics` | Prometheus metrics — default Node.js metrics + `http_requests_total` counter |

### Run Locally

```bash
cd hello-world
npm install
npm start
# Server starts on http://localhost:3000
```

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `PORT` | `3000` | Port the server listens on |

### Build Docker Image

```bash
cd hello-world
docker build -t hello-world:local .
docker run -p 3000:3000 hello-world:local
```

---

## 3. Helm Chart

The chart lives at `Helm/hello-world/` and deploys the application to Kubernetes.

### Key Values (`values.yaml`)

| Value | Default | Description |
|-------|---------|-------------|
| `image.repository` | `ghcr.io/GHCR_USERNAME/hello-world` | Container image (overridden by pipeline) |
| `image.tag` | `latest` | Image tag (overridden by pipeline with git SHA) |
| `replicaCount` | `1` | Number of pod replicas |
| `service.type` | `ClusterIP` | Kubernetes service type |
| `service.port` | `3000` | Service port |
| `ingress.enabled` | `false` | Enable ingress (disabled by default) |
| `serviceMonitor.enabled` | `true` | Creates a Prometheus ServiceMonitor |
| `autoscaling.enabled` | `false` | HPA (disabled by default) |

### Manual Deploy

```bash
helm upgrade --install hello-world ./Helm/hello-world \
  --namespace default \
  --set image.repository=ghcr.io/<your-github-username>/hello-world \
  --set image.tag=<git-sha> \
  --wait
```

### Access the App (port-forward)

```bash
kubectl port-forward svc/hello-world 3000:3000
curl http://localhost:3000
```

### Run Helm Tests

```bash
helm test hello-world
```

---

## 4. CI/CD Pipeline

### Workflows

#### `hello-world-release.yaml` — Build & Deploy
Triggers on push to the `deployment` branch when files under `hello-world/` or `Helm/hello-world/` change.

```
push → deployment branch
         │
    ┌────▼────┐
    │  build  │  docker build → push to GHCR (tagged with git SHA + latest)
    └────┬────┘
         │
    ┌────▼────┐
    │ deploy  │  helm upgrade --install (image tag = git SHA)
    └─────────┘
```

#### `monitoring-setup.yaml` — Prometheus + Grafana
A one-time manual workflow (`workflow_dispatch`). Installs `kube-prometheus-stack` into the `monitoring` namespace.

### GitHub Secrets Required

| Secret | Description |
|--------|-------------|
| `GHCR_USERNAME` | GitHub username for GHCR |
| `GHCR_PAT` | GitHub PAT with `write:packages` scope |
| `AWS_ACCESS_KEY_ID` | AWS access key |
| `AWS_SECRET_ACCESS_KEY` | AWS secret key |
| `EKS_CLUSTER_REGION` | AWS region (e.g. `us-east-1`) |
| `EKS_CLUSTER_NAME` | Name of the EKS cluster |

### Making a Deployment

1. Push changes to any file under `hello-world/` or `Helm/hello-world/` on the `deployment` branch.
2. The pipeline builds the Docker image, pushes it to GHCR, and deploys it to EKS via Helm.

---

## 5. Monitoring (Prometheus + Grafana)

### Install the Stack

Run the **Monitoring Setup** workflow manually from the GitHub Actions UI (Actions → Monitoring Setup → Run workflow), or install directly:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm upgrade --install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
  --wait
```

The `serviceMonitorSelectorNilUsesHelmValues=false` flag is required so Prometheus picks up the `ServiceMonitor` created by the hello-world chart.

### Access Grafana

```bash
# Get the default admin password
kubectl get secret -n monitoring monitoring-grafana \
  -o jsonpath="{.data.admin-password}" | base64 --decode

# Forward Grafana to localhost
kubectl port-forward -n monitoring svc/monitoring-grafana 3001:80
# Open http://localhost:3001  (admin / <password from above>)
```

### Access Prometheus

```bash
kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090:9090
# Open http://localhost:9090
```

### App Metrics

The app exposes Prometheus metrics at `/metrics`. Useful queries in Grafana:

```promql
# Total HTTP requests by route
sum by (route, status) (http_requests_total)

# Request rate over 1 minute
rate(http_requests_total[1m])

# Node.js heap memory
nodejs_heap_size_used_bytes
```
