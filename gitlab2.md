# Lab 1: SSH Key Setup & GitLab Integration

## 1. Generate SSH Key Pair

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

> ⚠️ **Note:** Do **not** use `-t rsa`. Use `ed25519` — it is faster, more secure, and modern.

---

## 2. Add the SSH Key to the SSH Agent

**Start the agent:**

```bash
eval "$(ssh-agent -s)"
```

**Add the private key:**

```bash
ssh-add ~/.ssh/id_ed25519
```

---

## 3. Copy the Public SSH Key

```bash
cat ~/.ssh/id_ed25519.pub
```

> Copy the entire output that starts with `ssh-ed25519`.

---

## 4. Add SSH Key to GitLab

1. Log in to **GitLab**.
2. Go to **User Settings** → **SSH Keys**.
3. Paste your copied key into the **Key** field.
4. Click **Add key**.

---

## 5. Create a New Project

1. In GitLab, click **New Project**.
2. Select **Blank Project**.
3. Fill in the project name and settings.
4. Click **Create project**.

---

## 6. Clone a GitLab Repository Using SSH

```bash
git branch -M main
git remote add origin <ssh url>
git push -u origin main
```

> Replace `your-username` and `your-repo` with your actual GitLab username and repository name.
