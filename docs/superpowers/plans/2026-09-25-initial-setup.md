# SLiMS 9 Customization Initial Setup & CI/CD Pipeline Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Establish clean customization isolation (custom template & plugins whitelist), revert core hacks, configure dual-branch git strategy, and implement automated GitHub Actions CI/CD to deploy to the live Docker Compose environment at `library.stiepemuda.ac.id`.

**Architecture:** Maintain an unpolluted `master` branch mirroring upstream SLiMS, a `production` branch triggering GitHub Actions deployment, and a `custom-pemuda` development branch. Deployment connects to `vps-pemuda` via SSH, executes `git reset --hard origin/production` in the live-mounted web root (`/home/itadmin/apps/library-pemuda/data/files`), and restarts `library-web` in ~2 seconds with zero volume reset.

**Tech Stack:** PHP 8.3, Apache, Docker Compose, MariaDB 10.6, SLiMS 9 Bulian, GitHub Actions (`appleboy/ssh-action`), Git.

## Global Constraints
- Do not modify core SLiMS files in `admin/`, `lib/`, `simbio2/`, or `sysconfig.inc.php`.
- Preserve Docker volume integrity (`dbdata`, `./data/db`, `./data/files/`).
- Local config files (`config/database.php`, `config/env.php`) must remain untracked in `.gitignore`.
- Whitelist custom assets: `template/stie_pemuda/` and `plugins/stie_pemuda_*/`.

---

### Task 1: Setup Branching Strategy (`production` & `custom-pemuda`)

**Files:**
- Create branches: `production`, `custom-pemuda`

**Interfaces:**
- Consumes: Current `master` branch (commit `9186e77e`)
- Produces: `production` branch for VPS deployments and `custom-pemuda` for feature work

- [ ] **Step 1: Check current git status and ensure clean working directory**

Run in terminal:
```powershell
git status
```
Expected output: clean (except modified `plugins/label_barcode/index.php` which will be addressed in Task 2).

- [ ] **Step 2: Create and switch to `production` branch**

Run:
```powershell
git checkout -b production
```
Expected: `Switched to a new branch 'production'`

- [ ] **Step 3: Create and switch to `custom-pemuda` development branch**

Run:
```powershell
git checkout -b custom-pemuda
```
Expected: `Switched to a new branch 'custom-pemuda'`

- [ ] **Step 4: Verify local branches**

Run:
```powershell
git branch
```
Expected output listing `* custom-pemuda`, `production`, and `master`.

---

### Task 2: Revert Core Mod and Calibrate `.gitignore`

**Files:**
- Modify: `plugins/label_barcode/index.php`
- Modify: `.gitignore`

**Interfaces:**
- Consumes: Existing git tracking rules in `.gitignore`
- Produces: Clean core repository and whitelist for custom templates/plugins

- [ ] **Step 1: Revert uncommitted changes in `plugins/label_barcode/index.php`**

Run:
```powershell
git checkout -- plugins/label_barcode/index.php
```

- [ ] **Step 2: Verify `plugins/label_barcode/index.php` is pristine**

Run:
```powershell
git status -s
```
Expected output: empty (working tree clean).

- [ ] **Step 3: Update `.gitignore` to whitelist `template/stie_pemuda/` and custom plugins**

In `.gitignore`, locate lines 27-31:
```gitignore
plugins/*
!plugins/read_counter/

template/*
```

Update to:
```gitignore
plugins/*
!plugins/read_counter/
!plugins/stie_pemuda_*/

template/*
!template/stie_pemuda/
```

- [ ] **Step 4: Verify `.gitignore` changes**

Run:
```powershell
git diff .gitignore
```
Verify the additions of `!plugins/stie_pemuda_*/` and `!template/stie_pemuda/`.

- [ ] **Step 5: Commit `.gitignore` update**

Run:
```powershell
git add .gitignore
git commit -m "build(git): whitelist custom template and plugins in gitignore"
```

---

### Task 3: Scaffold Custom OPAC Template (`template/stie_pemuda`)

**Files:**
- Create: `template/stie_pemuda/` (clone of `template/default/`)
- Modify: `template/stie_pemuda/tinfo.inc.php`

**Interfaces:**
- Consumes: SLiMS 9 template metadata format in `tinfo.inc.php`
- Produces: Independent theme selectable in SLiMS Admin (*System > Theme*)

