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

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: helmrepo
  namespace: flux-system
spec:
  type: default
  url: https://charts.bitnami.com/bitnami
  interval: 10m
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
    image:
      registry: docker.io
      repository: bitnami/nginx
      tag: ""
    service:
      type: ClusterIP
      port: 80
    ingress:
      enabled: false
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 250m
        memory: 256Mi
    autoscaling:
      enabled: false
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
      version: "22.6.10"           # matches: helm install my-nginx bitnami/nginx --version 22.6.10
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

## Step 3 — Clean Up Old Broken Resources

```bash
kubectl delete helmrelease myhelm -n flux-system
kubectl delete helmrepository helmrepo -n flux-system
kubectl delete kustomization my-app -n flux-system
```

---

## Step 4 — Commit and Push

```bash
git add clusters/production/flux-helm-lab/
git commit -m "Add nginx HelmRelease bitnami/nginx 22.6.10"
git push origin main
```

---

## Step 5 — Reconcile and Verify

```bash
# Trigger sync
flux reconcile kustomization flux-system --with-source -n flux-system

# Check HelmRepository
flux get sources helm -n flux-system
# NAME       READY   STATUS
# helmrepo   True    Fetched revision: ...

# Check HelmRelease
flux get helmreleases -n flux-system
# NAME     READY   STATUS
# myhelm   True    Release reconciliation succeeded

# Check pods
kubectl get pods -n myhelm-nginx
kubectl get svc -n myhelm-nginx
```

---

## Helm Equivalent Reference

| Helm CLI | FluxCD |
|----------|--------|
| `helm repo add bitnami https://charts.bitnami.com/bitnami` | `HelmRepository` |
| `helm install my-nginx bitnami/nginx --version 22.6.10` | `HelmRelease` |
| manual `helm install` | `install.createNamespace: true` + automated sync |

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

# Describe HelmRelease events
kubectl describe helmrelease myhelm -n flux-system
```
