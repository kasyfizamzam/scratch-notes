# PRD: Migrasi Arsitektur Polylith Monorepo

> **Untuk Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

## 1. Ringkasan Eksekutif

**Judul:** Migrasi ke Arsitektur Polylith Monorepo
**Status:** Draft
**Penulis:** Antigravity (Assistant)
**Tanggal:** 13-02-2026

### Tujuan
Melakukan migrasi aplikasi Next.js monolithic saat ini (`banjarbakula-presensi`) menjadi struktur **Polylith Monorepo** menggunakan Turborepo dan pnpm workspaces. Transisi ini bertujuan untuk meningkatkan modularitas kode, performa build, dan pengalaman pengembangan (DX) dengan menegakkan batasan yang ketat antara logika aplikasi, komponen UI, layer database, dan konfigurasi.

### Metrik Keberhasilan
- **Waktu Build:** Build inkremental menggunakan cache Turborepo mengurangi waktu build lokal hingga >40%.
- **Modularitas:** Komponen UI dan skema Database diekstraksi menjadi paket independen yang dapat diberi versi.
- **Pengalaman Pengembang:** Fitur baru dapat ditambahkan ke `apps/web` yang terpadu tanpa circular dependencies, memanfaatkan paket bersama.
- **Zero Regression:** Semua fitur yang ada (Auth, Absensi, Laporan) berfungsi identik setelah migrasi.

---

## 2. Pernyataan Masalah

Aplikasi saat ini adalah **Monolith**, di mana:
1.  **Tight Coupling:** Komponen UI, skema database, dan logika bisnis tercampur dalam satu direktori `src`.
2.  **Slow Feedback Loop:** Setiap perubahan berpotensi memicu rebuild/re-lint penuh dari seluruh basis kode.
3.  **Low Reusability:** Berbagi komponen atau logika (misalnya, skema Zod) dengan layanan masa depan (seperti aplikasi mobile terpisah atau service worker) sulit dilakukan tanpa menyalin kode.

## 3. Arsitektur yang Diusulkan: Polylith

Kita akan mengadopsi arsitektur **Polylith**, yang memisahkan "building blocks" (packages) dari "assembly lines" (apps).

### Tinjauan Struktur

```text
.
├── apps/
│   └── web/                # Aplikasi Next.js yang ada (Consumer)
├── packages/
│   ├── ui/                 # Komponen React + Tailwind Bersama (Provider)
│   ├── database/           # Skema Drizzle ORM, klien, migrasi (Provider)
│   ├── config/             # Tooling Bersama (ESLint, Prettier, TSConfig) (Provider)
│   └── schema/             # Skema validasi Zod Bersama (Provider)
├── turbo.json              # Konfigurasi pipeline build
├── pnpm-workspace.yaml     # Definisi Workspace
└── package.json            # Konfigurasi Root
```

### Keputusan Kunci
1.  **Tooling:** **Turborepo** untuk orkestrasi tugas dan caching. **pnpm** untuk manajemen dependensi yang efisien.
2.  **Version Control:** Repositori Git tunggal (Monorepo).
3.  **Deployment:** `apps/web` tetap menjadi unit deployment utama (Vercel/Docker). Paket bersama di-transpile pada waktu build.

---

## 4. Kebutuhan Fungsional

### 4.1. Konfigurasi Workspace
- **FR-01:** Root harus dikonfigurasi sebagai pnpm workspace.
- **FR-02:** `turbo.json` harus mendefinisikan pipeline untuk `build`, `dev`, `lint`, dan `type-check`.
- **FR-03:** Dependensi umum untuk semua aplikasi (misalnya, `prettier`) harus dikelola secara terpusat atau melalui paket `config`.

### 4.2. Abstraksi Database (`packages/database`)
- **FR-04:** Semua definisi Drizzle ORM (`schema.ts`) harus dipindahkan ke paket ini.
- **FR-05:** Paket harus mengekspor instance klien `db` singleton.
- **FR-06:** Migrasi database (`drizzle-kit`) harus dijalankan dari direktori ini, bukan dari aplikasi.

### 4.3. Library Komponen UI (`packages/ui`)
- **FR-07:** Elemen UI dasar (Tombol, Input, Kartu - kemungkinan wrapper Shadcn/Radix) harus dipindahkan ke sini.
- **FR-08:** Paket harus menyertakan preset konten `tailwind.config.ts` sendiri untuk memastikan styling konsisten saat digunakan.

### 4.4. Refactoring Aplikasi (`apps/web`)
- **FR-09:** Aplikasi Next.js harus mengimpor logika database dari `@banjarbakula/database`.
- **FR-10:** Aplikasi Next.js harus mengimpor komponen UI dari `@banjarbakula/ui`.
- **FR-11:** Logika bisnis spesifik proyek (misalnya, Server Actions untuk aksi, konfigurasi Auth.js) tetap berada di `apps/web`.

---

## 5. Strategi Migrasi (Bertahap)

### Fase 1: Inisialisasi & Lift-and-Shift
1.  Inisialisasi root monorepo.
2.  Pindahkan seluruh proyek saat ini ke dalam `apps/web`.
3.  Perbaiki path di `apps/web` untuk memastikan build berhasil di lokasi baru.

### Fase 2: Ekstraksi - Config & Database
1.  Ekstrak `tsconfig.json`, `biome.json` ke `packages/config`.
2.  Ekstrak `src/db` ke `packages/database`.
3.  Perbarui `apps/web` untuk menggunakan paket database baru.

### Fase 3: Ekstraksi - Layer UI
1.  Identifikasi komponen atomik di `apps/web/src/components/ui`.
2.  Pindahkan mereka ke `packages/ui`.
3.  Ekspor secara efektif dan perbarui impor di `apps/web`.

---

## 6. Risiko & Mitigasi

| Risiko | Dampak | Mitigasi |
| :--- | :--- | :--- |
| **Circular Dependencies** | Tinggi | Penegakan ketat graf dependensi: Apps bergantung pada Packages. Packages TIDAK bergantung pada Apps. |
| **Broken Imports** | Sedang | Gunakan regex find/replace di VS Code untuk memperbarui impor secara massal. Verifikasi dengan `tsc --noEmit`. |
| **Hilangnya Style Tailwind** | Sedang | Pastikan `transpilePackages` diatur dalam konfigurasi Next.js dan path `content` dalam konfigurasi Tailwind mencakup `../../packages/ui`. |
| **Downtime Migrasi** | Rendah | Lakukan migrasi di branch fitur. Verifikasi secara lokal sebelum menggabungkan ke `development`. |

## 7. Perubahan Alur Kerja Pengembangan (DX)

- **Perintah Dev Baru:** `pnpm dev` (menjalankan `turbo dev`, memulai semua aplikasi/paket secara paralel).
- **Perintah Add Baru:** `pnpm add <pkg> --filter <workspace>` (misalnya, `pnpm add lodash --filter @banjarbakula/web`).
- **Perubahan Database:** Jalankan `pnpm db:push` di dalam `packages/database`.
