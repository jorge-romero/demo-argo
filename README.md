# ArgoCD Sync Strategies Demo

This repository demonstrates the different sync strategies available in ArgoCD for GitOps deployments.

## Repository Structure

```
.
├── argocd-config/                  # ArgoCD configuration
│   ├── project.yaml                # AppProject definition
│   ├── repository-public.yaml      # Public repo registration
│   ├── repository-private-https.yaml  # Private repo (HTTPS)
│   ├── repository-private-ssh.yaml    # Private repo (SSH)
│   ├── repository-credentials-template.yaml  # Credential template
│   ├── app-of-apps.yaml            # App of Apps pattern
│   └── kustomization.yaml
├── base/                           # Base Kubernetes manifests
│   ├── namespace.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   └── kustomization.yaml
├── apps/                           # ArgoCD Application definitions
│   ├── manual-sync/                # Manual sync strategy
│   ├── auto-sync/                  # Basic auto-sync
│   ├── auto-sync-self-heal/        # Auto-sync with self-heal
│   ├── auto-sync-prune/            # Auto-sync with prune
│   └── full-sync-options/          # All options combined
└── README.md
```

## Sync Strategies Overview

### 1. Manual Sync (Default)

**Location:** `apps/manual-sync/application.yaml`

The most conservative approach. ArgoCD detects changes but requires manual intervention to sync.

| Feature | Behavior |
|---------|----------|
| Git Changes | Detected, shown as OutOfSync |
| Auto Apply | No - requires manual sync |
| Manual Changes | Not reverted |
| Deleted Resources | Not pruned |

**Use Case:** Production environments requiring strict change control and approval workflows.

```yaml
syncPolicy:
  syncOptions:
    - CreateNamespace=true
# No 'automated' block = manual sync only
```

---

### 2. Auto-Sync (Basic)

**Location:** `apps/auto-sync/application.yaml`

Automatically syncs when Git changes are detected.

| Feature | Behavior |
|---------|----------|
| Git Changes | Automatically applied |
| Auto Apply | Yes |
| Manual Changes | NOT reverted |
| Deleted Resources | NOT pruned |

**Use Case:** Development/staging environments where you want basic GitOps automation.

```yaml
syncPolicy:
  automated: {}
```

---

### 3. Auto-Sync with Self-Heal

**Location:** `apps/auto-sync-self-heal/application.yaml`

Automatically reverts any manual changes made directly to the cluster.

| Feature | Behavior |
|---------|----------|
| Git Changes | Automatically applied |
| Auto Apply | Yes |
| Manual Changes | Automatically reverted |
| Deleted Resources | NOT pruned |

**Use Case:** Environments enforcing strict GitOps compliance (no manual kubectl edits).

```yaml
syncPolicy:
  automated:
    selfHeal: true
```

**Example:** If someone runs `kubectl scale deployment demo-app --replicas=5`, ArgoCD will automatically revert it to match Git.

---

### 4. Auto-Sync with Prune

**Location:** `apps/auto-sync-prune/application.yaml`

Automatically deletes cluster resources that are removed from Git.

| Feature | Behavior |
|---------|----------|
| Git Changes | Automatically applied |
| Auto Apply | Yes |
| Manual Changes | NOT reverted |
| Deleted Resources | Automatically deleted |

**Use Case:** Full GitOps lifecycle management.

```yaml
syncPolicy:
  automated:
    prune: true
```

**Warning:** Be careful with prune in production! Removing a manifest file from Git will delete the corresponding resources.

---

### 5. Full Sync Options

**Location:** `apps/full-sync-options/application.yaml`

Combines all sync features with additional advanced options.

| Feature | Behavior |
|---------|----------|
| Git Changes | Automatically applied |
| Auto Apply | Yes |
| Manual Changes | Automatically reverted |
| Deleted Resources | Automatically deleted |
| Server-Side Apply | Enabled |
| Ignore Differences | Configured |

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
    allowEmpty: false
  syncOptions:
    - CreateNamespace=true
    - Validate=true
    - PruneLast=true
    - ServerSideApply=true
    - RespectIgnoreDifferences=true
  retry:
    limit: 5
    backoff:
      duration: 5s
      factor: 2
      maxDuration: 3m
```

## Sync Options Reference

| Option | Description |
|--------|-------------|
| `CreateNamespace=true` | Create namespace if it doesn't exist |
| `Validate=true` | Validate resources against K8s schema |
| `PruneLast=true` | Delete resources after all other operations |
| `PrunePropagationPolicy=foreground` | Wait for dependents to be deleted |
| `ServerSideApply=true` | Use server-side apply instead of kubectl apply |
| `ApplyOutOfSyncOnly=true` | Only sync resources that are out of sync |
| `RespectIgnoreDifferences=true` | Honor ignoreDifferences configuration |
| `FailOnSharedResource=true` | Fail if another app manages the same resource |

## Quick Start

### Prerequisites

- Kubernetes cluster
- ArgoCD installed ([Installation Guide](https://argo-cd.readthedocs.io/en/stable/getting_started/))
- `kubectl` configured

### Step 1: Register the Repository in ArgoCD

Choose the appropriate method based on your repository type:

#### Option A: Public Repository

```bash
# Apply the project and public repository configuration
kubectl apply -f argocd-config/project.yaml
kubectl apply -f argocd-config/repository-public.yaml
```

#### Option B: Private Repository (HTTPS with Token)

```bash
# Edit the file to add your GitHub credentials
vim argocd-config/repository-private-https.yaml

