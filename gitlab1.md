Here's your Lab 1 content formatted in Markdown:

```markdown
# Lab 1: Git Basics

## 1. Login as Root User

```
sudo su -
```

> Enter the same password used for your Ubuntu server when prompted.

---

## 2. Check Git Version

```bash
git --version
```

---

## 3. Configure Git Global Parameters

Set your name and email (used in Git commits):

```bash
git config --global user.name "amit"
git config --global user.email "amit@ow.com"
```

View current configuration:

```bash
git config --global --list
```

Set the default editor for Git (optional):

```bash
export EDITOR=vi
```

---

## 4. Create and Initialize a Local Git Repository

```bash
cd ~          # Go to home directory
mkdir my-repo # Create a directory
cd my-repo    # Move into that directory
git status    # Check current git status (will show not a repo yet)
git init      # Initialize as a git repo
ls -al        # Show hidden files (you'll see .git folder)
git status    # Check status again (now it's a Git repo)
```

---

## 5. Git File Tracking Commands

**Create a new file:**

```bash
touch Readme
```

**Check status:**

```bash
git status
```

> You'll see `Readme` under **Untracked files**.

**Add the file to staging:**

```bash
git add Readme
git status
```

**Commit the file:**

```bash
git commit -m "Adding file"
git status
```

**Check commit history:**

```bash
git log
```
```

You can copy this directly into any `.md` file or Markdown editor. Let me know if you'd like this saved as a downloadable file or formatted differently!
