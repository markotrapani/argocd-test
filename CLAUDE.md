# Redis Enterprise ArgoCD Upgrade Fix - Handoff Document

## Problem Statement

Customer uses ArgoCD to deploy Redis Enterprise Cluster (REC) and Redis Enterprise Databases (REDB). During REC upgrades, the cluster enters a "frozen" state. If ArgoCD attempts to update REDB while REC is frozen, the admission webhook rejects changes, causing sync failures.

**Goal**: Single ArgoCD sync that upgrades REC and REDB without failures - REC must be fully ready before REDB updates are applied.

## Solution Architecture

Two complementary mechanisms:

### 1. ArgoCD Custom Health Check for REC
Configures ArgoCD to understand REC health status natively. This enables sync wave gating.

```yaml
# Apply to argocd-cm ConfigMap in argocd namespace
data:
  resource.customizations.health.app.redislabs.com_RedisEnterpriseCluster: |
    hs = {}
    if obj.status ~= nil then
      if obj.status.state == "Running" and obj.status.specStatus == "Valid" then
        hs.status = "Healthy"
        hs.message = "REC is running and valid"
      elseif obj.status.state == "RecoveryReset" or obj.status.state == "Error" then
        hs.status = "Degraded"
        hs.message = obj.status.state
      else
        hs.status = "Progressing"
        hs.message = obj.status.state or "Initializing"
      end
    else
      hs.status = "Progressing"
      hs.message = "Waiting for status"
    end
    return hs
```

### 2. Sync Waves with Wait Job (Belt-and-Suspenders)
- Wave 5: RBAC for wait job
- Wave 10: REC deployment
- Wave 15: Wait-for-REC-ready Job (blocks until REC is Running/Valid)
- Wave 20: REDB deployment
- Wave 30: ValidatingWebhookConfiguration

### 3. Async Sync Pattern
ArgoCD sync operations timeout after ~5 minutes. REC bootstrap can take 10-15 minutes. Use:
```bash
argocd app sync <app-name> --async
argocd app wait <app-name> --health --timeout 1800
```

## Current State

### Git Repository
- **Repo**: https://github.com/markotrapani/argocd-test.git
- **Path**: `redis-enterprise-databases/`
- **Latest commit**: Should have wait-for-rec job files restored (user ran `git revert HEAD`)

### Files That Should Exist in Chart

```
redis-enterprise-databases/
├── Chart.yaml
├── values.yaml
├── values-local-initial.yaml    # For testing - REC 7.22.2-20
├── values-local-upgrade.yaml    # For testing - REDB memorySize change
├── templates/
│   ├── wait-for-rec-job.yaml    # Wave 15 - blocks until REC ready
│   ├── wait-for-rec-role.yaml   # Wave 5 - RBAC
│   ├── wait-for-rec-rolebinding.yaml  # Wave 5 - RBAC (may be in role.yaml)
│   ├── rec.yaml                 # Wave 10
│   ├── redb.yaml                # Wave 20
│   └── webhook.yaml             # Wave 30
└── charts/
    ├── redis-enterprise-operator/
    ├── redis-enterprise-cluster/
    └── redis-enterprise-db-def/
```

### Wait Job Template (templates/wait-for-rec-job.yaml)
```yaml
{{- $recname := index .Values "redis-enterprise-cluster" "recname" }}
apiVersion: batch/v1
kind: Job
metadata:
  name: wait-for-rec-ready-{{ .Release.Revision }}
  annotations:
    argocd.argoproj.io/hook: Sync
    argocd.argoproj.io/sync-wave: "15"
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  backoffLimit: 1
  activeDeadlineSeconds: 1800
  template:
    spec:
      serviceAccountName: admission-sa
      restartPolicy: Never
      containers:
        - name: wait-for-rec
          image: bitnami/kubectl:latest
          command:
            - /bin/bash
            - -c
            - |
              #!/bin/bash
              
              # Handle SIGTERM gracefully
              trap "echo 'Received SIGTERM, exiting'; exit 0" SIGTERM SIGINT
              
              REC_NAME="{{ $recname }}"
              echo "Waiting for REC $REC_NAME to be in Running state..."
              
              while true; do
                STATE=$(kubectl get rec "$REC_NAME" -n {{ .Release.Namespace }} -o jsonpath='{.status.state}' 2>/dev/null)
                SPEC_STATUS=$(kubectl get rec "$REC_NAME" -n {{ .Release.Namespace }} -o jsonpath='{.status.specStatus}' 2>/dev/null)
                
                echo "$(date): REC state: $STATE, specStatus: $SPEC_STATUS"
                
                if [[ "$STATE" == "Running" && "$SPEC_STATUS" == "Valid" ]]; then
                  echo "REC is ready! Proceeding with database deployments..."
                  exit 0
                fi
                
                echo "REC not ready yet, waiting 30 seconds..."
                sleep 30 &
                wait $!
              done
```

