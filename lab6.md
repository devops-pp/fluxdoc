# GitLab CI/CD Workshop — Lab 6
### Flux Image Automation with GitLab Container Registry

---

## Overview

```
GitLab Registry (image ready)   →   Flux watches for new tags
        ↓
Flux detects new image   →   Updates deployment manifest in Git   →   Rolls out to cluster
```

> **Prerequisites:**
> - Flux is bootstrapped and running on your cluster (`flux check` passes)
> - Your Flask app image is already being pushed to `registry.gitlab.com/amit/cicd`
> - A Kubernetes cluster is accessible via `kubectl`

---

## Step 6 — Create Tokens & Registry Secret for Flux

Flux needs two things: a **Personal Access Token** to write back to Git (for image update commits), and a **Deploy Token** to read from the Container Registry.

---

### 6A — Create a GitLab Personal Access Token (for Git write access)

Flux's `ImageUpdateAutomation` commits updated image tags back to `flux-gitops`. It needs a token with Git write permission.

1. Click your **avatar** (top-right) → **Edit profile**
2. In the left sidebar go to **Access Tokens**
3. Click **Add new token**

   | Field | Value |
   |-------|-------|
   | Token name | `flux-git-writer` |
   | Expiration date | Set as needed (e.g. 1 year) |
   | Scopes | ✓ `api` and ✓ `write_repository` |

4. Click **Create personal access token**
5. Copy the token immediately — it is shown only once

> This token was already used during `flux bootstrap gitlab` (Lab prerequisite). If bootstrap is already complete, you can reuse that same token here — no need to create a new one.

---

### 6B — Create a GitLab Deploy Token (for registry read access)

A Deploy Token is scoped to a single project and grants Flux read-only access to the Container Registry — safer than using your personal token.

1. Go to your `cicd` GitLab project → **Settings** → **Repository**
2. Scroll down and expand **Deploy tokens**
3. Click **Add token**

   | Field | Value |
   |-------|-------|
   | Name | `flux-image-reader` |
   | Username | Leave blank (GitLab auto-generates) |
   | Expiration date | Set as needed |
   | Scopes | ✓ `read_registry` only |

4. Click **Create deploy token**
5. Copy both the **username** (e.g. `gitlab+deploy-token-12345`) and the **token value** — shown only once

> **Deploy Token vs Personal Access Token**
>
> | | Deploy Token | Personal Access Token |
> |---|---|---|
> | Scope | Single project | Entire account |
> | Use here | Read registry images | Write commits to Git |
> | Revocation impact | Only this project | All services using it |

---

### 6C — Create the Kubernetes Registry Secret

Use the Deploy Token from Step 6B to create a secret Flux will use to scan the registry:

```bash
kubectl create secret docker-registry gitlab-registry-secret \
  --docker-server=registry.gitlab.com \
  --docker-username=<deploy-token-username> \
  --docker-password=<deploy-token-value> \
  --namespace=flux-system
```

Verify the secret was created:

```bash
kubectl get secret gitlab-registry-secret -n flux-system
```

---

## Step 6D — Enable Image Automation Controllers

The image-reflector and image-automation controllers are **not installed by default** with `flux bootstrap`. Without them, `ImageRepository`, `ImagePolicy`, and `ImageUpdateAutomation` resources will not work.

First, verify which controllers are currently running:

```bash
kubectl get pods -n flux-system
```

If you only see these four — the image controllers are missing:

```
helm-controller-xxx           1/1   Running
kustomize-controller-xxx      1/1   Running
notification-controller-xxx   1/1   Running
source-controller-xxx         1/1   Running
```

Re-run bootstrap with the `--components-extra` flag to add the missing controllers:

```bash
flux bootstrap gitlab \
  --owner=amit \
  --repository=flux-gitops \
  --branch=main \
  --path=clusters/production \
  --token-auth \
  --personal \
  --components-extra=image-reflector-controller,image-automation-controller
```

After it completes, verify all six controllers are running:

```bash
kubectl get pods -n flux-system
```

Expected output:

```
helm-controller-xxx                  1/1   Running
kustomize-controller-xxx             1/1   Running
notification-controller-xxx          1/1   Running
source-controller-xxx                1/1   Running
image-reflector-controller-xxx       1/1   Running
image-automation-controller-xxx      1/1   Running
```

> Re-running bootstrap is safe — it is idempotent and will not affect existing Flux resources.

---

## Step 7 — Deploy the Flask App to Kubernetes

Create a `k8s/` folder in your `cicd` project with a deployment manifest.

### k8s/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-app
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: flask-app
  template:
    metadata:
      labels:
        app: flask-app
    spec:
      containers:
        - name: flask-app
          image: registry.gitlab.com/amit/cicd:latest  # {"$imagepolicy": "flux-system:flask-app"}
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: flask-app
  namespace: default
spec:
  selector:
    app: flask-app
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
```

> The comment `# {"$imagepolicy": "flux-system:flask-app"}` is a **marker** that tells Flux exactly which line to rewrite when a new image tag is detected. Do not remove it.

