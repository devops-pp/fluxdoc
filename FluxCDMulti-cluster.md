# FluxCD Hub and Spoke Architecture
> Managing a Fleet of Clusters (staging, prod-eu, prod-us) from a Single Hub Cluster

**4 Labs · 35+ Commands · Production-Ready**

---

## Table of Contents

- [Concepts — Hub and Spoke with FluxCD](#concepts--hub-and-spoke-with-fluxcd)
- [Lab 1 — Hub Cluster Bootstrap](#lab-1--hub-cluster-bootstrap)
- [Lab 2 — Register Spoke Clusters](#lab-2--register-spoke-clusters)
- [Lab 3 — Deploy Workloads Across the Fleet](#lab-3--deploy-workloads-across-the-fleet)
- [Lab 4 — Environment Promotion (staging → prod)](#lab-4--environment-promotion-staging--prod)
- [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## Concepts — Hub and Spoke with FluxCD

### What is Hub and Spoke?

The Hub and Spoke model uses one central **hub cluster** running FluxCD to manage the **desired state** of multiple **spoke clusters** (staging, prod-eu, prod-us). The hub pulls from Git (the source of truth) and pushes reconciled state out to each spoke.

```
Desired State (Git)                Hub Cluster              Fleet State (Spokes)
──────────────────                 ───────────              ───────────────────
  Helm charts        ←── Flux ──→  flux hub k8s  ───────→  staging k8s
  OCI / Buckets                                  ───────→  prod-eu k8s
  Docker images                                  ───────→  prod-us k8s
  Git manifests
```

### Why Hub and Spoke?

| Problem | Hub and Spoke Solution |
|---|---|
| Each cluster running its own Flux = drift risk | One Flux controls all clusters uniformly |
| Promoting changes across envs is manual | Kustomization overlays promote staging → prod |
| Auditing deployments across clusters is hard | Single Git history covers the whole fleet |
| Spoke clusters need internet access to pull | Hub pulls once, applies to all spokes |

### Core Components

**Hub cluster** — Runs Flux controllers. Has `kubeconfig` access to all spoke clusters. Reads from Git and applies resources to spokes via the Kubernetes API.

**Spoke cluster** — Receives manifests applied by the hub. Does NOT run Flux itself. Only needs to be reachable from the hub (network/kubeconfig).

**`Kustomization` with `kubeConfig`** — The key Flux primitive that targets a spoke. Instead of applying to the local cluster, Flux uses a referenced kubeconfig Secret to apply to a remote cluster.

**Fleet repo structure:**

```
flux-gitops/
└── clusters/
    ├── hub/
    │   └── flux-system/              ← Flux bootstrap lives here
    ├── staging/
    │   ├── infrastructure.yaml       ← points to infra/ with staging values
    │   └── apps.yaml                 ← points to apps/ with staging values
    ├── prod-eu/
    │   ├── infrastructure.yaml
    │   └── apps.yaml
    └── prod-us/
        ├── infrastructure.yaml
        └── apps.yaml
```

---

## Lab 1 — Hub Cluster Bootstrap

**Objective:** Bootstrap FluxCD on the hub cluster and set up the fleet repository structure.

**Estimated Time:** 30 minutes

### Step 1 — Prerequisites

```bash
# Confirm you have kubeconfig access to all clusters
kubectl config get-contexts

# Expected contexts:
# hub-cluster
# staging-cluster
# prod-eu-cluster
# prod-us-cluster

# Switch to hub cluster context
kubectl config use-context hub-cluster

# Verify flux CLI
flux version
```

---

### Step 2 — Bootstrap Flux on the Hub Cluster

```bash
# Bootstrap using GitLab (adjust for your repo)
flux bootstrap gitlab \
  --owner=your-org \
  --repository=flux-gitops \
  --branch=main \
  --path=clusters/hub \
  --token-auth \
  --personal

# Verify all Flux controllers are running on the hub
flux check

# Expected:
# ✔ source-controller: deployment ready
# ✔ kustomize-controller: deployment ready
# ✔ helm-controller: deployment ready
# ✔ notification-controller: deployment ready
```

---

### Step 3 — Create the Fleet Directory Structure

```bash
# Clone the repo Flux just bootstrapped
git clone https://gitlab.com/your-org/flux-gitops
cd flux-gitops

# Create directories for each spoke cluster
mkdir -p clusters/staging
mkdir -p clusters/prod-eu
mkdir -p clusters/prod-us

# Create shared base directories
mkdir -p infrastructure/base
mkdir -p infrastructure/staging
mkdir -p infrastructure/prod-eu
mkdir -p infrastructure/prod-us

mkdir -p apps/base
mkdir -p apps/staging
mkdir -p apps/prod-eu
mkdir -p apps/prod-us
```

Your repo should now look like:

```
flux-gitops/
├── clusters/
│   ├── hub/
│   │   └── flux-system/
│   │       ├── gotk-components.yaml
│   │       ├── gotk-sync.yaml
│   │       └── kustomization.yaml
│   ├── staging/
│   ├── prod-eu/
│   └── prod-us/
├── infrastructure/
│   ├── base/
│   ├── staging/
│   ├── prod-eu/
│   └── prod-us/
└── apps/
    ├── base/
    ├── staging/
    ├── prod-eu/
    └── prod-us/
```

---

### Step 4 — Commit the Structure

```bash
git add .
git commit -m 'feat: initialize hub-and-spoke fleet structure'
git push
```

### ✅ Lab 1 Checklist

- [x] Flux bootstrapped on hub cluster at `clusters/hub`
- [x] `flux check` passes on hub cluster
- [x] Fleet directory structure created for staging, prod-eu, prod-us
- [x] Base and overlay directories created for infrastructure and apps

---

## Lab 2 — Register Spoke Clusters

**Objective:** Store kubeconfig credentials for each spoke cluster as Secrets on the hub, then create Flux Kustomizations that target each spoke.

**Estimated Time:** 35 minutes

### How Spoke Targeting Works

```
Hub Cluster
  └── Flux Kustomization (clusters/staging/apps.yaml)
        └── spec.kubeConfig.secretRef.name = staging-kubeconfig
            └── Secret/staging-kubeconfig in flux-system
                └── contains kubeconfig for staging cluster API server
                    └── Flux applies manifests THERE, not on hub
```

---

### Step 1 — Store Spoke Kubeconfigs as Secrets on the Hub

Run these commands **on the hub cluster** context.

```bash
kubectl config use-context hub-cluster

# Store staging cluster kubeconfig
kubectl create secret generic staging-kubeconfig \
  --namespace=flux-system \
  --from-file=value=<PATH_TO_STAGING_KUBECONFIG>

# Store prod-eu cluster kubeconfig
kubectl create secret generic prod-eu-kubeconfig \
  --namespace=flux-system \
  --from-file=value=<PATH_TO_PROD_EU_KUBECONFIG>

# Store prod-us cluster kubeconfig
kubectl create secret generic prod-us-kubeconfig \
  --namespace=flux-system \
  --from-file=value=<PATH_TO_PROD_US_KUBECONFIG>

# Verify all three secrets exist
kubectl get secrets -n flux-system | grep kubeconfig

# Expected:
# prod-eu-kubeconfig   Opaque   1      30s
# prod-us-kubeconfig   Opaque   1      25s
# staging-kubeconfig   Opaque   1      20s
```

> ⚠️ These secrets contain cluster credentials. Create them imperatively with `kubectl` — never commit raw kubeconfig files to Git. Use Sealed Secrets or SOPS if you need them in a GitOps workflow.

---

### Step 2 — Create a GitRepository Source on the Hub

The hub needs one `GitRepository` that all spoke Kustomizations reference as their source.

```bash
cat > clusters/hub/fleet-gitrepository.yaml <<EOF
---
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: fleet
  namespace: flux-system
spec:
  interval: 1m
  url: https://gitlab.com/your-org/flux-gitops
  branch: main
  secretRef:
    name: flux-system          # Reuse the token Flux bootstrap created
EOF
```

---

### Step 3 — Register the Staging Spoke

```bash
cat > clusters/staging/sync.yaml <<EOF
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: staging-infrastructure
  namespace: flux-system
spec:
  interval: 10m
  path: ./infrastructure/staging        # Apply staging infrastructure overlays
  prune: true
  sourceRef:
    kind: GitRepository
    name: fleet
    namespace: flux-system
  kubeConfig:                           # TARGET: apply to staging cluster, not hub
    secretRef:
      name: staging-kubeconfig
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: staging-apps
  namespace: flux-system
spec:
  interval: 5m
  path: ./apps/staging                  # Apply staging app overlays
  prune: true
  dependsOn:
    - name: staging-infrastructure      # Wait for infra before deploying apps
  sourceRef:
    kind: GitRepository
    name: fleet
    namespace: flux-system
  kubeConfig:
    secretRef:
      name: staging-kubeconfig
EOF
```

---

### Step 4 — Register the Prod-EU Spoke

```bash
cat > clusters/prod-eu/sync.yaml <<EOF
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: prod-eu-infrastructure
  namespace: flux-system
spec:
  interval: 10m
  path: ./infrastructure/prod-eu
  prune: true
  sourceRef:
    kind: GitRepository
    name: fleet
    namespace: flux-system
  kubeConfig:
    secretRef:
      name: prod-eu-kubeconfig
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: prod-eu-apps
  namespace: flux-system
spec:
  interval: 5m
  path: ./apps/prod-eu
  prune: true
  dependsOn:
    - name: prod-eu-infrastructure
  sourceRef:
    kind: GitRepository
    name: fleet
    namespace: flux-system
  kubeConfig:
    secretRef:
      name: prod-eu-kubeconfig
EOF
```

---

### Step 5 — Register the Prod-US Spoke

```bash
cat > clusters/prod-us/sync.yaml <<EOF
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: prod-us-infrastructure
  namespace: flux-system
spec:
  interval: 10m
  path: ./infrastructure/prod-us
  prune: true
  sourceRef:
    kind: GitRepository
    name: fleet
    namespace: flux-system
  kubeConfig:
    secretRef:
      name: prod-us-kubeconfig
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: prod-us-apps
  namespace: flux-system
spec:
  interval: 5m
  path: ./apps/prod-us
  prune: true
  dependsOn:
    - name: prod-us-infrastructure
  sourceRef:
    kind: GitRepository
    name: fleet
    namespace: flux-system
  kubeConfig:
    secretRef:
      name: prod-us-kubeconfig
EOF
```

---

### Step 6 — Add a Root Kustomization to Watch All Cluster Folders

```bash
cat > clusters/hub/fleet-sync.yaml <<EOF
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: fleet
  namespace: flux-system
spec:
  interval: 5m
  path: ./clusters              # Watch all cluster folders
  prune: true
  sourceRef:
    kind: GitRepository
    name: fleet
    namespace: flux-system
EOF
```

---

### Step 7 — Commit and Verify Spoke Registration

```bash
git add clusters/
git commit -m 'feat: register staging, prod-eu, prod-us spoke clusters'
git push

# Reconcile the hub
flux reconcile kustomization flux-system --with-source

# Watch all fleet kustomizations on the hub
flux get kustomizations -A

# Expected (all running on hub, targeting spokes):
# NAMESPACE    NAME                     REVISION      READY  MESSAGE
# flux-system  fleet                    main/abc123   True   Applied revision: main/abc123
# flux-system  staging-infrastructure   main/abc123   True   Applied revision: main/abc123
# flux-system  staging-apps             main/abc123   True   Applied revision: main/abc123
# flux-system  prod-eu-infrastructure   main/abc123   True   Applied revision: main/abc123
# flux-system  prod-eu-apps             main/abc123   True   Applied revision: main/abc123
# flux-system  prod-us-infrastructure   main/abc123   True   Applied revision: main/abc123
# flux-system  prod-us-apps             main/abc123   True   Applied revision: main/abc123
```

### ✅ Lab 2 Checklist

- [x] Kubeconfig secrets created for staging, prod-eu, prod-us in `flux-system`
- [x] Single `fleet` GitRepository source created on the hub
- [x] `sync.yaml` created per cluster with `spec.kubeConfig.secretRef`
- [x] `dependsOn` set so apps wait for infrastructure on each spoke
- [x] All fleet Kustomizations showing `Ready: True` on the hub

---

## Lab 3 — Deploy Workloads Across the Fleet

**Objective:** Use Kustomize base + overlays to deploy the same application to all three spokes with environment-specific configuration.

**Estimated Time:** 35 minutes

### Step 1 — Create a Base Application Manifest

The base is environment-agnostic — no replica counts, no environment-specific config.

```bash
cat > apps/base/deployment.yaml <<EOF
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: nginx:1.25
          ports:
            - containerPort: 80
          env:
            - name: APP_ENV
              value: base
EOF

cat > apps/base/service.yaml <<EOF
---
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 80
EOF

cat > apps/base/kustomization.yaml <<EOF
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
EOF
```

---

### Step 2 — Create the Staging Overlay

Staging: 1 replica, `APP_ENV=staging`, namespace `staging`.

```bash
cat > apps/staging/kustomization.yaml <<EOF
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: staging
resources:
  - ../base
patches:
  - patch: |-
      - op: replace
        path: /spec/replicas
        value: 1
      - op: replace
        path: /spec/template/spec/containers/0/env/0/value
        value: staging
    target:
      kind: Deployment
      name: my-app
EOF
```

---

### Step 3 — Create the Prod-EU Overlay

Prod-EU: 3 replicas, `APP_ENV=prod-eu`, namespace `prod-eu`.

```bash
cat > apps/prod-eu/kustomization.yaml <<EOF
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: prod-eu
resources:
  - ../base
patches:
  - patch: |-
      - op: replace
        path: /spec/replicas
        value: 3
      - op: replace
        path: /spec/template/spec/containers/0/env/0/value
        value: prod-eu
    target:
      kind: Deployment
      name: my-app
EOF
```

---

### Step 4 — Create the Prod-US Overlay

Prod-US: 3 replicas, `APP_ENV=prod-us`, namespace `prod-us`.

```bash
cat > apps/prod-us/kustomization.yaml <<EOF
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: prod-us
resources:
  - ../base
patches:
  - patch: |-
      - op: replace
        path: /spec/replicas
        value: 3
      - op: replace
        path: /spec/template/spec/containers/0/env/0/value
        value: prod-us
    target:
      kind: Deployment
      name: my-app
EOF
```

---

### Step 5 — Commit and Verify Across All Spokes

```bash
git add apps/
git commit -m 'feat: add my-app base and environment overlays'
git push

# Force reconcile from hub
flux reconcile kustomization fleet --with-source

# Check app deployed on staging spoke
kubectl config use-context staging-cluster
kubectl get deployment my-app -n staging
kubectl get pods -n staging

# Check app deployed on prod-eu spoke
kubectl config use-context prod-eu-cluster
kubectl get deployment my-app -n prod-eu
# Expected: 3 pods running (prod replicas)

# Check app deployed on prod-us spoke
kubectl config use-context prod-us-cluster
kubectl get deployment my-app -n prod-us
# Expected: 3 pods running (prod replicas)

# Switch back to hub for fleet-wide status
kubectl config use-context hub-cluster
flux get kustomizations -A
```

---

### Step 6 — Verify Environment Isolation

```bash
# Confirm staging only has 1 replica
kubectl config use-context staging-cluster
kubectl get deployment my-app -n staging -o jsonpath='{.spec.replicas}'
# Expected: 1

# Confirm prod-eu has 3 replicas
kubectl config use-context prod-eu-cluster
kubectl get deployment my-app -n prod-eu -o jsonpath='{.spec.replicas}'
# Expected: 3
```

### ✅ Lab 3 Checklist

- [x] Base manifests created in `apps/base/`
- [x] Staging overlay: 1 replica, staging namespace
- [x] Prod-EU overlay: 3 replicas, prod-eu namespace
- [x] Prod-US overlay: 3 replicas, prod-us namespace
- [x] Deployments verified directly on each spoke cluster

---

## Lab 4 — Environment Promotion (staging → prod)

**Objective:** Promote a new image version from staging to prod-eu and prod-us using Git as the promotion gate.

**Estimated Time:** 25 minutes

### The Promotion Flow

```
1. Update image tag in apps/staging/kustomization.yaml  →  commit + push
2. Flux reconciles staging spoke  →  verify staging is healthy
3. Update image tag in apps/prod-eu and apps/prod-us    →  commit + push
4. Flux reconciles prod spokes    →  promotion complete
```

All steps go through Git. No direct `kubectl set image`. The Git log is the audit trail.

---

### Step 1 — Deploy a New Image to Staging First

Update the image in the staging overlay only.

```bash
# Update staging overlay to use nginx:1.26
cat > apps/staging/kustomization.yaml <<EOF
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: staging
resources:
  - ../base
images:
  - name: nginx
    newTag: "1.26"               # New version — staging gets it first
patches:
  - patch: |-
      - op: replace
        path: /spec/replicas
        value: 1
      - op: replace
        path: /spec/template/spec/containers/0/env/0/value
        value: staging
    target:
      kind: Deployment
      name: my-app
EOF

git add apps/staging/kustomization.yaml
git commit -m 'feat: promote nginx 1.26 to staging'
git push
```

---

### Step 2 — Verify Staging is Healthy

```bash
# Watch staging Kustomization on hub
flux get kustomization staging-apps -n flux-system --watch

# Switch to staging cluster and verify
kubectl config use-context staging-cluster
kubectl get pods -n staging
kubectl describe deployment my-app -n staging | grep Image
# Expected: Image: nginx:1.26

# Run any integration tests or smoke tests here before promoting to prod
```

---

### Step 3 — Promote to Prod-EU and Prod-US

Once staging is verified, update the prod overlays.

```bash
kubectl config use-context hub-cluster

# Update prod-eu overlay
cat > apps/prod-eu/kustomization.yaml <<EOF
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: prod-eu
resources:
  - ../base
images:
  - name: nginx
    newTag: "1.26"               # Promote same version from staging
patches:
  - patch: |-
      - op: replace
        path: /spec/replicas
        value: 3
      - op: replace
        path: /spec/template/spec/containers/0/env/0/value
        value: prod-eu
    target:
      kind: Deployment
      name: my-app
EOF

# Update prod-us overlay
cat > apps/prod-us/kustomization.yaml <<EOF
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: prod-us
resources:
  - ../base
images:
  - name: nginx
    newTag: "1.26"               # Promote same version from staging
patches:
  - patch: |-
      - op: replace
        path: /spec/replicas
        value: 3
      - op: replace
        path: /spec/template/spec/containers/0/env/0/value
        value: prod-us
    target:
      kind: Deployment
      name: my-app
EOF

git add apps/prod-eu/kustomization.yaml apps/prod-us/kustomization.yaml
git commit -m 'feat: promote nginx 1.26 to prod-eu and prod-us'
git push
```

---

### Step 4 — Verify Promotion Across All Spokes

```bash
# Watch all fleet kustomizations on hub
flux get kustomizations -A --watch

# Verify prod-eu
kubectl config use-context prod-eu-cluster
kubectl describe deployment my-app -n prod-eu | grep Image
# Expected: Image: nginx:1.26

# Verify prod-us
kubectl config use-context prod-us-cluster
kubectl describe deployment my-app -n prod-us | grep Image
# Expected: Image: nginx:1.26

# Full fleet image summary (run from hub)
kubectl config use-context hub-cluster
for ctx in staging-cluster prod-eu-cluster prod-us-cluster; do
  echo "=== $ctx ==="
  kubectl --context=$ctx get deployment my-app \
    -n $(echo $ctx | cut -d'-' -f1,2 | sed 's/-cluster//') \
    -o jsonpath='{.spec.template.spec.containers[0].image}'
  echo ""
done
```

---

### Step 5 — Rollback if Needed

If prod shows issues, revert the commit — Flux automatically reconciles the previous state.

```bash
# Identify the last good commit
git log --oneline apps/prod-eu/kustomization.yaml

# Revert the promotion commit
git revert HEAD
git push

# Flux reconciles automatically within the interval
# Or force it immediately:
flux reconcile kustomization prod-eu-apps --with-source
flux reconcile kustomization prod-us-apps --with-source

# Verify rollback
kubectl config use-context prod-eu-cluster
kubectl describe deployment my-app -n prod-eu | grep Image
# Expected: Image: nginx:1.25 (previous version)
```

### ✅ Lab 4 Checklist

- [x] New image version deployed to staging only first
- [x] Staging verified healthy before promotion
- [x] Prod-EU and prod-US promoted in a single commit
- [x] All three spokes confirmed running the same image tag
- [x] Rollback process tested with `git revert`

---

## Quick Reference Cheat Sheet

| Task | Command |
|---|---|
| Switch to hub context | `kubectl config use-context hub-cluster` |
| Bootstrap Flux on hub | `flux bootstrap gitlab --path=clusters/hub ...` |
| Store spoke kubeconfig | `kubectl create secret generic <spoke>-kubeconfig --namespace=flux-system --from-file=value=<path>` |
| Watch fleet kustomizations | `flux get kustomizations -A` |
| Force reconcile hub | `flux reconcile kustomization flux-system --with-source` |
| Force reconcile a spoke | `flux reconcile kustomization staging-apps --with-source` |
| Check spoke events | `flux events --for Kustomization/staging-apps -n flux-system` |
| Verify image on spoke | `kubectl --context=staging-cluster get deployment my-app -n staging -o jsonpath='{.spec.template.spec.containers[0].image}'` |
| Rollback promotion | `git revert HEAD && git push` |
| View kustomize-controller logs | `kubectl logs -n flux-system -l app=kustomize-controller -f` |
| Preview kustomize build | `kustomize build apps/staging` |

---

### Hub and Spoke Security Checklist

- [ ] Spoke kubeconfig secrets stored in `flux-system` namespace only (not in Git)
- [ ] Hub cluster has minimal RBAC on each spoke (only what Flux needs)
- [ ] `prune: true` set on all spoke Kustomizations (enables clean teardown)
- [ ] `dependsOn` set so apps wait for infrastructure on every spoke
- [ ] Promotions always go through Git — no direct `kubectl set image` on spokes
- [ ] Staging validated before prod promotion on every release
- [ ] Spoke kubeconfig tokens rotated regularly and secrets updated

---

> **Architecture Tip:** For large fleets (10+ spokes), consider grouping spoke Kustomizations by region or tier (e.g., one Kustomization per region pointing to `./clusters/prod-eu/`) and using `postBuild.substituteFrom` with a ConfigMap per cluster to inject cluster-specific values (region, replica counts, DNS suffix) without duplicating overlays.
