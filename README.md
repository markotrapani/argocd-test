# Redis Enterprise ArgoCD Sync-Wave Fix

## Overview

This package implements a fix for ArgoCD sync failures during Redis Enterprise Cluster (REC) upgrades.

**The Problem**: When ArgoCD syncs during an REC upgrade, it tries to update REDBs while the cluster is in "frozen" state. The admission webhook rejects these changes with "cluster is frozen" errors.

**The Fix**: A Sync Hook Job at wave 15 polls the REC status and blocks until it returns to `Running` state with all pods at the expected version, ensuring REDBs (wave 20) only update after the upgrade completes.

## Sync Wave Order

```
Wave 5:  RBAC (ServiceAccount, Role, RoleBinding for wait job)
Wave 10: RedisEnterpriseCluster
Wave 15: wait-for-rec-ready Job (BLOCKS until REC is Running with correct version)
Wave 20: RedisEnterpriseDatabase(s)
Wave 30: ValidatingWebhookConfiguration
Wave 35: Admission CronJob (patches webhook with CA cert)
```

## Key Files

| File | Purpose |
|------|---------|
| `templates/wait-for-rec-job.yaml` | Sync hook that blocks until REC is ready (wave 15) |
| `templates/wait-for-rec-role.yaml` | RBAC for the wait job (wave 5) |
| `templates/post-install-webhook.yaml` | CronJob with sync-wave 35 annotation |

## ArgoCD Configuration

Add these settings to the `argocd-cm` ConfigMap to prevent sync timeouts:

```yaml
data:
  timeout.reconciliation: "1800s"
  timeout.hard.reconciliation: "1800s"
```

Apply with:
```bash
kubectl patch configmap argocd-cm -n argocd --type merge -p '
data:
  timeout.reconciliation: "1800s"
  timeout.hard.reconciliation: "1800s"
'
kubectl rollout restart statefulset argocd-application-controller -n argocd
```

## Sync Pattern

REC upgrades can take 10-15+ minutes. Use async sync to avoid CLI timeouts:

```bash
# Start sync without waiting
argocd app sync <app-name> --async

# Wait for healthy (up to 30 min)
argocd app wait <app-name> --health --timeout 1800
```

## Testing

### Prerequisites

- Kind cluster with 3+ worker nodes
- ArgoCD installed and configured
- `kubectl`, `argocd` CLI tools

### Initial Deploy

```bash
# Create namespace and secrets
kubectl create namespace infra
kubectl create secret generic redis-secret-redis-replicated -n infra \
  --from-literal=password=testpass123 \
  --from-literal=port=13001 \
  --from-literal=service_name=redis-replicated-headless

# Create ArgoCD application
cat <<EOF | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: redis-test
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR_USER/redis-argocd-test.git
    path: redis-enterprise-databases
    targetRevision: HEAD
    helm:
      valueFiles:
        - values-local-initial.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: infra
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
EOF

# Sync
argocd app sync redis-test --async
argocd app wait redis-test --health --timeout 1800
```

### Upgrade Test

```bash
# Switch to upgrade values
argocd app set redis-test --values values-local-upgrade.yaml

# Sync upgrade
argocd app sync redis-test --async
argocd app wait redis-test --health --timeout 1800
```

### Monitor Progress

```bash
# Watch REC status
kubectl get rec -n infra -w

# Watch wait job logs
kubectl logs -f job/wait-for-rec-ready-1 -n infra

# Watch pods
kubectl get pods -n infra -w
```

## Expected Behavior

1. ArgoCD syncs wave 5 (RBAC)
2. ArgoCD syncs wave 10 (REC - upgrade starts)
3. ArgoCD syncs wave 15 (wait job starts, polls every 30s)
4. REC goes through states: `Upgrade` → `Running`
5. Wait job verifies ALL pods have expected version
6. Wait job exits successfully
7. ArgoCD syncs wave 20 (REDBs update)
8. ArgoCD syncs wave 30 (Webhook)
9. ArgoCD syncs wave 35 (CronJob)
10. **No "cluster is frozen" errors**

## Cleanup

```bash
# Delete ArgoCD app first
kubectl patch application redis-test -n argocd -p '{"metadata":{"finalizers":[]}}' --type=merge
kubectl delete application redis-test -n argocd

# Remove finalizers from CRs
kubectl patch rec test-redis -n infra -p '{"metadata":{"finalizers":[]}}' --type=merge
kubectl patch redb redis-replicated -n infra -p '{"metadata":{"finalizers":[]}}' --type=merge

# Delete CRDs (remove finalizers first)
for crd in redisenterpriseclusters.app.redislabs.com redisenterprisedatabases.app.redislabs.com; do
  kubectl get crd $crd -o json | jq 'del(.metadata.finalizers)' | kubectl replace -f -
  kubectl delete crd $crd
done

# Delete namespace
kubectl delete namespace infra --force --grace-period=0
```

## Values Files

- `values-local-initial.yaml` - Initial deploy (for testing)
- `values-local-upgrade.yaml` - Upgrade deploy (for testing)
