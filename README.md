# DevOps Setup: EKS + Ansible + GitHub Actions

Complete DevOps pipeline for provisioning AWS EKS cluster and deploying applications with monitoring.

## Architecture

- **Infrastructure**: AWS EKS cluster provisioned via Ansible
- **Application**: Apache HTTP Server deployed on Kubernetes
- **Monitoring**: Prometheus + Grafana for metrics and dashboards
- **CI/CD**: GitHub Actions for automated deployment

## Prerequisites

1. AWS Account with AdministratorAccess permissions
2. GitHub repository with the following secrets:
   - `AWS_ACCESS_KEY_ID`
   - `AWS_SECRET_ACCESS_KEY`

## Quick Start

### 1. Setup GitHub Secrets

Go to your repository → Settings → Secrets and variables → Actions, and add:

```
AWS_ACCESS_KEY_ID: your-aws-access-key
AWS_SECRET_ACCESS_KEY: your-aws-secret-key
```

### 2. Run Infrastructure Pipeline

The infrastructure pipeline will automatically trigger on:
- Push to `main` branch (if `ansible/` files changed)
- Manual trigger via GitHub Actions UI

Or trigger manually:
1. Go to Actions tab in your GitHub repository
2. Select "Infrastructure Pipeline"
3. Click "Run workflow"

This will:
- Create VPC, subnets, security groups
- Provision EKS cluster with managed node groups
- Configure IAM roles
- Generate and store kubeconfig

### 3. Run Application Pipeline

The application pipeline will automatically trigger after infrastructure pipeline completes, or:
- Push to `main` branch (if `k8s/` files changed)
- Manual trigger via GitHub Actions UI

This will:
- Deploy Apache HTTP Server
- Install Prometheus for metrics
- Install Grafana for dashboards
- Output service URLs

## Accessing Services

After successful deployment, check the GitHub Actions logs for service URLs:

### Apache Web Server
- URL will be displayed in pipeline output
- Format: `http://<load-balancer-dns>`
- Shows default Apache welcome page

### Prometheus
- URL will be displayed in pipeline output  
- Format: `http://<prometheus-load-balancer-dns>`
- Access metrics and targets

### Grafana
- URL will be displayed in pipeline output
- Format: `http://<grafana-load-balancer-dns>`
- **Username**: `admin`
- **Password**: `admin123`
- Pre-configured with Prometheus datasource

## Manual Commands

If you want to run locally:

### Deploy Infrastructure
```bash
cd ansible
ansible-playbook -i inventory/hosts.ini site.yml -e ansible_account_id=YOUR_ACCOUNT_ID
```

### Deploy Applications
```bash
# Configure kubectl
aws eks update-kubeconfig --region us-west-2 --name devops-eks-cluster

# Deploy Apache
kubectl apply -f k8s/apache/

# Install monitoring
kubectl create namespace monitoring
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm install prometheus prometheus-community/prometheus -n monitoring --set server.service.type=LoadBalancer
helm install grafana grafana/grafana -n monitoring --set service.type=LoadBalancer --set adminPassword=admin123
```

### Get Service URLs
```bash
kubectl get svc apache-service
kubectl get svc prometheus-server -n monitoring  
kubectl get svc grafana -n monitoring
```

## Cleanup

To destroy the infrastructure:

```bash
# Delete Kubernetes resources
kubectl delete -f k8s/apache/
helm uninstall prometheus -n monitoring
helm uninstall grafana -n monitoring
kubectl delete namespace monitoring

# Delete EKS cluster (via AWS Console or CLI)
aws eks delete-nodegroup --cluster-name devops-eks-cluster --nodegroup-name worker-nodes
aws eks delete-cluster --name devops-eks-cluster
```

## File Structure

```
.
├── ansible/
│   ├── site.yml                    # Main playbook
│   ├── group_vars/all.yml          # Configuration variables
│   ├── inventory/hosts.ini         # Inventory file
│   ├── requirements.yml            # Ansible collections
│   └── roles/eks/tasks/main.yml    # EKS provisioning tasks
├── k8s/
│   └── apache/
│       ├── deployment.yaml         # Apache deployment
│       └── service.yaml           # Apache LoadBalancer service
├── .github/workflows/
│   ├── infra-pipeline.yml         # Infrastructure CI/CD
│   └── app-pipeline.yml           # Application CI/CD
└── README.md
```

## Key Features

- **Clean, minimal code** - No debugging steps or complex logic
- **Error handling** - Handles duplicate resources gracefully
- **High availability** - Multi-AZ deployment
- **Security** - Worker nodes in private subnets
- **Monitoring** - Built-in Prometheus and Grafana
- **Automation** - Complete CI/CD pipeline