### Wait Job RBAC (templates/wait-for-rec-role.yaml)
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: wait-for-rec-role
  annotations:
    argocd.argoproj.io/sync-wave: "5"
rules:
  - apiGroups: ["app.redislabs.com"]
    resources: ["redisenterpriseclusters"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: wait-for-rec-rolebinding
  annotations:
    argocd.argoproj.io/sync-wave: "5"
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: wait-for-rec-role
subjects:
  - kind: ServiceAccount
    name: admission-sa
    namespace: {{ .Release.Namespace }}
```

## Testing Procedure

### 1. Create Fresh Kind Cluster
```bash
kind delete cluster --name desktop

cat <<EOF > /tmp/kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
  - role: worker
EOF

kind create cluster --name desktop --config /tmp/kind-config.yaml
```

### 2. Install ArgoCD
```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd
```

### 3. Apply ArgoCD Configuration (Timeouts + CRD Health Check)

**CRITICAL**: These settings prevent ArgoCD sync timeouts during long REC operations.

```bash
kubectl patch configmap argocd-cm -n argocd --type merge -p '
data:
  timeout.reconciliation: "1800s"
  timeout.hard.reconciliation: "1800s"
  resource.customizations.health.apiextensions.k8s.io_CustomResourceDefinition: |
    hs = {}
    if obj.status ~= nil and obj.status.conditions ~= nil then
      for i, condition in ipairs(obj.status.conditions) do
        if condition.type == "Established" and condition.status == "True" then
          hs.status = "Healthy"
          hs.message = "CRD is established"
          return hs
        end
      end
    end
    hs.status = "Progressing"
    hs.message = "Waiting for CRD to be established"
    return hs
'

kubectl rollout restart statefulset argocd-application-controller -n argocd
kubectl rollout restart deployment argocd-repo-server argocd-server -n argocd
kubectl rollout status statefulset argocd-application-controller -n argocd --timeout=120s
```

**Why these settings?**

- `timeout.reconciliation: 1800s` - Extends ArgoCD's internal reconciliation timeout (default is ~3min)
- `timeout.hard.reconciliation: 1800s` - Hard limit for reconciliation
- CRD health check - Prevents timeout errors when ArgoCD checks CRD health during heavy cluster operations

### 4. Setup ArgoCD CLI
```bash
ARGOCD_PWD=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)
kubectl port-forward svc/argocd-server -n argocd 9999:443 &
sleep 3
argocd login localhost:9999 --username admin --password "$ARGOCD_PWD" --insecure
```

### 5. Apply CRDs and Create Namespace
```bash
# CRDs should be in the chart at charts/redis-enterprise-operator/crds/
kubectl apply -f <path-to-crds>/crds.yaml

kubectl create namespace infra
kubectl create secret generic redis-secret-redis-replicated -n infra \
  --from-literal=password=testpass123 \
  --from-literal=port=13001 \
  --from-literal=service_name=redis-replicated-headless
```

### 6. Create and Sync ArgoCD Application
```bash
cat <<EOF | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: redis-test
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/markotrapani/argocd-test.git
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

# Async sync - doesn't timeout
argocd app sync redis-test --async

# Wait for healthy (up to 30 min)
argocd app wait redis-test --health --timeout 1800
```

### 7. Monitor Progress
```bash
# Terminal 1
kubectl get pods -n infra -w

# Terminal 2
kubectl get rec -n infra -w

# Terminal 3
kubectl logs -f job/wait-for-rec-ready-1 -n infra
```

## Known Issues

### 1. Subchart Node Selectors
Production charts have `nodeSelector: workload-category: core` and operator affinity for `workload-category: system`. For local testing, these must be disabled in subchart values.yaml files:
- `charts/redis-enterprise-operator/values.yaml`: Set `affinity: ""`
- `charts/redis-enterprise-cluster/values.yaml`: Set `nodeSelector: {}`

### 2. Finalizers Block Namespace Deletion
When deleting REC/REDB, finalizers can block. Force remove:
```bash
kubectl patch rec <name> -n infra -p '{"metadata":{"finalizers":[]}}' --type=merge
kubectl patch redb <name> -n infra -p '{"metadata":{"finalizers":[]}}' --type=merge
```

### 3. Admission CronJob Errors
The `redis-admission-cronjob` runs hourly and will error until REC is ready. This is expected and harmless. To silence during testing:
```bash
kubectl patch cronjob redis-admission-cronjob -n infra -p '{"spec":{"suspend":true}}'
```

### 4. REC Bootstrap Time
REC with 3 nodes takes 5-15 minutes to bootstrap on Kind. This is normal. States progress:
1. `BootstrappingFirstPod`
2. `Bootstrapping`
3. `Running`

## Clean Slate Commands

**CRITICAL LESSONS LEARNED:**

1. **STOP ArgoCD FIRST** - ArgoCD will recreate resources while you're deleting them
2. **Check deletionTimestamp** - A resource can EXIST but be STUCK in deletion. Always verify no deletionTimestamp!
3. **Finalizers block deletion** - Always remove finalizers BEFORE deleting
4. **Order matters** - Delete in this order: ArgoCD app → CRs → CRDs → Cluster resources → Namespace
5. **Delete secrets and jobs too** - The `redis-secret-*` secrets and `wait-for-rec-ready-*` jobs persist and cause errors on redeploy
6. **Jobs are immutable** - You cannot update a Job spec, must delete and recreate

```bash
# === STEP 1: STOP ARGOCD FROM RECREATING RESOURCES ===
# This MUST be first or ArgoCD will recreate things as you delete them!
kubectl patch application redis-test -n argocd -p '{"metadata":{"finalizers":[]}}' --type=merge 2>/dev/null
kubectl delete application redis-test -n argocd --force --grace-period=0 2>/dev/null

