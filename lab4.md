# FluxCD Helm Lab
### Deploy nginx via Bitnami Helm Chart
### Repo: flux-gitops | Cluster: clusters/production

---

## Repo Structure

```
flux-gitops/
└── clusters/
    └── production/
        ├── flux-system/               ← existing bootstrap (do not touch)
        └── flux-helm-lab/             ← add these files
            ├── kustomization.yaml
            ├── 01-helmrepository.yaml
            ├── 02-values-configmap.yaml
            ├── 03-helmrelease.yaml
            └── 04-kustomization.yaml
```

---

## Step 1 — Create App Folder

```bash
git clone https://github.com/<YOUR_USER>/flux-gitops.git
cd flux-gitops
mkdir -p clusters/production/flux-helm-lab
```

---

## Step 2 — Create Manifests

### clusters/production/flux-helm-lab/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - 01-helmrepository.yaml
  - 02-values-configmap.yaml
  - 03-helmrelease.yaml
  - 04-kustomization.yaml
```

---

### clusters/production/flux-helm-lab/01-helmrepository.yaml

> OCI URL requires `type: oci` — using `type: default` with an `oci://` URL
> is what caused all previous errors.

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: helmrepo
  namespace: flux-system
  annotations:
    reconcile.fluxcd.io/requestedAt: "1"
spec:
  type: oci                                        # ← must be oci when URL starts with oci://
  url: oci://registry-1.docker.io/bitnamicharts   # ← no trailing slash
  interval: 5m
  timeout: 60s
```

---

### clusters/production/flux-helm-lab/02-values-configmap.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-helm-values
  namespace: flux-system
data:
  values.yaml: |
    replicaCount: 1
    service:
      type: ClusterIP
```

---

### clusters/production/flux-helm-lab/03-helmrelease.yaml

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: myhelm
  namespace: flux-system
spec:
  targetNamespace: myhelm-nginx
  interval: 5m
  chart:
    spec:
      chart: nginx
      version: "22.6.10"
      sourceRef:
        kind: HelmRepository
        name: helmrepo
        namespace: flux-system
      interval: 10m
  install:
    createNamespace: true
    remediation:
      retries: 3
  upgrade:
    remediation:
      retries: 3
      remediateLastFailure: true
    cleanupOnFail: true
  valuesFrom:
    - kind: ConfigMap
      name: nginx-helm-values
      valuesKey: values.yaml
```

---

### clusters/production/flux-helm-lab/04-kustomization.yaml

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: flux-helm-lab
  namespace: flux-system
spec:
  interval: 5m
  prune: false
  sourceRef:
    kind: GitRepository
    name: flux-system
    namespace: flux-system
  path: "./clusters/production/flux-helm-lab"
```

---

## Step 3 — Commit and Push

```bash
git add clusters/production/flux-helm-lab/
git commit -m "Fix HelmRepository type to oci"
git push origin main
```

---

## Step 4 — Clean Up and Reconcile

```bash
# Delete stale objects from cluster
kubectl delete helmrelease myhelm -n flux-system --ignore-not-found
kubectl delete helmrepository helmrepo -n flux-system --ignore-not-found
kubectl delete helmchart flux-system-myhelm -n flux-system --ignore-not-found

# Reconcile from Git
flux reconcile kustomization flux-system --with-source -n flux-system

# Watch
flux get helmreleases -n flux-system --watch
```

---

## Step 5 — Verify

```bash
flux get sources helm -n flux-system
# NAME       READY   STATUS
# helmrepo   True    Helm repository is ready

flux get helmreleases -n flux-system
# NAME     REVISION   READY   MESSAGE
# myhelm   22.6.10    True    Release reconciliation succeeded

kubectl get pods -n myhelm-nginx
kubectl get svc -n myhelm-nginx
```

---

## Root Cause Summary

| What was wrong | Fix |
|---|---|
| `type: default` + `oci://` URL | Must use `type: oci` with OCI URLs |
| `type: oci` + old Flux version | Flux v0.32+ required for OCI — check with `flux version` |
| Stale cluster objects overriding Git | Force delete + reconcile |
| Complex values causing YAML error | Minimal values only |

---

## Useful Commands

```bash
# Force reconcile
flux reconcile kustomization flux-system --with-source -n flux-system

# Force reconcile HelmRelease only
flux reconcile helmrelease myhelm -n flux-system

# Suspend / resume
flux suspend helmrelease myhelm -n flux-system
flux resume helmrelease myhelm -n flux-system

# View all Flux resources
flux get all -n flux-system

# View logs
flux logs -n flux-system --follow

# Describe HelmRelease
kubectl describe helmrelease myhelm -n flux-system
```
