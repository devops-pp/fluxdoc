# Kustomize + kind: Environment-Specific Configurations

## Overview

This guide walks you through a hands-on practice setup using Kustomize for managing environment-specific Kubernetes configurations with a local kind (Kubernetes in Docker) cluster.

---

## Prerequisites

```bash
# Install kind
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.22.0/kind-linux-amd64
chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# Kustomize (built into kubectl 1.14+, or install standalone)
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
sudo mv kustomize /usr/local/bin/

# Verify
kind version
kubectl version --client
kustomize version
```

---

## Step 1: Create a kind Cluster

```bash
# Simple single-node cluster
kind create cluster --name kustomize-demo

# OR multi-node cluster (recommended for realistic testing)
cat <<EOF > kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
EOF

kind create cluster --name kustomize-demo --config kind-config.yaml

# Verify cluster is running
kubectl cluster-info --context kind-kustomize-demo
kubectl get nodes
```

---

## Step 2: Project Structure

Create this directory layout — it's the standard Kustomize pattern:

```bash
mkdir -p my-app/base my-app/overlays/{dev,staging,production}
cd my-app
```

The resulting structure:

```
my-app/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/
    │   ├── patch-replicas.yaml
    │   └── kustomization.yaml
    ├── staging/
    │   ├── patch-replicas.yaml
    │   ├── patch-resources.yaml
    │   └── kustomization.yaml
    └── production/
        ├── patch-replicas.yaml
        ├── patch-resources.yaml
        ├── patch-hpa.yaml
        └── kustomization.yaml
```

---

## Step 3: Create the Base

### `base/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:
    app: my-app
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
              valueFrom:
                configMapKeyRef:
                  name: my-app-config
                  key: APP_ENV
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "200m"
              memory: "256Mi"
```

### `base/service.yaml`

```yaml
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
  type: ClusterIP
```

### `base/configmap.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-app-config
data:
  APP_ENV: "base"
  LOG_LEVEL: "info"
  MAX_CONNECTIONS: "100"
```

### `base/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml

# Add common labels to ALL resources
commonLabels:
  managed-by: kustomize
  team: platform
```

---

## Step 4: Dev Overlay

### `overlays/dev/patch-replicas.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 1                  # Dev: minimal replicas
```

### `overlays/dev/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# Point to base
bases:
  - ../../base

# Add dev namespace
namespace: dev

# Prefix all resource names
namePrefix: dev-

# Add dev-specific labels
commonLabels:
  environment: dev

# Override ConfigMap values
configMapGenerator:
  - name: my-app-config
    behavior: merge
    literals:
      - APP_ENV=development
      - LOG_LEVEL=debug
      - MAX_CONNECTIONS=10

# Apply patches
patches:
  - path: patch-replicas.yaml
```

---

## Step 5: Staging Overlay

### `overlays/staging/patch-replicas.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 2
```

### `overlays/staging/patch-resources.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      containers:
        - name: my-app
          resources:
            requests:
              cpu: "200m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
```

### `overlays/staging/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
  - ../../base

namespace: staging
namePrefix: staging-

commonLabels:
  environment: staging

configMapGenerator:
  - name: my-app-config
    behavior: merge
    literals:
      - APP_ENV=staging
      - LOG_LEVEL=warn
      - MAX_CONNECTIONS=50

patches:
  - path: patch-replicas.yaml
  - path: patch-resources.yaml
```

---

## Step 6: Production Overlay

### `overlays/production/patch-replicas.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 5
```

### `overlays/production/patch-resources.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      containers:
        - name: my-app
          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              cpu: "1000m"
              memory: "1Gi"
```

### `overlays/production/patch-hpa.yaml`

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 5
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

### `overlays/production/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
  - ../../base

namespace: production
namePrefix: prod-

commonLabels:
  environment: production

configMapGenerator:
  - name: my-app-config
    behavior: merge
    literals:
      - APP_ENV=production
      - LOG_LEVEL=error
      - MAX_CONNECTIONS=500

# Production also adds an HPA
resources:
  - patch-hpa.yaml

patches:
  - path: patch-replicas.yaml
  - path: patch-resources.yaml
```

---

## Step 7: Preview & Deploy

### Preview (dry-run) — always do this first!

```bash
# See what dev will look like
kubectl kustomize overlays/dev

# See staging diff
kubectl kustomize overlays/staging

# Production
kubectl kustomize overlays/production
```

### Create namespaces

```bash
kubectl create namespace dev
kubectl create namespace staging
kubectl create namespace production
```

### Deploy to dev

```bash
kubectl apply -k overlays/dev
kubectl get all -n dev
```

### Deploy to staging

```bash
kubectl apply -k overlays/staging
kubectl get all -n staging
```

### Deploy to production

```bash
kubectl apply -k overlays/production
kubectl get all -n production
```

---

## Step 8: Verify Differences

```bash
# Compare replica counts across environments
kubectl get deploy -n dev
kubectl get deploy -n staging
kubectl get deploy -n production

# Check ConfigMap values per environment
kubectl get configmap -n dev -o yaml | grep -A5 "data:"
kubectl get configmap -n staging -o yaml | grep -A5 "data:"

# Verify resource limits differ
kubectl get deploy prod-my-app -n production \
  -o jsonpath='{.spec.template.spec.containers[0].resources}' | jq

# Check HPA only exists in production
kubectl get hpa -A
```

---

## Advanced Techniques

### Strategic Merge Patch vs JSON 6902 Patch

```yaml
# JSON 6902 patch — more surgical, targets specific fields
patches:
  - target:
      kind: Deployment
      name: my-app
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 3
      - op: add
        path: /spec/template/spec/containers/0/env/-
        value:
          name: NEW_VAR
          value: "hello"
```

### Image Tag Management per Environment

```yaml
# In overlays/production/kustomization.yaml
images:
  - name: nginx
    newTag: "1.25-stable"      # Pin to stable tag in prod

# In overlays/dev/kustomization.yaml
images:
  - name: nginx
    newTag: "latest"           # Use latest in dev
```

### Secret Generation

```yaml
# In any kustomization.yaml (never commit actual secrets!)
secretGenerator:
  - name: db-credentials
    literals:
      - DB_USER=admin
      - DB_PASSWORD=changeme   # In real life: use external secret managers
    type: Opaque
```

### Component Reuse (Kustomize Components)

```yaml
# components/monitoring/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component
resources:
  - servicemonitor.yaml

# Then in staging/production overlays only:
components:
  - ../../components/monitoring
```

---

## Cleanup

```bash
# Remove deployments
kubectl delete -k overlays/dev
kubectl delete -k overlays/staging
kubectl delete -k overlays/production

# Delete namespaces
kubectl delete namespace dev staging production

# Destroy kind cluster
kind delete cluster --name kustomize-demo
```

---

## Key Takeaways

| Concept | What It Does |
|---|---|
| `base/` | Single source of truth for all shared manifests |
| `overlays/` | Environment-specific changes layered on top |
| `namePrefix` | Prevents resource name collisions across envs |
| `namespace` | Isolates environments within the same cluster |
| `configMapGenerator` | Generates versioned ConfigMaps, avoids stale config |
| `patches` | Strategic merge patches override only what you specify |
| `images` | Swap image tags per environment without editing manifests |
| `components` | Reusable add-ons (e.g., monitoring) included selectively |

> **Golden Rule:** `base/` should work standalone. `overlays/` should only contain what differs.
