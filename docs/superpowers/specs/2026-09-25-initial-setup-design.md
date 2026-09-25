# Design Spec: SLiMS 9 Customization Initial Setup & CI/CD Pipeline (STIE Pemuda)

## 1. Background & Goals
* **Project**: Senayan Library Management System (SLiMS 9 Bulian D Ace `v9.8.0`), forked from `slims/slims9_bulian` into `marbootmasjid/slims9_bulian`.
* **Current Production State**: The application is already deployed and live on VPS serving `library.stiepemuda.ac.id` using Docker Compose.
* **Goal**: Establish a robust, maintainable foundation for customizing SLiMS 9 for STIE Pemuda that:
  1. Keeps customizations cleanly separated from SLiMS 9 core to allow seamless upstream security/bugfix sync.
  2. Implements an automated CI/CD pipeline via GitHub Actions to deploy updates to the VPS on push.
  3. Ensures zero data loss and persistent storage preservation across Docker container rebuilds.

---

## 2. Architecture & Git Branching Strategy

```
Upstream (slims/slims9_bulian)
       │ (sync patches)
       ▼
   master (clean mirror of upstream)
       │
       ▼ (merge tested customizations)
   production ──[git push]──► GitHub Actions ──[SSH]──► VPS library.stiepemuda.ac.id
       ▲                                               (docker compose up -d --build slims)
       │ (development)
   custom-pemuda (local feature / theme branch)
```

### 2.1 Branch Responsibilities
* `master`: Tracks upstream `slims/slims9_bulian` strictly without custom modifications.
* `production`: Production-ready branch for STIE Pemuda. Protected branch that triggers the deployment pipeline.
* `custom-pemuda`: Local working branch for developing themes, plugins, and configurations.

### 2.2 Workspace & Gitignore Updates
Modify `.gitignore` to track STIE Pemuda customized components while keeping core files unpolluted:
* Whitelist custom templates: `!template/stie_pemuda/`
* Whitelist custom plugins: `!plugins/stie_pemuda_*/`
* Ensure local environment and credentials remain ignored: `config/database.php`, `config/env.php`, `.env`.

---

## 3. Customization Boundaries (Zero Core Hacks)

### 3.1 Custom OPAC Template
* Source: Clone baseline from `template/default/` into `template/stie_pemuda/`.
* Identification: Set `tinfo.inc.php` metadata (Theme Name: "STIE Pemuda Bulian Theme").
* Modifications: All branding, header, footer, color palettes, and institutional logos are applied exclusively within `template/stie_pemuda/`.

### 3.2 Label & Barcode Clean Revert
* Revert hardcoded strings in `plugins/label_barcode/index.php`.
* Replace with dynamic configuration or dedicated STIE Pemuda print plugin so core SLiMS updates will not create merge conflicts.

---

## 4. CI/CD Deployment Pipeline (GitHub Actions)

### 4.1 Workflow Definition (`.github/workflows/deploy.yml`)
* **Trigger**: `push` to branch `production`.
* **Runner**: `ubuntu-latest`.
* **Action**: `appleboy/ssh-action` connecting to the production VPS.
* **Deployment Execution**:
  ```bash
  cd ${{ secrets.VPS_PROJECT_DIR }}
  git fetch origin production
  git reset --hard origin/production
  docker compose up -d --build slims
  ```

### 4.2 Required GitHub Secrets
1. `VPS_HOST`: IP or domain of the VPS (`library.stiepemuda.ac.id`).
2. `VPS_PORT`: SSH port (default: `22`).
3. `VPS_USERNAME`: SSH username with permissions to manage Docker.
4. `VPS_SSH_KEY`: SSH private key.
5. `VPS_PROJECT_DIR`: Absolute path to the repository directory on the VPS.

---

## 5. Persistence & Safety Verification

### 5.1 Volume Integrity
Docker volumes defined in `docker-compose.yml` safeguard user data:
* `dbdata`: MariaDB data directory (`/var/lib/mysql`).
* `appfiles`: Uploaded files and attachments (`/var/www/html/files`).
* `appimages`: Uploaded book covers and member photos (`/var/www/html/images`).
* `apprepo`: Repository documents (`/var/www/html/repository`).
Rebuilding the `slims` container does not touch or reset any of these volumes.

### 5.2 Rollback Plan
If any breaking bug is introduced in `production`:
1. `git revert <commit-hash>` locally.
2. `git push origin production`.
3. GitHub Actions automatically rebuilds the container with the stable code.

---

## 6. Implementation Checklist
1. **Branch Setup**: Create and checkout `custom-pemuda` and `production` branches.
2. **Gitignore Calibration**: Update `.gitignore` for `template/stie_pemuda/` and custom plugins.
3. **Template Scaffold**: Duplicate `template/default` into `template/stie_pemuda` and configure metadata.
4. **Revert Core Edits**: Restore `plugins/label_barcode/index.php` to clean state.
5. **Workflow Creation**: Implement `.github/workflows/deploy.yml`.
6. **Documentation**: Provide clear guide for adding the 5 GitHub Secrets and setting up the VPS git remote.
