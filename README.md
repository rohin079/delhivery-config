
# Canary Deployment in Kubernetes using Argo Rollouts

**Submitted by:** Rohin Mehrotra 
**For:** Delhivery DevOps Intern Assignment
## Project Documentation

Welcome to the documentation for implementing canary deployments using Argo Rollouts! This guide details the process of setting up progressive delivery for applications in a Kubernetes environment using Argo Rollouts and monitoring with Prometheus and Grafana.

Project Repositories:
- Application: [Delhivery-DevOps-Assignment](https://github.com/rohin079/Delhivery-DevOps-Assignment.git)
- Configuration: [delhivery-config](https://github.com/rohin079/delhivery-config)

## Table of Contents
1. [Setup and Configuration](#setup-and-configuration)
2. [CI/CD Implementation](#cicd-implementation)
3. [Canary Deployment](#canary-deployment)
4. [Monitoring Setup](#monitoring-setup)

## 1. Setup and Configuration


### 1.1 Cluster Setup

First, create a `kind-config.yaml` file for a three-node cluster configuration:
```yaml
# Three node cluster config (1 control-plane, 2 workers)
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  # Master/control-plane node
  - role: control-plane
  # Worker nodes
  - role: worker
  - role: worker
```

Create the cluster using the configuration:
```bash
# Create Kind cluster using the config file
kind create cluster --name delhiverycluster --config kind-config.yaml

# Verify the cluster nodes
kubectl get nodes
```

This setup creates:
- One control-plane node for cluster management
- Two worker nodes for running applications and workloads

### 1.2 Install Argo Rollouts
```bash
# Create namespace
kubectl create namespace argo-rollouts

# Install Argo Rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# Access Dashboard
kubectl argo rollouts dashboard
```

### 1.3 Install Monitoring Stack
```bash
# Create namespace
kubectl create namespace prometheus

# Add Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install Prometheus and Grafana
helm install prometheus prometheus-community/kube-prometheus-stack -n prometheus
```

## 2. CI/CD Implementation

### 2.1 GitHub Actions Workflow
```yaml
name: Python application
on:
  push:
    branches: ["main"]
  pull_request:
    branches: ["main"]

permissions:
  contents: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python 3.10
        uses: actions/setup-python@v5
        with:
          python-version: "3.10"
          
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install flake8 pytest
          if [ -f requirements.txt ]; then pip install -r requirements.txt; fi
          
      - name: Lint with flake8
        run: |
          # stop the build if there are Python syntax errors or undefined names
          flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics
          # exit-zero treats all errors as warnings
          flake8 . --count --exit-zero --max-complexity=10 --max-line-length=127 --statistics

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
        
      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: rohin07
          password: ${{ secrets.DOCKERHUB_PASSWORD }}
          
      - name: Build and Push Docker Image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: rohin07/delhivery:1.1.1-${{github.run_number}}

      - name: Checkout delhivery-config
        uses: actions/checkout@v4
        with:
          repository: rohin079/delhivery-config
          token: ${{ secrets.REPO_TOKEN }}
          path: ./delhivery-config
          
      - name: Update deployment YAML
        run: |
          sudo wget https://github.com/mikefarah/yq/releases/download/v4.40.5/yq_linux_amd64 -O /usr/local/bin/yq
          sudo chmod +x /usr/local/bin/yq
          yq e -i '.spec.template.spec.containers[0].image = "rohin07/delhivery:1.1.1-${{ github.run_number }}"' ./delhivery-config/dev/frontend-deployment.yaml
          yq e -i '.spec.template.spec.containers[0].image = "rohin07/delhivery:1.1.1-${{ github.run_number }}"' ./delhivery-config/dev/frontend-rollout.yaml
          
      - name: Commit and Push changes to config files
        run: |
          cd ./delhivery-config
          git add .
          git config user.name "github-actions[bot]"
          git config user.email "github-actions@github.com"
          git commit -am "updated: build number for frontend image"
          git push
```

## 3. Canary Deployment

### 3.1 Rollout Configuration
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: delhivery-rollouts
spec:
  replicas: 5
  strategy:
    canary:
      steps:
        - setWeight: 20
        - pause: {}
        - setWeight: 40
        - pause: {}
        - setWeight: 60
        - pause: {}
        - setWeight: 80
        - pause: {}
  selector:
    matchLabels:
      app: rollouts-demo
  template:
    metadata:
      labels:
        app: rollouts-demo
    spec:
      containers:
        - name: delhivery-rollout
          image: rohin07/delhivery:1.1.1-8
          ports:
            - containerPort: 8082
```

### 3.2 Service Configuration
```yaml
apiVersion: v1
kind: Service
metadata:
  name: delhivery-rollouts
  labels:
    app: rollouts-demo
spec:
  ports:
    - name: metrics
      port: 8082
      targetPort: 8082
  selector:
    app: rollouts-demo
```

## 4. Monitoring Setup

### 4.1 ServiceMonitor Configuration
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: argo-rollouts-metrics
  namespace: prometheus
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: argo-rollouts-metrics
  namespaceSelector:
    matchNames:
      - argo-rollouts
  endpoints:
    - port: metrics
```

### 4.2 Key Commands

Monitor Rollouts:
```bash
# Watch rollout status
kubectl argo rollouts get rollout delhivery-rollouts --watch

# Access dashboards
kubectl port-forward svc/prometheus-operated -n prometheus 9090:9090
kubectl port-forward svc/prometheus-grafana -n prometheus 3000:80
```

Manage Rollouts:
```bash
# Promote rollout
kubectl argo rollouts promote delhivery-rollouts

# Abort rollout
kubectl argo rollouts abort delhivery-rollouts

# Rollback
kubectl argo rollouts undo delhivery-rollouts
```

## Project Architecture

[Space for architecture diagram]

## CI/CD Pipeline Flow

[Space for CI/CD flow diagram]

## Conclusion
This implementation demonstrates:
- Automated canary deployments using Argo Rollouts
- Progressive delivery with traffic splitting
- Real-time monitoring with Prometheus and Grafana
- Automated CI/CD pipeline using GitHub Actions
- GitOps-based configuration management
