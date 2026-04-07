# Sealed Secrets + FluxCD Lab Guide
> Encrypt Kubernetes Secrets with Bitnami Sealed Secrets and GitOps Automation via Flux

**3 Labs · Production-Ready**

---

## Table of Contents

- [Overview](#overview)
- [Lab 1 — Install Sealed Secrets](#lab-1--install-sealed-secrets)
- [Lab 2 — Seal & Commit Secrets](#lab-2--seal--commit-secrets)
- [Lab 3 — Flux GitOps Integration](#lab-3--flux-gitops-integration)
- [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## Overview

### Your Actual Repo Structure

```
flux-gitops/
└── clusters/
    └── production/
        ├── apps/                        ← workload manifests
        └── flux-system/
            ├── gotk-components.yaml     ← Flux controllers (generated, do not edit)
            ├── gotk-sync.yaml           ← GitRepository + root Kustomization
            └── kustomization.yaml       ← root kustomize manifest
```

### Target Structure After This Lab

```
flux-gitops/
└── clusters/
    └── production/
        ├── apps/
        ├── flux-system/
        │   ├── gotk-components.yaml
        │   ├── gotk-sync.yaml
        │   └── kustomization.yaml
        ├── infrastructure/              ← NEW: Sealed Secrets controller
        │   ├── sealed-secrets-helmrepo.yaml
        │   ├── sealed-secrets-release.yaml
        │   └── kustomization.yaml
        └── secrets/                     ← NEW: SealedSecret manifests
            └── db-credentials-sealed.yaml
```

### How Sealed Secrets Works

```
Developer                 Git Repository              Kubernetes Cluster
─────────────            ────────────────            ──────────────────
Plain Secret YAML   →    SealedSecret YAML      →    Decrypted Secret
(never committed)        (safe to commit)             (applied by controller)
```

Sealed Secrets uses **asymmetric cryptography**:

- The **public key** (fetched from the cluster) encrypts secrets — safe to share.
- The **private key** (managed by the controller in-cluster) decrypts secrets — never leaves the cluster.
- A `SealedSecret` is a Kubernetes CRD. The controller watches for it, decrypts it, and produces a standard `Secret`.

### Sealed Secrets vs SOPS

| Feature | Sealed Secrets | SOPS + age |
|---|---|---|
| Key management | In-cluster controller | External (age key / KMS) |
| Encryption tool | `kubeseal` CLI | `sops` CLI |
| Output format | `SealedSecret` CRD | Encrypted YAML |
| Cluster dependency | Yes (needs controller for sealing) | No (encrypt offline) |
| Key rotation | Controller handles automatically | Manual re-encryption |

---

## Lab 1 — Install Sealed Secrets

**Objective:** Deploy the Sealed Secrets controller into your cluster using Flux, then install the `kubeseal` CLI.

**Estimated Time:** 20 minutes

### Step 1 — Create the Infrastructure Directory

All controller-level Helm releases go under `clusters/production/infrastructure/` — separate from your app workloads in `apps/`.

```bash
# From the root of your flux-gitops repo
mkdir -p clusters/production/infrastructure
```

---

### Step 2 — Create the HelmRepository Source

```bash
cat > clusters/production/infrastructure/sealed-secrets-helmrepo.yaml <<EOF
---
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: sealed-secrets
  namespace: flux-system
spec:
  interval: 1h
  url: https://bitnami-labs.github.io/sealed-secrets
EOF
```

---

### Step 3 — Create the HelmRelease

```bash
cat > clusters/production/infrastructure/sealed-secrets-release.yaml <<EOF
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: sealed-secrets
  namespace: flux-system
spec:
  interval: 1h
  chart:
    spec:
      chart: sealed-secrets
      version: '>=2.0.0'
      sourceRef:
        kind: HelmRepository
        name: sealed-secrets
        namespace: flux-system
  install:
    crds: Create
  upgrade:
    crds: CreateReplace
  values:
    fullnameOverride: sealed-secrets-controller
EOF
```

---

### Step 4 — Add a Kustomization Resource List for Infrastructure

```bash
cat > clusters/production/infrastructure/kustomization.yaml <<EOF
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - sealed-secrets-helmrepo.yaml
  - sealed-secrets-release.yaml
EOF
```

---

### Step 5 — Wire Infrastructure into the Existing Root Kustomization

Your existing `clusters/production/flux-system/kustomization.yaml` already lists `gotk-components.yaml` and `gotk-sync.yaml`. Add the infrastructure path so Flux picks it up.

Edit `clusters/production/flux-system/kustomization.yaml`:

```yaml
# clusters/production/flux-system/kustomization.yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - gotk-components.yaml
  - gotk-sync.yaml
  - ../infrastructure           # ADD THIS LINE — points to your new infrastructure folder
```

---

### Step 6 — Commit and Push

```bash
git add clusters/production/infrastructure/
git add clusters/production/flux-system/kustomization.yaml
git commit -m 'feat: add Sealed Secrets controller via Flux HelmRelease'
git push

# Reconcile
flux reconcile source git flux-system
flux reconcile kustomization flux-system --with-source
```

---

### Step 7 — Verify the Controller is Running

```bash
# Watch the HelmRelease
flux get helmreleases -n flux-system

# Expected output:
# NAME             REVISION  SUSPENDED  READY  MESSAGE
# sealed-secrets   2.x.x     False      True   Release reconciliation succeeded

# Confirm the controller pod is running
kubectl get pods -n flux-system -l app.kubernetes.io/name=sealed-secrets

# Confirm the CRD was installed
kubectl get crd sealedsecrets.bitnami.com
```

---

### Step 8 — Install the `kubeseal` CLI

```bash
KUBESEAL_VERSION=$(curl -s https://api.github.com/repos/bitnami-labs/sealed-secrets/releases/latest \
  | grep tag_name | cut -d '"' -f 4 | sed 's/v//')

curl -OL "https://github.com/bitnami-labs/sealed-secrets/releases/download/v${KUBESEAL_VERSION}/kubeseal-${KUBESEAL_VERSION}-linux-amd64.tar.gz"

tar -xvzf kubeseal-${KUBESEAL_VERSION}-linux-amd64.tar.gz kubeseal
install -m 755 kubeseal /usr/local/bin/kubeseal

kubeseal --version
```

---

### Step 9 — Fetch and Commit the Public Certificate

```bash
# Fetch the public key from the running controller
kubeseal --fetch-cert \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=flux-system \
  > pub-sealed-secrets.pem

# View the certificate
cat pub-sealed-secrets.pem

# Commit the public cert — it is safe to share
git add pub-sealed-secrets.pem
git commit -m 'feat: add Sealed Secrets public certificate'
git push
```

> **Why commit the public cert?** It allows offline sealing and CI/CD pipelines to encrypt secrets without direct cluster access.

### ✅ Lab 1 Checklist

- [x] `clusters/production/infrastructure/` created with HelmRepository + HelmRelease
- [x] `clusters/production/flux-system/kustomization.yaml` updated to include `../infrastructure`
- [x] Sealed Secrets controller pod running in `flux-system`
- [x] `kubeseal` CLI installed and verified
- [x] Public certificate fetched and committed to Git

---

## Lab 2 — Seal & Commit Secrets

**Objective:** Create a plain Kubernetes Secret, encrypt it with `kubeseal`, and safely push the `SealedSecret` into your repo under `clusters/production/secrets/`.

**Estimated Time:** 25 minutes

### Step 1 — Create the Secrets Directory

```bash
mkdir -p clusters/production/secrets
```

---

### Step 2 — Create a Plain Secret in `/tmp/`

Write the plaintext secret — this file will **never** be committed to Git.

```bash
cat > /tmp/db-credentials-plain.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: production
type: Opaque
stringData:
  DATABASE_URL: postgresql://admin:super-secret-password@db.example.com:5432/mydb
  DATABASE_PASSWORD: super-secret-password
  API_KEY: sk-prod-abc123xyz789
EOF
```

> ⚠️ **Save to `/tmp/`** — not inside your `flux-gitops` repo. This file must never be committed.

---

### Step 3 — Seal the Secret with `kubeseal`

```bash
# Seal using the public cert committed in Lab 1
kubeseal \
  --cert pub-sealed-secrets.pem \
  --format yaml \
  < /tmp/db-credentials-plain.yaml \
  > clusters/production/secrets/db-credentials-sealed.yaml

# Inspect the output — values are now encrypted
cat clusters/production/secrets/db-credentials-sealed.yaml
```

The sealed output looks like this:

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-credentials
  namespace: production
spec:
  encryptedData:
    DATABASE_URL: AgBy3i4OJSWK+PiTySYZZA9rO43cGDEq...
    DATABASE_PASSWORD: AgCIom5GqMHQnPkIW84fmBQmHpR5D...
    API_KEY: AgBDK2qHjy3CdTn7n6LmnMD0G1HjnkZD...
  template:
    metadata:
      name: db-credentials
      namespace: production
    type: Opaque
```

> ✅ This file is safe to commit. The encrypted values can only be decrypted by the controller's private key inside your cluster.

---

### Step 4 — Understanding Scoping

`kubeseal` supports three encryption scopes:

| Scope | Bound To | Flag |
|---|---|---|
| `strict` (default) | Secret name **and** namespace — cannot be renamed or moved | `--scope strict` |
| `namespace-wide` | Namespace only — can be renamed within the same namespace | `--scope namespace-wide` |
| `cluster-wide` | No binding — can be used in any namespace | `--scope cluster-wide` |

```bash
# Example: namespace-wide scope
kubeseal \
  --cert pub-sealed-secrets.pem \
  --scope namespace-wide \
  --format yaml \
  < /tmp/db-credentials-plain.yaml \
  > clusters/production/secrets/db-credentials-sealed.yaml
```

---

### Step 5 — Commit and Push

```bash
git add clusters/production/secrets/db-credentials-sealed.yaml
git commit -m 'feat: add sealed db-credentials secret'
git push

# Remove the plaintext immediately
rm /tmp/db-credentials-plain.yaml
```

---

### Step 6 — Update a Sealed Secret

To rotate or update a secret value, re-create and re-seal.

```bash
cat > /tmp/db-credentials-updated.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: production
type: Opaque
stringData:
  DATABASE_URL: postgresql://admin:NEW-password@db.example.com:5432/mydb
  DATABASE_PASSWORD: NEW-password
  API_KEY: sk-prod-newkey456
EOF

kubeseal \
  --cert pub-sealed-secrets.pem \
  --format yaml \
  < /tmp/db-credentials-updated.yaml \
  > clusters/production/secrets/db-credentials-sealed.yaml

git add clusters/production/secrets/db-credentials-sealed.yaml
git commit -m 'chore: rotate db-credentials secret'
git push

rm /tmp/db-credentials-updated.yaml
```

### ✅ Lab 2 Checklist

- [x] Plain secret created in `/tmp/` (not inside the repo)
- [x] Secret sealed using `pub-sealed-secrets.pem`
- [x] `SealedSecret` written to `clusters/production/secrets/`
- [x] Committed and pushed — plaintext deleted immediately after

---

## Lab 3 — Flux GitOps Integration

**Objective:** Add a Flux `Kustomization` that watches `clusters/production/secrets/` so the controller automatically decrypts and applies any `SealedSecret` committed there.

**Estimated Time:** 25 minutes

### How the Full Flow Works

```
git push SealedSecret YAML
        ↓
source-controller pulls from GitLab
        ↓
kustomize-controller applies SealedSecret CRD to cluster
        ↓
Sealed Secrets controller detects new SealedSecret
        ↓
Decrypts with in-cluster private key
        ↓
Creates plain Secret — available to Pods
```

---

### Step 1 — Add a Kustomization Resource List for Secrets

```bash
cat > clusters/production/secrets/kustomization.yaml <<EOF
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - db-credentials-sealed.yaml
EOF
```

---

### Step 2 — Create the Flux Kustomization for Secrets

```bash
cat > clusters/production/secrets-sync.yaml <<EOF
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: secrets
  namespace: flux-system
spec:
  interval: 5m
  path: ./clusters/production/secrets   # Watches your secrets folder
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system                    # Reuses the GitRepository Flux bootstrap created
    namespace: flux-system
  dependsOn:
    - name: flux-system                  # Wait for Sealed Secrets controller first
  healthChecks:
    - apiVersion: v1
      kind: Secret
      name: db-credentials
      namespace: production
EOF
```

> **Note:** No `decryption` block is needed — unlike SOPS, the `SealedSecret` controller handles decryption automatically as a CRD controller. Flux simply applies the manifest.

---

### Step 3 — Wire Secrets Sync into the Root Kustomization

Update `clusters/production/flux-system/kustomization.yaml` to also include the secrets sync:

```yaml
# clusters/production/flux-system/kustomization.yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - gotk-components.yaml
  - gotk-sync.yaml
  - ../infrastructure          # Sealed Secrets controller (added in Lab 1)
  - ../secrets-sync.yaml       # ADD THIS LINE — Flux Kustomization for secrets
```

---

### Step 4 — Commit and Push

```bash
git add clusters/production/secrets/kustomization.yaml
git add clusters/production/secrets-sync.yaml
git add clusters/production/flux-system/kustomization.yaml
git commit -m 'feat: add secrets Kustomization for Sealed Secrets'
git push

# Force immediate reconciliation
flux reconcile source git flux-system
flux reconcile kustomization flux-system --with-source
```

---

### Step 5 — Watch the Reconciliation

```bash
# Watch all Kustomizations
flux get kustomizations --watch

# Expected output:
# NAME       REVISION        SUSPENDED  READY  MESSAGE
# flux-system main/abc123    False      True   Applied revision: main/abc123
# secrets    main/abc123     False      True   Applied revision: main/abc123

# Confirm the SealedSecret was applied
kubectl get sealedsecret db-credentials -n production

# Confirm the controller decrypted it into a plain Secret
kubectl get secret db-credentials -n production

# Verify the decrypted value
kubectl get secret db-credentials -n production \
  -o jsonpath='{.data.DATABASE_PASSWORD}' | base64 -d
```

---

### Step 6 — Inspect the Controller Logs

```bash
# View Sealed Secrets controller logs
kubectl logs -n flux-system \
  -l app.kubernetes.io/name=sealed-secrets \
  -f | grep -i "unsealed\|error"

# Expected log line on success:
# {"event":"Unsealed","namespace":"production","name":"db-credentials"}
```

---

### Step 7 — Key Rotation (Production Practice)

The controller automatically generates a new sealing key every **30 days**. Old keys are retained so existing `SealedSecrets` can still be decrypted.

```bash
# List all sealing keys in the cluster
kubectl get secrets -n flux-system \
  -l sealedsecrets.bitnami.com/sealed-secrets-key

# After rotation — re-fetch the new public cert
kubeseal --fetch-cert \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=flux-system \
  > pub-sealed-secrets.pem

git add pub-sealed-secrets.pem
git commit -m 'chore: update Sealed Secrets public cert after rotation'
git push
```

> ⚠️ After key rotation, re-seal all existing secrets with the new cert to ensure they can be decrypted if old keys are ever removed.

### ✅ Lab 3 Checklist

- [x] `clusters/production/secrets/kustomization.yaml` created listing all SealedSecrets
- [x] `clusters/production/secrets-sync.yaml` Flux Kustomization created (no `decryption` block)
- [x] `clusters/production/flux-system/kustomization.yaml` updated to include `../secrets-sync.yaml`
- [x] Flux successfully applied the `SealedSecret`
- [x] Controller logs confirm `Unsealed` event
- [x] Plain `Secret` verified in the `production` namespace

---

## Final Repo Structure

After completing all 3 labs your repo looks like:

```
flux-gitops/
├── pub-sealed-secrets.pem               ← public cert (safe to commit)
└── clusters/
    └── production/
        ├── apps/                        ← your workloads (unchanged)
        ├── flux-system/
        │   ├── gotk-components.yaml
        │   ├── gotk-sync.yaml
        │   └── kustomization.yaml       ← updated: includes infrastructure + secrets-sync
        ├── infrastructure/              ← Sealed Secrets Helm controller
        │   ├── sealed-secrets-helmrepo.yaml
        │   ├── sealed-secrets-release.yaml
        │   └── kustomization.yaml
        ├── secrets/                     ← SealedSecret manifests (safe to commit)
        │   ├── db-credentials-sealed.yaml
        │   └── kustomization.yaml
        └── secrets-sync.yaml            ← Flux Kustomization watching secrets/
```

---

## Quick Reference Cheat Sheet

| Task | Command |
|---|---|
| Install kubeseal | `curl -OL https://github.com/bitnami-labs/sealed-secrets/releases/...` |
| Fetch public cert | `kubeseal --fetch-cert --controller-name=sealed-secrets-controller --controller-namespace=flux-system > pub-sealed-secrets.pem` |
| Seal a secret | `kubeseal --cert pub-sealed-secrets.pem --format yaml < /tmp/plain.yaml > clusters/production/secrets/sealed.yaml` |
| Seal (namespace-wide) | `kubeseal --cert pub-sealed-secrets.pem --scope namespace-wide --format yaml < /tmp/plain.yaml > clusters/production/secrets/sealed.yaml` |
| Verify decryption | `kubectl get secret db-credentials -n production -o jsonpath='{.data.DATABASE_PASSWORD}' \| base64 -d` |
| List sealing keys | `kubectl get secrets -n flux-system -l sealedsecrets.bitnami.com/sealed-secrets-key` |
| Force Flux reconcile | `flux reconcile kustomization flux-system --with-source` |
| Watch kustomizations | `flux get kustomizations --watch` |
| Controller logs | `kubectl logs -n flux-system -l app.kubernetes.io/name=sealed-secrets -f` |
| Watch HelmReleases | `flux get helmreleases -n flux-system` |

---

> **Security Reminder:** Never commit plaintext `Secret` YAML to Git. Only commit `SealedSecret` resources. Always write plain YAML to `/tmp/` and delete it immediately after sealing.
