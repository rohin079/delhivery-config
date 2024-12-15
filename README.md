
# Delhivery Configuration Repository
## Repository Structure and Setup

### Repository Organization
```plaintext
delhivery-config/
├── dev/
│   ├── frontend-deployment.yaml
│   ├── frontend-rollout.yaml
│   └── frontend-service.yaml
└── monitoring/
    ├── servicemonitor.yaml
    └── grafana-dashboard.json
```

### Key Components

#### 1. Rollout Configuration (`dev/frontend-rollout.yaml`)
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
          resources:
            requests:
              cpu: 500m
              memory: 0.5Gi
            limits:
              cpu: 1000m
              memory: 1Gi
          ports:
            - containerPort: 8082
```

#### 2. Service Configuration (`dev/frontend-service.yaml`)
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

#### 3. Monitoring Configuration (`monitoring/servicemonitor.yaml`)
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

### CI/CD Integration

The configuration repository is automatically updated by the GitHub Actions workflow in the main application repository. When a new image is built and pushed, the workflow:

1. Clones this config repository
2. Updates the image tag in rollout configuration
3. Commits and pushes the changes

Example workflow step:
```yaml
- name: Update deployment configuration
  run: |
    yq e -i '.spec.template.spec.containers[0].image = "rohin07/delhivery:1.1.1-${{ github.run_number }}"' ./delhivery-config/dev/frontend-rollout.yaml
```

### Monitoring Setup

The monitoring configuration includes:
- ServiceMonitor for Prometheus metrics collection
- Grafana dashboard configuration for visualization
- Target endpoints for metrics scraping

### Usage Instructions

1. Clone the repository:
```bash
git clone https://github.com/rohin079/delhivery-config.git
```

2. Apply configurations:
```bash
# Apply rollout
kubectl apply -f dev/frontend-rollout.yaml

# Apply service
kubectl apply -f dev/frontend-service.yaml

# Apply monitoring
kubectl apply -f monitoring/servicemonitor.yaml
```

3. Verify the deployment:
```bash
kubectl argo rollouts get rollout delhivery-rollouts
```

### Best Practices

1. **Version Control**
   - All changes should be made through pull requests
   - Maintain clear commit messages
   - Tag significant versions

2. **Configuration Management**
   - Keep configurations environment-specific (dev/, prod/, etc.)
   - Use consistent naming conventions
   - Document all configuration parameters

3. **Security**
   - No sensitive data in configuration files
   - Use Kubernetes secrets for sensitive information
   - Regular security reviews of configurations

### Troubleshooting

Common issues and solutions:
1. **Image Pull Errors**
   - Verify image tag in rollout configuration
   - Check Docker Hub credentials
   - Confirm image exists in registry

2. **Metrics Collection Issues**
   - Verify ServiceMonitor labels match
   - Check Prometheus target configuration
   - Confirm metrics endpoint accessibility

3. **Rollout Issues**
   - Check rollout status with `kubectl argo rollouts get rollout`
   - Verify service selector matches pod labels
   - Review rollout events with `kubectl describe rollout`
