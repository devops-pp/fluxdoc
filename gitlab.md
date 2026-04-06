# Lab 5: GitLab CI/CD – Your First Pipeline

---

## Overview

Create a `.gitlab-ci.yml` file in your GitLab `my-repo` repository to trigger an automated CI/CD pipeline job.

---

## Step 1: Create the `.gitlab-ci.yml` File Using `vi`

```bash
vi .gitlab-ci.yml
```

> 💡 **Tip:** Press `i` to enter **Insert mode** in `vi` before typing.

---

## Step 2: Add the Following Code

```yaml
job1:
  script:
    - echo "Hello from GitLab CI"
```

> Save and exit `vi` by pressing `Esc`, then typing `:wq` and pressing `Enter`.

---

## Step 3: Add and Commit the File

```bash
git add .gitlab-ci.yml
git commit -m "Adding GitLab CI pipeline file"
git push origin main
```

---

## Step 4: Verify the Job is Running

1. Go to your **GitLab repository** in the browser.
2. Click on **Build** → **Pipelines** from the left sidebar.
3. You should see a pipeline triggered by your commit.
4. Click on the pipeline to view the **job1** status and logs.
5. A green ✅ indicates the job ran successfully.

---

> ⚠️ **Note:** The `.gitlab-ci.yml` file must be in the **root directory** of your repository and use correct **YAML indentation** — otherwise the pipeline will fail.
