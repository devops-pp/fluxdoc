# FluxCD GitOps Lab: Environment-Specific Deployments with Kustomize

## Overview

Flux is bootstrapped in **`flux-gitops`** under `clusters/production/flux-system/`.
This means Flux watches **`clusters/production/`** — everything you want reconciled must live there.

**Actual repo structure (from bootstrap):**

```
flux-gitops/
└── clusters/
    ├── kustomize-demo/          ← NOT watched by Flux — ignore this
    └── production/              ← Flux watches THIS path
        └── flux-system/         ← auto-generated, do not edit
            ├── gotk-components.yaml
            ├── gotk-sync.yaml
            └── kustomization.yaml
```

**Why reconciliation wasn't working:** The `namespaces/` directory was placed under
`clusters/kustomize-demo/` which Flux never sees. It must be under `clusters/production/`.

**Target structure after this lab:**

```
flux-gitops/
└── clusters/
    └── production/              ← everything goes here
        ├── flux-system/         ← do not touch
        ├── namespaces/
        │   └── namespaces.yaml
        ├── sources/
        │   └── gitrepository.yaml
        └── apps/
            ├── dev.yaml
            ├── staging.yaml
            └── production.yaml
```

**Reconciliation flow:**

```
gitlab.com/amitopenwriteup/manifest-repo
       │  (polled every 1m by source-controller)
       ▼
   FluxCD — watching clusters/production/ in flux-gitops
       │
       ├──▶ dev namespace        (overlays/dev,        every 1m)
       ├──▶ staging namespace    (overlays/staging,    every 2m)
       └──▶ production namespace (overlays/production, every 5m)
```

---

## Nemespace creation

The `namespaces.yaml` currently in `clusters/kustomize-demo/namespaces/` must be moved:

```bash
git clone <flux repo httpsurl>
cd flux-gitops

# Move to the correct watched path
mkdir -p clusters/production/namespaces
vi    clusters/production/namespaces/namespaces.yaml
```

```
apiVersion: v1
kind: Namespace
metadata:
  name: dev
---
apiVersion: v1
kind: Namespace
metadata:
  name: staging
---
apiVersion: v1
kind: Namespace
metadata:
  name: production
```

```
git commit -m "add"
git push
```

Verify Flux picks it up:

```bash
flux reconcile kustomization flux-system --with-source
kubectl get namespaces | grep -E 'dev|staging|production'
```

---

## Step 1: Declare the GitRepository Source

Tell Flux's `source-controller` to watch `manifest-repo`.
```
mkdir -p clusters/production/sources
vi clusters/production/sources/gitrepository.yaml
```
```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: manifest-repo
  namespace: flux-system
spec:
  interval: 1m
  url: https://gitlab.com/amitopenwriteup/manifest-repo
  ref:
    branch: main
  secretRef:
    name: flux-system       # created by bootstrap — holds your GitLab token
```

```bash

# create the file above, then:
git add clusters/production/sources/
git commit -m "feat: add GitRepository source for manifest-repo"
git push
```

**Verify:**

```bash
flux get sources git
```

Expected:

```
NAME           REVISION        SUSPENDED  READY  MESSAGE
flux-system    main@sha1:...   False      True   stored artifact...
manifest-repo  main@sha1:...   False      True   stored artifact...
```

---

## Step 2: Create Flux Kustomizations per Environment

```bash
mkdir -p clusters/production/apps
```

### Dev — `clusters/production/apps/dev.yaml`

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: my-app-dev
  namespace: flux-system
spec:
  interval: 1m
  sourceRef:
    kind: GitRepository
    name: manifest-repo
  path: ./overlays/dev
  prune: true
  targetNamespace: dev
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: dev-my-app
      namespace: dev
  timeout: 3m
```

### Staging — `clusters/production/apps/staging.yaml`

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: my-app-staging
  namespace: flux-system
spec:
  interval: 2m
  sourceRef:
    kind: GitRepository
    name: manifest-repo
  path: ./overlays/staging
  prune: true
  targetNamespace: staging
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: staging-my-app
      namespace: staging
  timeout: 3m
  dependsOn:
    - name: my-app-dev
```

### Production — `clusters/production/apps/production.yaml`

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: my-app-production
  namespace: flux-system
spec:
  interval: 5m
  sourceRef:
    kind: GitRepository
    name: manifest-repo
  path: ./overlays/production
  prune: true
  targetNamespace: production
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: prod-my-app
      namespace: production
    - apiVersion: autoscaling/v2
      kind: HorizontalPodAutoscaler
      name: prod-my-app
      namespace: production
  timeout: 5m
  dependsOn:
    - name: my-app-staging
  retryInterval: 2m
```

```bash
git add clusters/production/apps/
git commit -m "feat: add Flux Kustomizations for dev, staging, production"
git push
```

---

## Step 3: Watch Flux Reconcile

```bash
flux get kustomizations --watch
```

Expected output:

```
NAME                REVISION        SUSPENDED  READY  MESSAGE
flux-system         main@sha1:abc   False      True   Applied revision: main@sha1:abc
my-app-dev          main@sha1:abc   False      True   Applied revision: main@sha1:abc
my-app-staging      main@sha1:abc   False      True   Applied revision: main@sha1:abc
my-app-production   main@sha1:abc   False      True   Applied revision: main@sha1:abc
```

**Verify resources:**

```bash
kubectl get all -n dev
kubectl get all -n staging
kubectl get all -n production
kubectl get hpa -n production     # HPA only in production
```

---

## Step 4: The GitOps Workflow

### App changes → push to `manifest-repo`

```bash
cd manifest-repo
vim overlays/staging/patch-replicas.yaml   # replicas: 2 → 3
git add -A && git commit -m "scale: bump staging replicas to 3"
git push
# Flux reconciles automatically within 2m
```

### Flux config changes → push to `flux-gitops`

```bash
cd flux-gitops
vim clusters/production/apps/production.yaml   # e.g. change interval
git add -A && git commit -m "config: adjust production interval"
git push
```

### Force immediate reconcile

```bash
flux reconcile source git manifest-repo
flux reconcile kustomization my-app-dev --with-source
flux reconcile kustomization my-app-staging --with-source
flux reconcile kustomization my-app-production --with-source
```

---

## Step 5: Suspend & Resume

```bash
# Freeze production
flux suspend kustomization my-app-production

# Confirm
flux get kustomization my-app-production   # SUSPENDED = True

# Resume
flux resume kustomization my-app-production
```

---


---

## Full Verification Checklist

```bash
flux check                                              # system health
flux get sources git                                    # both repos fetched
flux get kustomizations                                 # all 4 Ready
flux events                                             # recent activity
kubectl get all -n dev
kubectl get all -n staging
kubectl get all -n production
kubectl get hpa -A                                      # only in production
kubectl logs -n flux-system deploy/kustomize-controller -f
```

---



