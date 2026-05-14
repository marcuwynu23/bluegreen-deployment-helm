# Blue-Green Deployment Helm Chart

A production-ready Helm chart for blue-green deployment using Argo Rollouts. This chart deploys a backend (Express TypeScript) and frontend (Vite) application with automated blue-green deployment strategies.

## Features

- **Blue-Green Deployment**: Zero-downtime deployments using Argo Rollouts
- **Multi-Environment Support**: Separate configurations for staging and production
- **Horizontal Pod Autoscaling**: Automatic scaling based on CPU and memory metrics
- **Ingress with TLS**: Configurable ingress with TLS support
- **Analysis Templates**: Pre/post deployment analysis for success rate and response time
- **ArgoCD Integration**: GitOps-ready with ArgoCD Application manifests

## Prerequisites

- Kubernetes cluster (v1.20+)
- Helm 3.x
- Argo Rollouts installed on your cluster
- ArgoCD (optional, for GitOps deployment)
- NGINX Ingress Controller (for ingress support)
- Prometheus (optional, for analysis metrics)

### Install Argo Rollouts

```bash
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
```

### Install ArgoCD (Optional)

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

## Quick Start

### Option 1: Deploy with Helm

#### 1. Add the Helm Repository (if publishing)

```bash
helm repo add bluegreen-deployment-helm https://your-repo.github.io/helm-charts
helm repo update
```

#### 2. Install the Chart

**Default deployment:**
```bash
helm install bluegreen-deployment-helm . -n bluegreen --create-namespace
```

**Staging deployment:**
```bash
helm install bluegreen-deployment-helm . -n bluegreen-staging --create-namespace -f values/values.staging.yaml
```

**Production deployment:**
```bash
helm install bluegreen-deployment-helm . -n bluegreen-production --create-namespace -f values/values.production.yaml
```

#### 3. Verify Installation

```bash
kubectl get rollouts -n bluegreen
kubectl get services -n bluegreen
kubectl get pods -n bluegreen
```

### Option 2: Deploy with ArgoCD

#### 1. Update the Application Manifest

Edit `argocd/application.yaml` and update the `repoURL` to point to your actual Git repository:

```yaml
source:
  repoURL: https://github.com/your-username/bluegreen-deploy-helm
  targetRevision: HEAD
  path: .
```

#### 2. Apply the Application

```bash
kubectl apply -f argocd/application.yaml
```

#### 3. Access ArgoCD UI

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Visit `https://localhost:8080` and login with the initial password:
```bash
kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server -o name | cut -d'/' -f2
argocd admin initial-password -n argocd <pod-name>
```

## Blue-Green Deployment Workflow

### Understanding Blue-Green Strategy

This chart uses Argo Rollouts blue-green strategy which:
1. Deploys the new version alongside the existing version
2. Routes traffic to the preview service for testing
3. Runs analysis checks on the preview deployment
4. Promotes the new version to active service after successful analysis
5. Scales down the old version after a delay

### Manual Deployment Steps

#### 1. Deploy New Version

```bash
helm upgrade bluegreen-deployment-helm . -n bluegreen --set backend.image.tag=new-tag --set frontend.image.tag=new-tag
```

#### 2. Check Rollout Status

```bash
kubectl argo rollouts get rollout backend-rollout -n bluegreen
kubectl argo rollouts get rollout frontend-rollout -n bluegreen
```

#### 3. Promote to Active (if auto-promotion is disabled)

```bash
kubectl argo rollouts promote backend-rollout -n bluegreen
kubectl argo rollouts promote frontend-rollout -n bluegreen
```

#### 4. Access Preview Service

```bash
kubectl port-forward svc/backend-preview-service -n bluegreen 3000:3000
kubectl port-forward svc/frontend-preview-service -n bluegreen 8080:80
```

### Automatic Deployment

For environments with `autoPromotionEnabled: true` (staging by default), the rollout will automatically promote after successful analysis.

## Configuration

### Values Reference

| Parameter | Description | Default |
|-----------|-------------|---------|
| `global.namespace` | Default namespace | `default` |
| `backend.image.repository` | Backend image repository | `ghcr.io/marcuwynu23/express-typescript-sample` |
| `backend.image.tag` | Backend image tag | `sha-7f74f7b` |
| `backend.replicaCount` | Number of backend replicas | `3` |
| `backend.service.port` | Backend service port | `3000` |
| `backend.rollout.autoPromotionEnabled` | Auto-promote backend | `false` |
| `frontend.image.repository` | Frontend image repository | `ghcr.io/marcuwynu23/vite-app-sample` |
| `frontend.image.tag` | Frontend image tag | `sha-a6af942` |
| `frontend.replicaCount` | Number of frontend replicas | `2` |
| `frontend.ingress.enabled` | Enable ingress | `true` |
| `frontend.hpa.enabled` | Enable HPA | `true` |
| `namespace.name` | Namespace name | `bluegreen` |
| `namespace.create` | Create namespace | `true` |

