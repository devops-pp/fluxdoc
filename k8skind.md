Here is your content cleaned and formatted in **Markdown (.md)** so you can directly copy and save as a downloadable file (e.g., `kind-setup.md`):

---

````markdown
# KIND (Kubernetes in Docker) Setup Guide

## Install KIND

Execute the below script on the host to install the `kind` command.

### Switch to root user
```bash
sudo su
# Password: Cloud@123$
````

### Installation Script

```bash
#!/bin/bash

# For AMD64 / x86_64
[ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64

chmod +x ./kind
sudo cp ./kind /usr/local/bin/kind
rm -rf kind
```

### Verify Installation

```bash
kind --version
```

Expected Output:

```
kind version 0.20.0
```

---

## Delete Cluster

```bash
kind delete cluster
```

---

## Create Multi-Node Cluster

### Create Config File

Create a file named `config.yaml`:

```yaml
# 4 node (3 workers) cluster config
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4

nodes:
- role: control-plane
  image: kindest/node:v1.35.1
- role: worker
  image: kindest/node:v1.35.1
```

### Create Cluster

```bash
kind create cluster --config=config.yaml
```

Expected Output:

```
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.28.0)
 ✓ Preparing nodes
 ✓ Writing configuration
 ✓ Starting control-plane
 ✓ Installing CNI
 ✓ Installing StorageClass
 ✓ Joining worker nodes
```

---

## Set kubectl Context

```bash
kubectl cluster-info --context kind-kind
```

---

## Check Cluster Status

### Using KIND

```bash
kind get clusters
```

### Using kubectl

```bash
kubectl cluster-info
```

Example Output:

```
Kubernetes control plane is running at https://127.0.0.1:41273
CoreDNS is running at https://127.0.0.1:41273/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

---

## Check Node Status

```bash
kubectl get nodes
```

Example Output:

```
NAME                 STATUS   ROLES           AGE   VERSION
kind-control-plane   Ready    control-plane   22m   v1.28.0
kind-worker          Ready    <none>          21m   v1.28.0
kind-worker2         Ready    <none>          21m   v1.28.0
kind-worker3         Ready    <none>          21m   v1.28.0
```

---

## Check Kubernetes Version

```bash
kubectl version
```

Example Output:

```
Client Version: v1.28.3
Kustomize Version: v5.0.4
```

> Note: Cluster version (v1.28.0) may differ slightly from client version.

---

## Install Helm

### Download Helm

```bash
wget https://get.helm.sh/helm-v3.10.2-linux-amd64.tar.gz
```

### Extract

```bash
tar -zxvf helm-v3.10.2-linux-amd64.tar.gz
```

### Move Binary

```bash
mv linux-amd64/helm /usr/local/bin/helm
```

### Verify Installation

```bash
helm version
```

---

## Summary

* KIND installed and verified
* Multi-node Kubernetes cluster created
* kubectl configured
* Helm installed for package management

---

```

---

If you want, I can also generate a **downloadable `.md` file link** or convert this into a **PDF / DOCX for training material**.
```
