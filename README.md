# ArgoCD ApplicationSet Multi-Cluster Helm Deployment

This repository demonstrates how to use ArgoCD ApplicationSet to automatically deploy multiple Helm chart instances across different Kubernetes clusters and namespaces using a GitOps approach.

## What is this?

This project sets up an automated deployment system where:
- Multiple Kubernetes clusters can be managed from a single Git repository
- Helm charts are deployed automatically based on the directory structure
- Each application instance can have its own configuration
- Changes to the repository automatically trigger updates in your clusters

## Prerequisites

Before you begin, you need:

1. **Kubernetes clusters** - At least one cluster, but this example assumes three clusters
2. **ArgoCD installed** on your management cluster
3. **kubectl** configured to access your clusters
4. **argocd CLI** installed on your local machine
5. **Git** installed on your local machine

### Register Your Git Repository

Add this repository (or your fork) to ArgoCD:
```bash
argocd repo add https://github.com/digitalstudium/argocd-example.git
```

For private repositories, you'll need to add credentials:
```bash
argocd repo add git@github.com:digitalstudium/argocd-example.git --ssh-private-key-path ~/.ssh/id_rsa
```

### Register Your Kubernetes Clusters

Register each cluster with ArgoCD (replace with your actual cluster names):

```bash
# If using multiple clusters, add them to ArgoCD
argocd cluster add cluster-name-1
argocd cluster add cluster-name-2
argocd cluster add cluster-name-3

# List registered clusters to verify
argocd cluster list
```

**Note**: The cluster where ArgoCD is installed is automatically registered as `in-cluster`.

## Directory Structure Explained

```
clusters/
├── cluster-name-1/          # Your Kubernetes cluster name
│   └── namespaces/
│       ├── bar/             # Kubernetes namespace
│       │   └── charts/
│       │       ├── nginx/   # Helm chart name
│       │       │   ├── instance1/   # Instance name
│       │       │   │   ├── metadata.yaml  # Chart version & repo info
│       │       │   │   └── values.yaml    # Helm values for this instance
```

Each level has a specific meaning:
- **Cluster name**: Must match the cluster name registered in ArgoCD
- **Namespace**: The Kubernetes namespace where the chart will be deployed
- **Chart name**: The name of the Helm chart to deploy
- **Instance name**: Allows multiple instances of the same chart
- **metadata.yaml**: Contains chart version and repository URL
- **values.yaml**: Custom Helm values for this specific instance

## How It Works

The ApplicationSet (`appset.yaml`) automatically:
1. Scans the repository for all `metadata.yaml` files
2. Creates an ArgoCD Application for each instance found
3. Deploys the specified Helm chart version to the correct cluster and namespace
4. Applies the custom values from `values.yaml`

## Getting Started

### Step 1: Fork or Clone This Repository
```bash
git clone https://github.com/digitalstudium/argocd-example.git
cd argocd-example
```

### Step 2: Update Cluster Names
Replace `cluster-name-1`, `cluster-name-2`, `cluster-name-3` with your actual cluster names as shown in `argocd cluster list`.

### Step 3: Update Git Repository URLs
In `appset.yaml`, replace the Git repository URLs with your own:
```yaml
repoURL: git@github.com:YOUR_USERNAME/YOUR_REPO.git
```

### Step 4: Apply the ApplicationSet
```bash
kubectl apply -f appset.yaml
```

### Step 5: Watch Your Applications Deploy
Check ArgoCD UI or use:
```bash
kubectl get applications -n argocd
# or
argocd app list
```

## Adding a New Application

To deploy a new application, create the directory structure:

```bash
# Example: Deploy PostgreSQL to cluster-name-1 in namespace "database"
mkdir -p clusters/cluster-name-1/namespaces/database/charts/postgresql/instance1

# Create metadata.yaml
cat > clusters/cluster-name-1/namespaces/database/charts/postgresql/instance1/metadata.yaml << EOF
version: 12.5.8
repoUrl: https://charts.bitnami.com/bitnami
EOF

# Create values.yaml with your custom configuration
cat > clusters/cluster-name-1/namespaces/database/charts/postgresql/instance1/values.yaml << EOF
auth:
  postgresPassword: "mysecretpassword"
  database: "myapp"
EOF

# Commit and push
git add .
git commit -m "Add PostgreSQL to cluster-name-1"
git push
```

ArgoCD will automatically detect the new files and create the application.

## Customizing Values

Each instance has its own `values.yaml` file. For example, to configure nginx:

```yaml
# clusters/cluster-name-1/namespaces/foo/charts/nginx/instance1/values.yaml
replicaCount: 3
service:
  type: LoadBalancer
  port: 80
resources:
  limits:
    cpu: 200m
    memory: 256Mi
```

## Understanding metadata.yaml

The `metadata.yaml` file tells ArgoCD which Helm chart to use:

```yaml
version: 21.0.3                                        # Helm chart version
repoUrl: https://mirror.yandex.ru/helm/charts.bitnami.com  # Helm repository URL
```

## Common Tasks

### Update a Chart Version
1. Edit the `metadata.yaml` file and change the version
2. Commit and push the change
3. ArgoCD will automatically update the deployment

### Deploy to a Different Namespace
Simply create the directory structure under a different namespace folder.

### Remove an Application
Delete the instance directory and push the change. ArgoCD will automatically remove the application.

## Troubleshooting

### Application Not Appearing
- Check that cluster names match exactly with those registered in ArgoCD
- Verify the ApplicationSet is created: `kubectl get applicationset -n argocd`
- Check ApplicationSet logs: `kubectl logs -n argocd deployment/argocd-applicationset-controller`

### Deployment Failing
- Check the ArgoCD UI for error messages
- Verify the Helm repository URL is accessible
- Ensure the chart version exists in the repository
- Check your `values.yaml` for syntax errors

### Finding Your Cluster Names
```bash
# List all registered clusters
argocd cluster list

# Or check secrets
kubectl get secrets -n argocd -l argocd.argoproj.io/secret-type=cluster
```

## Important Notes

- The namespace will be automatically created if it doesn't exist (due to `CreateNamespace=true`)
- Applications are set to auto-sync and self-heal
- Deleting files from Git will automatically remove the corresponding applications (prune is enabled)
- Make sure your Git repository is accessible by ArgoCD with proper credentials

## Next Steps

1. Explore adding more charts from different Helm repositories
2. Set up different environments (dev, staging, prod) using different clusters
3. Implement Helm value overlays for common configurations
4. Add health checks and notifications for your deployments

For more information about ArgoCD ApplicationSets, visit the [official documentation](https://argo-cd.readthedocs.io/en/stable/user-guide/application-set/).
