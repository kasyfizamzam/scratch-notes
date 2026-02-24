# Panduan Workflow Git & Pull Request

Dokumen ini menjelaskan standar workflow Git yang digunakan dalam pengembangan proyek **Banjarbakula Presensi**. Kita menggunakan strategi branching berbasis **Feature Branch Workflow** dengan Pull Request (PR) untuk menjaga kualitas kode.

## 1. Struktur Branch

| Branch | Deskripsi | Aturan |
| :--- | :--- | :--- |
| **`main`** | Kode produksi (siap deploy ke server). | **Protected**. Tidak boleh ada commit langsung. Hanya menerima merge dari `development`. |
| **`development`** | Branch utama pengembangan. | **Protected**. Tempat integrasi fitur baru. Kode di sini harus stabil (dapat dibuild/dijalankan). |
| **`feature/*`** | Branch fitur spesifik. | Dibuat dari `development`. Dimerge kembali ke `development` via PR. |
| **`fix/*`** | Branch perbaikan bug. | Sama seperti feature branch. |
| **`hotfix/*`** | Bug critical di production. | Dibuat dari `main`, dimerge ke `main` DAN `development`. |

---

## 2. Alur Pengembangan (Local -> Development)

### Langkah 1: Persiapan (Sync)
Sebelum memulai pekerjaan baru, pastikan local `development` Anda up-to-date.

```bash
git checkout development
git pull origin development
```

### Langkah 2: Buat Branch Baru
Gunakan format nama: `tipe/nama-deskriptif`.
Tipe: `feat`, `fix`, `docs`, `refactor`.

```bash
# Contoh: Fitur baru untuk login
git checkout -b feat/login-page
```

### Langkah 3: Coding & Commit
Lakukan perubahan kode. Gunakan **Conventional Commits** untuk pesan commit.

```bash
git add .
git commit -m "feat: implementasi desain halaman login"
```

> **Format Commit:**
> `tipe: deskripsi singkat (huruf kecil)`
>
> Contoh:
> - `feat: tambah tombol export pdf`
> - `fix: perbaiki error saat upload foto`
> - `docs: update panduan instalasi`

### Langkah 4: Push ke Remote
Push branch fitur Anda ke GitHub.

```bash
git push -u origin feat/login-page
```

---

## 3. Pull Request (Review & Merge)

Setelah kode dipush, langkah selanjutnya adalah menggabungkannya ke `development`.

1.  Buka repository di GitHub.
2.  Anda akan melihat notifikasi "Compare & pull request". Klik tombol tersebut.
3.  **Base:** `development` | **Compare:** `feat/login-page`.
4.  Isi judul dan deskripsi PR dengan jelas.
5.  Minta review dari rekan tim (Assignees/Reviewers).
6.  Jika ada revisi, perbaiki di local, commit, dan push lagi (PR akan otomatis terupdate).
7.  Jika sudah **Approved** dan CI/CD lolos, klik **Merge Pull Request** (Squash and merge disarankan agar history rapi).

---

## 4. Deployment (Development -> Main/Server)

Proses ini dilakukan ketika fitur-fitur di `development` sudah diuji dan siap dirilis ke server production.

### Langkah 1: Persiapan Rilis (Versioning)
Sebelum membuat PR ke main, lakukan hal berikut di branch `development` (atau branch `release/*` jika ada):

1.  **Update Versi Aplikasi:**
    Buka `package.json` dan naikkan versi (semver), contoh: `0.5.1` -> `0.5.2`.
    ```json
    "version": "0.5.2",
    ```

2.  **Buat Changelog:**
    Buat file baru di `docs/changelog/v0.5.2.md`.
    Pastikan formatnya memiliki bagian `**Ringkasan:**` agar muncul di menu "Tentang Aplikasi".
    ```markdown
    # Changelog v0.5.2
    
    ## 🚀 Fitur Baru
    **Ringkasan:**
    Menambahkan fitur X dan memperbaiki bug Y.
    
    ### Detail
    - Item 1
    ```

3.  **Commit Perubahan:**
    ```bash
    git add package.json docs/changelog/v0.5.2.md
    git commit -m "chore: bump version to 0.5.2"
    git push origin development
    ```

### Langkah 2: Buat Pull Request Rilis
1.  Buka GitHub.
2.  Buat Pull Request baru.
3.  **Base:** `main` | **Compare:** `development`.
4.  Judul PR: `chore: release v1.x.x` atau `Release [Tanggal]`.
5.  Review perubahan untuk memastikan aman.

### Langkah 3: Merge ke Main
Setelah dikonfirmasi aman, merge PR tersebut ke `main`.

### Langkah 4: Deploy di Server
Masuk ke server (SSH) dan tarik perubahan terbaru.

```bash
# Di server production
cd /path/to/project
git checkout main
git pull origin main

# Install ulang dependency (jika ada perubahan)
pnpm install

# Jalankan migrasi database (jika ada perubahan schema)
pnpm migrate

# Build ulang aplikasi
pnpm build

# Restart service (contoh menggunakan PM2)
pm2 restart banjarbakula-presensi
```

---

## 5. Menghapus Branch (Cleanup)

Setelah branch fitur dimerge, hapus branch tersebut agar repository tetap bersih.

**Di GitHub:** Biasanya ada tombol "Delete branch" setelah merge.

**Di Local:**
```bash
git checkout development
git pull origin development
git branch -d feat/login-page
```
