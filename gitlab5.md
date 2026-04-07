# Lab 5b: GitLab Runner – Install, Register & Run

---

## Step 1: Create a New Project Runner in GitLab UI

1. Go to your GitLab project.
2. Navigate to **Settings** → **CI/CD**.
3. Scroll to the **Runners** section.
4. Click **New Project Runner**.
5. Provide the runner details (description, tags, etc.).
6. Click **Create Runner**.

---

## Step 2: Install GitLab Runner

```bash
# Download the binary for your system
sudo curl -L --output /usr/local/bin/gitlab-runner \
  https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-linux-amd64

# Give it permission to execute
sudo chmod +x /usr/local/bin/gitlab-runner

# Create a GitLab Runner user
sudo useradd --comment 'GitLab Runner' --create-home gitlab-runner --shell /bin/bash

# Install and run as a service
sudo gitlab-runner install --user=gitlab-runner --working-directory=/home/gitlab-runner
sudo gitlab-runner start
```

---

## Step 3: Register GitLab Runner

Get the registration token from either:

- **GitLab UI** → Admin Area → Runners → Registration token, **or**
- **Project** → Settings → CI/CD → Runners

Then run:

```bash
sudo gitlab-runner register
```

You will be prompted for the following details:

| Prompt | Example Value |
|--------|---------------|
| GitLab instance URL | `http://gitlab.example.com/` |
| Registration token | `<YOUR_TOKEN>` |
| Description for the runner | `my-runner` |
| Tags for the runner | `docker,build` |
| Executor | `shell`, `docker`, `docker+machine`, `kubernetes` |

### Example – Using Docker Executor

```
executor: shell

```

---

## Step 4: Customize Your `.gitlab-ci.yml`

Update your CI file to use the runner tag:

```yaml
job1:
  tags:
    - myrunner
  script:
    - echo "Hello from GitLab CI"
```

```bash
git add .gitlab-ci.yml
git commit -m "Updated pipeline to use myrunner tag"
git push origin main
```

---

## Step 5: Fix Runner Logout Issue & Start Manually

Edit the runner's bash logout file to remove any conflicting content:

```bash
sudo vi /home/gitlab-runner/.bash_logout
```

> Remove **everything** inside the file, then save and exit with `Esc` → `:wq` → `Enter`.

Then start the runner manually:

```bash
sudo gitlab-runner run
```

---

## Step 6: Check the Pipeline Output

1. Go to your **GitLab repository** in the browser.
2. Click **Build** → **Pipelines** from the left sidebar.
3. Click on the latest pipeline to view job logs.
4. A green ✅ confirms the runner executed the job successfully.

---

> ⚠️ **Note:** Make sure the tag in `.gitlab-ci.yml` (e.g., `myrunner`) matches the tag assigned to your registered runner — otherwise the job will remain **stuck** and never execute.
