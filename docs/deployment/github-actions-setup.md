# Panduan Konfigurasi GitHub Actions & Deployment VPS (STIE Pemuda)

Dokumen ini menjelaskan langkah-langkah untuk menghubungkan repository GitHub (`marbootmasjid/slims9_bulian`) dengan VPS `library.stiepemuda.ac.id` sehingga proses deploy berjalan otomatis saat kode di-push ke branch `production`.

---

## 1. Daftar GitHub Secrets yang Diperlukan

Buka repository Anda di GitHub, lalu navigasikan ke:
**Settings** > **Secrets and variables** > **Actions** > Klik tombol **New repository secret**.

Tambahkan 5 secret berikut:

| Nama Secret | Nilai / Value | Keterangan |
|---|---|---|
| `VPS_HOST` | `library.stiepemuda.ac.id` (atau alamat IP public VPS Anda) | Host VPS target |
| `VPS_PORT` | `22` | Port SSH VPS (sesuaikan jika custom port) |
| `VPS_USERNAME` | `itadmin` | User yang menjalankan Docker di VPS |
| `VPS_SSH_KEY` | *(Isi Private Key SSH user `itadmin`)* | Kunci otentikasi SSH tanpa password |
| `VPS_PROJECT_DIR` | `/home/itadmin/apps/library-pemuda/data/files` | Lokasi live webroot SLiMS di VPS |

---

## 2. Cara Menyiapkan SSH Key untuk `VPS_SSH_KEY` (Jika Belum Punya)

Jika user `itadmin` di VPS belum memiliki pasangan SSH key khusus GitHub Actions:

1. **Login ke VPS** sebagai `itadmin`:
   ```bash
   ssh itadmin@library.stiepemuda.ac.id
   ```

2. **Generate SSH Key baru** (tekan Enter untuk semua prompt tanpa passphrase):
   ```bash
   ssh-keygen -t ed25519 -C "github-actions-deploy" -f ~/.ssh/id_github_actions
   ```

3. **Daftarkan Public Key ke authorized_keys**:
   ```bash
   cat ~/.ssh/id_github_actions.pub >> ~/.ssh/authorized_keys
   chmod 600 ~/.ssh/authorized_keys
   ```

4. **Tampilkan Private Key**:
   ```bash
   cat ~/.ssh/id_github_actions
   ```
   *Salin seluruh teks yang muncul (termasuk baris `-----BEGIN OPENSSH PRIVATE KEY-----` hingga `-----END OPENSSH PRIVATE KEY-----`) dan tempelkan ke secret `VPS_SSH_KEY` di GitHub.*

---

## 3. Persiapan Satu Kali di VPS (One-Time Setup)

Agar Git di VPS mengenali branch `production` dan tidak terkendala permission file Linux:

Jalankan perintah ini sekali di VPS:
```bash
cd ~/apps/library-pemuda/data/files
git config core.filemode false
git fetch origin
git checkout -B production origin/production
```

---

## 4. Alur Kerja Harian (Development to Production)

1. **Koding & Kustomisasi di Lokal**:
   Gunakan branch kerja `custom-pemuda`:
   ```bash
   git checkout custom-pemuda
   # Lakukan kustomisasi tema di template/stie_pemuda/
   git add .
   git commit -m "feat(theme): kustomisasi header dan logo STIE Pemuda"
   ```

2. **Rilis ke Produksi**:
   Merge ke branch `production` lalu push ke GitHub:
   ```bash
   git checkout production
   git merge custom-pemuda --no-edit
   git push origin production
   ```

3. **Otomasi Berjalan**:
   * GitHub Actions akan otomatis terpicu.
   * Menghubungkan ke VPS via SSH.
   * Melakukan `git reset --hard origin/production` di folder live.
   * Menghapus folder `install/` untuk keamanan.
   * Merestart kontainer `library-web` dalam ~2 detik.
   * Perubahan langsung aktif di https://library.stiepemuda.ac.id!

---

## 5. Mengaktifkan Tema di SLiMS

Setelah dideploy ke VPS:
1. Login ke admin SLiMS (`https://library.stiepemuda.ac.id/index.php?p=login`).
2. Masuk ke menu **System** > **Theme**.
3. Pada tab *Public Template*, pilih dan aktifkan **STIE Pemuda Theme**.
