# Redis Enterprise ArgoCD Sync-Wave Fix - Local Testing Guide

## Overview

This package tests the ArgoCD sync-wave fix that prevents "REC is frozen" errors during Redis Enterprise upgrades.

**The Problem**: When ArgoCD syncs during an REC upgrade, it tries to update REDBs while the cluster is in upgrade state. The admission webhook rejects these changes with "cluster is frozen".

**The Fix**: A Sync Hook Job at wave 15 polls the REC status and blocks until it returns to `Running` state, ensuring REDBs (wave 20) only update after the upgrade completes.

## Sync Wave Order

```
Wave 5:  RBAC (Role/RoleBinding for wait job)
Wave 10: RedisEnterpriseCluster
Wave 15: wait-for-rec-ready Job (BLOCKS until REC is Running)
Wave 20: RedisEnterpriseDatabase(s)
Wave 30: Webhook configuration
```

## Prerequisites

- Kind cluster with ArgoCD installed
- `kubectl`, `helm`, `argocd` CLI tools
- GitHub account (to push the chart)

## Setup Instructions

### 1. Push to GitHub

ArgoCD needs a git repo. Create one and push this chart:

```bash
# From the directory containing redis-enterprise-databases/
git init
git add .
git commit -m "Redis Enterprise chart with sync-wave fix"

# Create repo on GitHub, then:
git remote add origin https://github.com/YOUR_USER/redis-argocd-test.git
git branch -M main
git push -u origin main
```

### 2. Install CRDs

```bash
kubectl apply -f redis-enterprise-databases/charts/redis-enterprise-operator/crds/crds.yaml
```

### 3. Create Namespace and Secrets

```bash
kubectl create namespace infra

# Create dummy REDB password secret (required by REDB template)
kubectl create secret generic redis-secret-redis-replicated -n infra \
  --from-literal=password=testpass123 \
  --from-literal=port=13001 \
  --from-literal=service_name=redis-replicated-headless
```

### 4. Delete Any Existing Webhook

```bash
kubectl delete validatingwebhookconfiguration redis-enterprise-admission --ignore-not-found
```

### 5. Create ArgoCD Application

```bash
argocd app create redis-test \
  --repo https://github.com/YOUR_USER/redis-argocd-test.git \
  --path redis-enterprise-databases \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace infra \
  --helm-set-file values=values-local-initial.yaml
```

Or use this Application manifest:

```yaml
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
```

### 6. Initial Sync (Deploy at v7.4.2-54)

```bash
argocd app sync redis-test
```

### 7. Wait for REC to be Running

```bash
# Watch until state=Running, specStatus=Valid
kubectl get rec test-redis -n infra -w

# Or use this loop:
while true; do
  kubectl get rec test-redis -n infra -o jsonpath='State: {.status.state}, SpecStatus: {.status.specStatus}' 2>/dev/null
  echo ""
  sleep 10
done
```

### 8. Trigger Upgrade (The Actual Test)

Update the ArgoCD app to use upgrade values:

```bash
argocd app set redis-test --values values-local-upgrade.yaml
argocd app sync redis-test
```

### 9. Watch the Fix in Action

**Terminal 1 - Watch jobs:**
```bash
kubectl get jobs -n infra -w
```

**Terminal 2 - Watch wait job logs:**
```bash
kubectl logs -f job/wait-for-rec-ready-2 -n infra
```

**Terminal 3 - Watch operator logs:**
```bash
kubectl logs -f deployment/redis-enterprise-operator -n infra
```

## Expected Behavior

### With the Fix (Normal)

1. ArgoCD syncs wave 10 (REC upgrade starts)
2. ArgoCD syncs wave 15 (wait job starts)
3. Wait job polls REC status every 30 seconds
4. REC goes through states: `Upgrade` → eventually `Running`
5. Wait job exits successfully when REC is Running
6. ArgoCD syncs wave 20 (REDBs update successfully)
7. **No "REC is frozen" errors**

### Without the Fix (Broken)

1. ArgoCD syncs REC upgrade
2. ArgoCD immediately tries to sync REDBs
3. Admission webhook rejects with "cluster is frozen"
4. Sync fails

## Cleanup

```bash
argocd app delete redis-test --yes
kubectl delete rec --all -n infra
kubectl delete redb --all -n infra
kubectl delete jobs --all -n infra
kubectl delete namespace infra
```

## Files

- `values-local-initial.yaml` - Initial deploy (v7.4.2-54)
- `values-local-upgrade.yaml` - Upgrade deploy (v7.4.6-52)
- `templates/wait-for-rec-job.yaml` - The fix (wave 15)
- `templates/wait-for-rec-role.yaml` - RBAC for the fix (wave 5)
