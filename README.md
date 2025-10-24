# ArgoCD Installation and Setup Guide

This guide walks you through installing ArgoCD, retrieving the admin password, and deploying an application.

## Prerequisites

- A running Kubernetes cluster
- `kubectl` configured to access your cluster
- Git repository with your application manifests

## Installation Steps

### 1. Install ArgoCD

Create the ArgoCD namespace and install ArgoCD:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 2. Wait for ArgoCD Pods to be Ready

```bash
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=300s
```

### 3. Access ArgoCD UI

Expose the ArgoCD server using port-forward:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Access the UI at: `https://localhost:8080`

> **Note**: You may see a certificate warning in your browser. This is expected with self-signed certificates.

### 4. Get ArgoCD Admin Password

Retrieve the initial admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

**Default credentials:**
- Username: `admin`
- Password: (output from the command above)

> **Security Note**: Change the default password after first login.

### 5. Install ArgoCD CLI (Optional)

**Linux/WSL:**
```bash
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64
```

**macOS:**
```bash
brew install argocd
```

**Windows:**
Download from [ArgoCD releases](https://github.com/argoproj/argo-cd/releases/latest)

### 6. Login via CLI (Optional)

```bash
argocd login localhost:8080 --username admin --password <your-password> --insecure
```

## Deploy Application

### 1. Create Application Manifest

Create a file `argocd/application.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: scenario-app
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  
  source:
    repoURL: https://github.com/Anup-Neupane/my-app.git
    targetRevision: main
    path: .
  
  destination:
    server: https://kubernetes.default.svc
    namespace: demo
  
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
  
  ignoreDifferences:
  - group: ""
    kind: Secret
    name: docker-credentials
    jsonPointers:
    - /data
```

### 2. Apply the Application

```bash
kubectl apply -f argocd/application.yaml
```

### 3. Verify Deployment

Check the application status:

```bash
kubectl get applications -n argocd
```

Or view in the ArgoCD UI at `https://localhost:8080`

## Application Configuration Explained

- **name**: `scenario-app` - The application name in ArgoCD
- **namespace**: `argocd` - ArgoCD namespace
- **repoURL**: Your Git repository URL
- **targetRevision**: `main` - Branch to track
- **path**: `.` - Path to Kubernetes manifests in the repo
- **destination.namespace**: `demo` - Target namespace for deployment
- **syncPolicy.automated**: Automatically sync changes from Git
  - `prune: true` - Remove resources deleted from Git
  - `selfHeal: true` - Revert manual changes
- **syncOptions.CreateNamespace**: Automatically create the target namespace
- **ignoreDifferences**: Ignore changes to specific resources (e.g., secrets)

## Common Commands

```bash
# List applications
kubectl get applications -n argocd

# Get application details
kubectl describe application scenario-app -n argocd

# Sync application manually
argocd app sync scenario-app

# Get application status
argocd app get scenario-app

# Delete application
kubectl delete application scenario-app -n argocd
```

## Troubleshooting

### Pod Not Starting
```bash
kubectl get pods -n argocd
kubectl logs <pod-name> -n argocd
```

### Application Not Syncing
```bash
kubectl describe application scenario-app -n argocd
argocd app get scenario-app
```

### Reset Admin Password
```bash
kubectl -n argocd patch secret argocd-secret -p '{"stringData": {"admin.password": ""}}'
kubectl -n argocd delete pod -l app.kubernetes.io/name=argocd-server
```

## Uninstall ArgoCD

```bash
kubectl delete -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl delete namespace argocd
```

## Additional Resources

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [ArgoCD GitHub Repository](https://github.com/argoproj/argo-cd)
- [Best Practices](https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/)
