# Lab 10: GitLab CI/CD – Artifacts

---

## Overview

In this lab, you will create a CI/CD pipeline that installs `cowsay`, generates a text output file, and stores it as a **GitLab artifact** that can be downloaded after the job completes.
---

sudo su

#Cloud@123$

vi /etc/sudoers

gitlab-runner  ALL=(ALL) NOPASSWD: ALL
# esc :wq!


---

## Step 1: Update `.gitlab-ci.yml`

Open your existing `.gitlab-ci.yml` file:

```bash
vi .gitlab-ci.yml
```

Replace the contents with the following:

```yaml
stages:
  - hello

say-hello:
  stage: hello
  tags:
    - myrunner
  before_script:
    - sudo apt-get update -y
    - sudo apt-get install -y cowsay
  script:
    - /usr/games/cowsay "hello from GitLab CI" > output.txt
  artifacts:
    paths:
      - output.txt
    expire_in: 1 hour
  after_script:
    - sudo apt-get remove -y cowsay
```

> Save and exit `vi` by pressing `Esc`, then typing `:wq` and pressing `Enter`.

---

## Step 2: Commit and Push

```bash
git add .gitlab-ci.yml
git commit -m "Lab 10 - CI pipeline with cowsay artifact"
git push origin main
```

---

## Step 3: Pipeline Breakdown

| Section | Purpose |
|---|---|
| `stages` | Defines the pipeline stage named `hello` |
| `tags` | Routes the job to the runner tagged `myrunner` |
| `before_script` | Runs before the main script — updates packages and installs `cowsay` |
| `script` | Runs `cowsay` and saves its output to `output.txt` |
| `artifacts` | Stores `output.txt` for download, expires after **1 hour** |
| `after_script` | Cleans up by removing `cowsay` after the job |

---

## Step 4: View and Download the Artifact

1. Go to your **GitLab repository** in the browser.
2. Click **Build** → **Pipelines** from the left sidebar.
3. Click on the latest pipeline run.
4. Click on the **say-hello** job.
5. On the right side, click **Browse** or **Download** under the **Artifacts** section.
6. Download and open `output.txt` to see the cowsay output.

---

## Expected Output in `output.txt`

```
 ______________________
< hello from GitLab CI >
 ----------------------
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||
```

---

> 💡 **Tip:** Artifacts are useful for storing build outputs, test reports, compiled binaries, or any file generated during a pipeline that you want to access after the job finishes.

> ⚠️ **Note:** The artifact will **expire after 1 hour** as configured. Increase `expire_in` (e.g., `1 week`, `30 days`) if you need longer retention.