Apply the manifest to your cluster:

```bash
kubectl apply -f k8s/deployment.yaml
kubectl get pods
```

---

## Step 8 — Create Flux Image Automation Resources

Clone your `flux-gitops` repo and create the apps folder:

```bash
git clone https://gitlab.com/amit/flux-gitops.git
cd flux-gitops
mkdir -p clusters/production/apps
```

Add the following three files inside `clusters/production/apps/`.

---

### image-repository.yaml

```yaml
apiVersion: image.toolkit.fluxcd.io/v1
kind: ImageRepository
metadata:
  name: flask-app
  namespace: flux-system
spec:
  image: registry.gitlab.com/amit/cicd
  interval: 1m
  secretRef:
    name: gitlab-registry-secret
```

This tells Flux: *"Scan this registry every 1 minute and list all available tags."*

---

### image-policy.yaml

```yaml
apiVersion: image.toolkit.fluxcd.io/v1
kind: ImagePolicy
metadata:
  name: flask-app
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: flask-app
  policy:
    alphabetical:
      order: asc
```

> We use `alphabetical` ordering because our tags are commit SHAs (e.g. `a1b2c3d4`). For semver tags like `v1.2.3`, use:
>
> ```yaml
> policy:
>   semver:
>     range: ">=0.0.1"
> ```

---

### image-update-automation.yaml

```yaml
apiVersion: image.toolkit.fluxcd.io/v1
kind: ImageUpdateAutomation
metadata:
  name: flask-app
  namespace: flux-system
spec:
  interval: 1m
  sourceRef:
    kind: GitRepository
    name: flux-system
  git:
    checkout:
      ref:
        branch: main
    commit:
      author:
        email: fluxbot@example.com
        name: Flux Bot
      messageTemplate: "chore: update flask-app image to {{range .Updated.Images}}{{.}}{{end}}"
    push:
      branch: main
  update:
    path: ./clusters/production
    strategy: Setters
```

---

Commit and push all three files:

```bash
git add clusters/production/apps/
git commit -m "Add Flux image automation for Flask app"
git push origin main
```

Flux will automatically pick up the new manifests within its reconciliation interval (default 1 minute).

---

## Step 9 — Verify Image Automation

Check that Flux can reach and scan the registry:

```bash
flux get image repository flask-app
```

Expected output:
```
NAME        LAST SCAN   TAGS   READY   MESSAGE
flask-app   10s ago     3      True    successful scan: found 3 tags
```

Check the image policy has resolved a latest tag:

```bash
flux get image policy flask-app
```

Expected output:
```
NAME        LATEST IMAGE                               READY
flask-app   registry.gitlab.com/amit/cicd:a1b2c3d4    True
```

Check the update automation is active:

```bash
flux get image update flask-app
```

---

## Step 10 — End-to-End Test

Make a small change to `app.py` in your `cicd` project:

```python
@app.route('/')
def hello_world():
    return 'Hello, Flux!'   # updated
```

Push the change:

```bash
git add app.py
git commit -m "Update greeting for Flux demo"
git push origin main
```

What happens automatically:

```
1. GitLab CI builds new image → tagged with new commit SHA → pushed to registry
2. Flux ImageRepository scans registry → detects new tag (within 1 min)
3. Flux ImagePolicy selects the new tag as latest
4. Flux ImageUpdateAutomation commits updated image ref to flux-gitops repo
5. Flux reconciles the cluster → triggers rolling update of flask-app deployment
```

Watch the rollout:

```bash
kubectl rollout status deployment/flask-app
```

Test the running app:

```bash
kubectl port-forward svc/flask-app 8080:80
curl http://localhost:8080
# Hello, Flux!
```

---

## File Structure (End State)

```
cicd/                            ← app repo (gitlab.com/amit/cicd)
├── app.py
├── requirements.txt
├── dockerfile
├── .gitlab-ci.yml
└── k8s/
    └── deployment.yaml          ← contains $imagepolicy marker

flux-gitops/                     ← Flux config repo (gitlab.com/amit/flux-gitops)
└── clusters/
    └── production/
        ├── flux-system/         ← auto-generated by bootstrap
        └── apps/
            ├── image-repository.yaml
            ├── image-policy.yaml
            └── image-update-automation.yaml
```

---

## Troubleshooting

| Error | Fix |
|-------|-----|
| `ImageRepository` not ready | Check deploy token has `read_registry` scope; run `flux logs --kind=ImageRepository` |
| `unauthorized` when Flux scans | Recreate `gitlab-registry-secret` with correct deploy token username and password |
| Image tag not updating in Git | Ensure `# {"$imagepolicy": "flux-system:flask-app"}` comment is on the same line as `image:` |
| `ImageUpdateAutomation` not committing | Verify Flux has write access to `flux-gitops` repo; check bootstrap token permissions |
| Pod stuck in `ImagePullBackOff` | Add `imagePullSecrets` in `deployment.yaml` pointing to a registry secret in `default` namespace |
| Flux not picking up new manifests | Run `flux reconcile source git flux-system` to force an immediate sync |