- [ ] **Step 1: Copy `template/default` directory to `template/stie_pemuda`**

Run in PowerShell:
```powershell
Copy-Item -Path "template\default" -Destination "template\stie_pemuda" -Recurse
```

- [ ] **Step 2: Update theme metadata in `template/stie_pemuda/tinfo.inc.php`**

Update `template/stie_pemuda/tinfo.inc.php`:
```php
<?php
$sysconf['template']['theme_name'] = 'STIE Pemuda Theme';
$sysconf['template']['theme_version'] = '1.0.0';
$sysconf['template']['theme_description'] = 'Custom OPAC theme for Perpustakaan STIE Pemuda';
$sysconf['template']['theme_author'] = 'IT STIE Pemuda';
$sysconf['template']['theme_url'] = 'https://library.stiepemuda.ac.id';
```

- [ ] **Step 3: Verify git tracks the new template files**

Run:
```powershell
git status -s
```
Expected: `?? template/stie_pemuda/` is visible and tracked by Git (thanks to the `.gitignore` whitelist).

- [ ] **Step 4: Commit custom template baseline**

Run:
```powershell
git add template/stie_pemuda/
git commit -m "feat(theme): scaffold stie_pemuda custom OPAC theme"
```

---

### Task 4: Implement GitHub Actions Deployment Workflow

**Files:**
- Create: `.github/workflows/deploy.yml`

**Interfaces:**
- Consumes: GitHub Secrets (`VPS_HOST`, `VPS_PORT`, `VPS_USERNAME`, `VPS_SSH_KEY`, `VPS_PROJECT_DIR`)
- Produces: Automated deployment pipeline running on push to `production` branch

- [ ] **Step 1: Create `.github/workflows/deploy.yml`**

Write `.github/workflows/deploy.yml`:
```yaml
name: Deploy to Production VPS

on:
  push:
    branches:
      - production

jobs:
  deploy:
    name: Deploy to library.stiepemuda.ac.id
    runs-on: ubuntu-latest

    steps:
      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.VPS_HOST }}
          port: ${{ secrets.VPS_PORT }}
          username: ${{ secrets.VPS_USERNAME }}
          key: ${{ secrets.VPS_SSH_KEY }}
          script_stop: true
          script: |
            echo "==> Deploying to SLiMS Live Directory..."
            cd ${{ secrets.VPS_PROJECT_DIR }}
            git config core.filemode false
            git fetch origin production
            git reset --hard origin/production
            rm -rf install/
            echo "==> Restarting SLiMS Web Container..."
            docker restart library-web
            echo "==> Deployment completed successfully!"
```

- [ ] **Step 2: Validate YAML syntax**

Verify file exists and content is properly indented.

- [ ] **Step 3: Commit workflow file**

Run:
```powershell
git add .github/workflows/deploy.yml
git commit -m "ci: add GitHub Actions workflow for production VPS deployment"
```

---

### Task 5: Merge Setup to `production` & Create Deployment Guide

**Files:**
- Create: `docs/deployment/github-actions-setup.md`

**Interfaces:**
- Consumes: Configuration parameters established in Spec
- Produces: Documentation for configuring repository secrets on GitHub and running first deployment

- [ ] **Step 1: Write `docs/deployment/github-actions-setup.md`**

Write guide containing:
- List of 5 GitHub Secrets to configure on GitHub repository (`VPS_HOST`, `VPS_PORT`, `VPS_USERNAME`, `VPS_SSH_KEY`, `VPS_PROJECT_DIR`)
- Instructions on obtaining/generating SSH key for `itadmin`
- Step-by-step verification instructions

- [ ] **Step 2: Commit documentation**

Run:
```powershell
git add docs/deployment/github-actions-setup.md
git commit -m "docs: add GitHub Actions setup and deployment guide"
```

- [ ] **Step 3: Merge `custom-pemuda` changes into `production` branch**

Run:
```powershell
git checkout production
git merge custom-pemuda --no-edit
git checkout custom-pemuda
```

- [ ] **Step 4: Verify branch sync**

Run:
```powershell
git log -n 3 --oneline
```
Ensure commit history contains the template, gitignore, workflow, and doc commits.