# Apply the configuration
kubectl apply -f argocd-config/project.yaml
kubectl apply -f argocd-config/repository-private-https.yaml
```

#### Option C: Private Repository (SSH Key)

```bash
# Edit the file to add your SSH private key
vim argocd-config/repository-private-ssh.yaml

# Apply the configuration
kubectl apply -f argocd-config/project.yaml
kubectl apply -f argocd-config/repository-private-ssh.yaml
```

#### Option D: Using ArgoCD CLI

```bash
# Public repository
argocd repo add https://github.com/jorge-romero/demo-argo.git

# Private repository with HTTPS
argocd repo add https://github.com/jorge-romero/demo-argo.git \
  --username <username> \
  --password <token>

# Private repository with SSH
argocd repo add git@github.com:jorge-romero/demo-argo.git \
  --ssh-private-key-path ~/.ssh/id_rsa
```

#### Option E: Using Kustomize (All at once)

```bash
# Apply all ArgoCD configuration
kubectl apply -k argocd-config/
```

### Step 2: Verify Repository Registration

```bash
# Using ArgoCD CLI
argocd repo list

# Or check the secret
kubectl get secrets -n argocd -l argocd.argoproj.io/secret-type=repository
```

### Step 3: Deploy an Example Application

1. Fork/clone this repository

2. Update the `repoURL` in the application manifests to point to your repository

3. Apply an ArgoCD Application:

```bash
# Manual sync strategy
kubectl apply -f apps/manual-sync/application.yaml

# Or auto-sync with self-heal
kubectl apply -f apps/auto-sync-self-heal/application.yaml

# Or use the App of Apps pattern (deploys all examples)
kubectl apply -f argocd-config/app-of-apps.yaml
```

### Step 4: View in ArgoCD UI or CLI

```bash
argocd app list
argocd app get demo-app-manual-sync
```

## Testing Sync Behaviors

### Test Manual Sync

```bash
# Apply the manual sync app
kubectl apply -f apps/manual-sync/application.yaml

# Make a change in Git (e.g., change replicas)
# ArgoCD will show OutOfSync but won't auto-sync

# Manually sync
argocd app sync demo-app-manual-sync
```

### Test Self-Heal

```bash
# Apply the self-heal app
kubectl apply -f apps/auto-sync-self-heal/application.yaml

# Make a manual change
kubectl -n demo-app-self-heal scale deployment demo-app --replicas=5

# Watch ArgoCD revert the change
kubectl -n demo-app-self-heal get deployment demo-app -w
```

### Test Prune

```bash
# Apply the prune app
kubectl apply -f apps/auto-sync-prune/application.yaml

# Remove a resource from Git (e.g., delete configmap.yaml)
# ArgoCD will automatically delete the ConfigMap from the cluster
```

## Comparison Matrix

| Strategy | Auto-Sync | Self-Heal | Prune | Recommended For |
|----------|-----------|-----------|-------|-----------------|
| Manual | No | No | No | Production with approvals |
| Auto-Sync | Yes | No | No | Development |
| Self-Heal | Yes | Yes | No | Strict GitOps |
| Prune | Yes | No | Yes | Full lifecycle |
| Full Options | Yes | Yes | Yes | Mature GitOps |

## Best Practices

1. **Start Conservative:** Begin with manual sync in production
2. **Enable Self-Heal Gradually:** Test in staging first
3. **Be Careful with Prune:** Ensure proper Git workflows before enabling
4. **Use Retry Policies:** Configure retries for transient failures
5. **Configure Ignore Differences:** For auto-generated fields (clusterIP, etc.)
6. **Monitor Sync Status:** Set up notifications for sync failures

## ArgoCD Configuration Details

### AppProject

The `argocd-config/project.yaml` defines:
- **Source Repositories:** Which Git repos can be used
- **Destinations:** Which clusters/namespaces apps can deploy to
- **Resource Whitelists:** Which Kubernetes resources are allowed
- **Roles:** RBAC for project-level access control

### Repository Registration

| File | Use Case |
|------|----------|
| `repository-public.yaml` | Public GitHub repositories |
| `repository-private-https.yaml` | Private repos with username/token |
| `repository-private-ssh.yaml` | Private repos with SSH keys |
| `repository-credentials-template.yaml` | Reusable credentials for multiple repos |

### App of Apps

The `app-of-apps.yaml` implements the App of Apps pattern, allowing ArgoCD to manage all the demo Applications from Git.

## Additional Resources

- [ArgoCD Sync Documentation](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-options/)
- [ArgoCD Auto-Sync](https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/)
- [ArgoCD Best Practices](https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/)