### Custom Values

Create a custom values file:

```yaml
# custom-values.yaml
backend:
  image:
    tag: v2.0.0
  replicaCount: 5
  rollout:
    autoPromotionEnabled: false

frontend:
  image:
    tag: v2.0.0
  ingress:
    hosts:
      - host: myapp.example.com
        paths:
          - path: /
            pathType: Prefix
```

Deploy with custom values:
```bash
helm install bluegreen-deployment-helm . -n bluegreen --create-namespace -f custom-values.yaml
```

## Monitoring and Analysis

### View Rollout Status

```bash
kubectl argo rollouts get rollout backend-rollout -n bluegreen --watch
kubectl argo rollouts get rollout frontend-rollout -n bluegreen --watch
```

### View Rollout History

```bash
kubectl argo rollouts history backend-rollout -n bluegreen
kubectl argo rollouts history frontend-rollout -n bluegreen
```

### Rollback to Previous Version

```bash
kubectl argo rollouts undo backend-rollout -n bluegreen
kubectl argo rollouts undo frontend-rollout -n bluegreen
```

### Check Analysis Runs

```bash
kubectl get analysisruns -n bluegreen
kubectl describe analysisrun <analysisrun-name> -n bluegreen
```

## Accessing the Application

### Via Ingress

If ingress is enabled, access the application via the configured host:
- Production: `https://app.production.example.com`
- Staging: `https://app.staging.example.com`

### Via Port Forward

```bash
# Frontend
kubectl port-forward svc/frontend-service -n bluegreen 8080:80

# Backend
kubectl port-forward svc/backend-service -n bluegreen 3000:3000
```

## Troubleshooting

### Rollout Stuck in Progress

```bash
# Check rollout status
kubectl argo rollouts get rollout backend-rollout -n bluegreen

# Check events
kubectl describe rollout backend-rollout -n bluegreen

# Force promote if needed
kubectl argo rollouts promote backend-rollout -n bluegreen
```

### Analysis Failing

```bash
# Check analysis run logs
kubectl logs -n bluegreen -l app.kubernetes.io/name=backend --tail=100

# Verify Prometheus is accessible
kubectl get svc -n prometheus

# Check analysis template configuration
kubectl get analysistemplate success-rate -n argocd -o yaml
```

### Pods Not Starting

```bash
# Check pod status
kubectl get pods -n bluegreen

# Check pod logs
kubectl logs -n bluegreen <pod-name>

# Describe pod for events
kubectl describe pod <pod-name> -n bluegreen
```

### Image Pull Errors

```bash
# Create image pull secret (if using private registry)
kubectl create secret docker-registry regcred \
  --docker-server=ghcr.io \
  --docker-username=YOUR_USERNAME \
  --docker-password=YOUR_TOKEN \
  -n bluegreen

# Update values to use the secret
# Add to values.yaml:
imagePullSecrets:
  - name: regcred
```

## Advanced Usage

### Custom Analysis Templates

Create custom analysis templates for your specific metrics:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: custom-analysis
  namespace: argocd
spec:
  args:
  - name: service-name
  metrics:
  - name: custom-metric
    interval: 1m
    count: 5
    successCondition: result[0] >= 0.90
    provider:
      prometheus:
        address: http://prometheus-server.prometheus.svc.cluster.local
        query: |
          your_custom_prometheus_query
```

### Multiple Environments

Deploy to multiple environments using different value files:

```bash
# Development
helm install bluegreen-dev . -n bluegreen-dev -f values/values.dev.yaml

# Staging
helm install bluegreen-staging . -n bluegreen-staging -f values/values.staging.yaml

# Production
helm install bluegreen-prod . -n bluegreen-production -f values/values.production.yaml
```

### Canary Deployment (Alternative Strategy)

To use canary instead of blue-green, modify the rollout strategy in values:

```yaml
backend:
  rollout:
    strategy: canary
    canary:
      steps:
      - setWeight: 10
      - pause: {duration: 2m}
      - setWeight: 20
      - pause: {duration: 2m}
      - setWeight: 50
      - pause: {duration: 2m}
```

## Cleanup

### Uninstall with Helm

```bash
helm uninstall bluegreen-deployment-helm -n bluegreen
kubectl delete namespace bluegreen
```

### Remove ArgoCD Application

```bash
kubectl delete application bluegreen-deployment-helm -n argocd
```

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## License

This project is licensed under the MIT License.

## Support

For issues and questions, please open an issue on the GitHub repository.
