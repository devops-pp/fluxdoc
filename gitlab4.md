# Lab 4: GitLab Installation on Ubuntu VM

---

## Prerequisites

- Ubuntu 20.04 or 22.04 VM (recommended)
- At least **4 GB RAM** and **2 CPUs**
- Root or sudo access
- Internet access
- A hostname pointing to your VM's IP (e.g., `gitlab.example.com`) – optional but recommended for external access

---

## Step 1: Update Your System

```bash
sudo apt update 

---

## Step 2: Install Required Dependencies

```bash
sudo apt install -y curl openssh-server ca-certificates tzdata perl

```

---

## Step 3: Add GitLab Repository

```bash
curl https://packages.gitlab.com/install/repositories/gitlab/gitlab-ee/script.deb.sh | sudo bash
```

---

## Step 4: Install GitLab

First, find your VM's IP address:

```bash
ip addr | grep ens
```

Then install GitLab, replacing `http://gitlab.example.com` with your actual domain or public IP:

```bash
sudo EXTERNAL_URL="http://gitlab.example.com" apt install -y gitlab-ee
```

---

## Step 5: Configure GitLab

```bash
sudo gitlab-ctl reconfigure
```

> ⏳ This may take a few minutes. It will set up **NGINX**, **PostgreSQL**, **Redis**, and other required components.

---

## Step 6: Access GitLab

Open a browser and go to:

```
http://<your-domain-or-ip>
```

### Default Admin Credentials

| Field    | Value                                  |
|----------|----------------------------------------|
| Username | `root`                                 |
| Password | Located at `/etc/gitlab/initial_root_password` |

> 💡 **Note:** On the first visit, GitLab will ask you to set the admin password.