# === STEP 2: Remove finalizers from Custom Resources ===
kubectl patch rec test-redis -n infra -p '{"metadata":{"finalizers":[]}}' --type=merge 2>/dev/null
kubectl patch redb redis-replicated -n infra -p '{"metadata":{"finalizers":[]}}' --type=merge 2>/dev/null

# === STEP 3: Force delete any stuck pods ===
kubectl delete pod -n infra -l app=redis-enterprise --force --grace-period=0 2>/dev/null

# === STEP 4: Remove finalizers from CRDs and delete ===
for crd in redisenterpriseclusters.app.redislabs.com redisenterprisedatabases.app.redislabs.com redisenterpriseactiveactivedatabases.app.redislabs.com redisenterpriseremoteclusters.app.redislabs.com; do
  kubectl patch crd $crd -p '{"metadata":{"finalizers":[]}}' --type=merge 2>/dev/null
  kubectl delete crd $crd --force --grace-period=0 2>/dev/null &
done
wait

# === STEP 5: Delete cluster-scoped resources ===
kubectl delete clusterrole admission-role --ignore-not-found
kubectl delete clusterrolebinding admission-crb --ignore-not-found
kubectl delete validatingwebhookconfiguration redis-enterprise-admission --ignore-not-found

# === STEP 6: Delete namespace (with finalizer removal if stuck) ===
kubectl delete namespace infra --force --grace-period=0 2>/dev/null &
sleep 2
kubectl get namespace infra -o json 2>/dev/null | jq '.spec.finalizers = []' | kubectl replace --raw "/api/v1/namespaces/infra/finalize" -f - 2>/dev/null
wait

# === STEP 7: PROPER VERIFICATION (check deletionTimestamp!) ===
echo "=== VERIFICATION ==="

# Check namespace - must not exist OR have deletionTimestamp
NS_CHECK=$(kubectl get namespace infra -o jsonpath='{.metadata.deletionTimestamp}' 2>/dev/null)
if [ -z "$NS_CHECK" ]; then
  kubectl get namespace infra 2>/dev/null && echo "⚠️  WARNING: infra EXISTS" || echo "✓ infra: GONE"
else
  echo "⚠️  WARNING: infra STUCK IN DELETION (deletionTimestamp: $NS_CHECK)"
fi

# Check CRDs
kubectl get crd | grep redislabs 2>/dev/null && echo "⚠️  WARNING: CRDs still exist" || echo "✓ CRDs: GONE"

# Check ArgoCD app
kubectl get application -n argocd 2>/dev/null | grep redis && echo "⚠️  WARNING: ArgoCD app exists" || echo "✓ ArgoCD app: GONE"

# Check for ANY Redis resources with deletionTimestamp (stuck in deletion)
echo ""
echo "Checking for stuck resources..."
kubectl get rec -A -o custom-columns="NS:.metadata.namespace,NAME:.metadata.name,DELETING:.metadata.deletionTimestamp" 2>/dev/null | grep -v "<none>" && echo "⚠️  WARNING: REC stuck in deletion!" || echo "✓ No stuck RECs"
```

## Success Criteria

1. Single `argocd app sync` (with async+wait) completes successfully
2. REC reaches Running state
3. REDB is created AFTER REC is Running (not before)
4. Webhook is created last
5. No "frozen cluster" rejection errors

## Files to Deliver to Customer

1. `argocd-rec-health-check.yaml` - ConfigMap patch for ArgoCD
2. `sync-wave-examples.yaml` - How to annotate REC/REDB templates
3. `README.md` - Implementation guide
4. Optionally: `wait-for-rec-job.yaml` template if they want belt-and-suspenders

## Questions for Customer

1. How is their ArgoCD deployed? (self-managed, Akuity, OpenShift GitOps)
2. Can they modify argocd-cm ConfigMap?
3. What's their CI/CD pipeline - can it use async sync + wait pattern?
4. How long does their REC upgrade typically take?
