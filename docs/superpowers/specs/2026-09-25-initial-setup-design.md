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

### 4.1 Production VPS Architecture Details
* **Host**: `vps-pemuda` (`library.stiepemuda.ac.id`)
* **User**: `itadmin`
* **Docker Compose Directory**: `/home/itadmin/apps/library-pemuda`
* **SLiMS Live Repository Directory**: `/home/itadmin/apps/library-pemuda/data/files`
* **Web Container**: `library-web` (port `127.0.0.1:8082`, live volume `./data/files/:/var/www/html`)
* **Database Container**: `library-db` (MariaDB 10.6, persistent in `./data/db`)
* **Remote Git**: `origin` -> `git@github.com:marbootmasjid/slims9_bulian.git`

### 4.2 Workflow Definition (`.github/workflows/deploy.yml`)
* **Trigger**: `push` to branch `production`.
* **Runner**: `ubuntu-latest`.
* **Action**: `appleboy/ssh-action` connecting to `vps-pemuda`.
* **Deployment Execution (Near-Zero Downtime, ~3 seconds)**:
  ```bash
  cd /home/itadmin/apps/library-pemuda/data/files
  git config core.filemode false
  git fetch origin production
  git reset --hard origin/production
  rm -rf install/
  docker restart library-web
  ```
  *(Karena folder `./data/files/` ter-mount live ke kontainer `library-web`, pembaruan kode langsung aktif seketika setelah `git reset --hard` dan restart ringan kontainer tanpa perlu rebuild image lama).*

### 4.3 Required GitHub Secrets
1. `VPS_HOST`: IP atau domain VPS (`library.stiepemuda.ac.id`).
2. `VPS_PORT`: SSH port (default `22`).
3. `VPS_USERNAME`: `itadmin`
4. `VPS_SSH_KEY`: SSH Private Key milik user `itadmin`.
5. `VPS_PROJECT_DIR`: `/home/itadmin/apps/library-pemuda/data/files`

---

## 5. Persistence & Safety Verification

### 5.1 Volume Integrity
* Database MariaDB persisten di host path: `/home/itadmin/apps/library-pemuda/data/db`.
* Webroot & asset persisten di host path: `/home/itadmin/apps/library-pemuda/data/files`.
* Deploy via `git reset --hard origin/production` tidak akan menghapus file yang di-ignore (`config/database.php`, `config/env.php`, folder `files/*`, `images/persons/*`, `images/docs/*`, `repository/*`).

### 5.2 Rollback Plan
Jika terjadi kesalahan pada rilis `production`:
1. Lakukan `git revert <commit-hash>` pada branch `production`.
2. Push ke GitHub (`git push origin production`).
3. GitHub Actions otomatis mengembalikan kode stabil di VPS dalam 3 detik.

---

## 6. Implementation Checklist
1. **Branch Setup**: Buat dan switch ke branch `custom-pemuda` dan branch `production`.
2. **Gitignore Calibration**: Update `.gitignore` agar mengizinkan tracking `!template/stie_pemuda/` dan `!plugins/stie_pemuda_*/`.
3. **Template Scaffold**: Salin `template/default` menjadi `template/stie_pemuda` dengan metadata tema STIE Pemuda.
4. **Revert Core Edits**: Bersihkan hardcode di `plugins/label_barcode/index.php`.
5. **Workflow Creation**: Buat skrip `.github/workflows/deploy.yml` sesuai arsitektur VPS.
6. **Documentation**: Panduan langkah penambahan SSH Key ke GitHub Secrets dan testing deploy pertama kali.
