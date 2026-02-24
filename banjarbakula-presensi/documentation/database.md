# Dokumentasi Skema Database

Dokumen ini menjelaskan struktur database PostgreSQL yang digunakan dalam aplikasi Banjarbakula Presensi secara mendetail.

---

## Daftar Isi

1. [Ikhtisar](#1-ikhtisar)
2. [Diagram Relasi](#2-diagram-relasi)
3. [Tabel Master Data](#3-tabel-master-data)
4. [Tabel Pengguna](#4-tabel-pengguna)
5. [Tabel Kehadiran](#5-tabel-kehadiran)
6. [Tabel Jadwal & Shift](#6-tabel-jadwal--shift)
7. [Tabel Cuti & Izin](#7-tabel-cuti--izin)
8. [Tabel Analitik](#8-tabel-analitik)
9. [Tabel Kebijakan](#9-tabel-kebijakan)
10. [Tabel Sistem](#10-tabel-sistem)

---

## 1. Ikhtisar

### Teknologi Database

| Komponen | Teknologi |
|----------|-----------|
| RDBMS | PostgreSQL 16 |
| ORM | Drizzle ORM |
| Migration Tool | Drizzle Kit |

### Konvensi Penamaan

| Elemen | Konvensi | Contoh |
|--------|----------|--------|
| Tabel | snake_case, plural | `attendance_records` |
| Kolom | snake_case | `check_in_at` |
| Primary Key | `id` (text UUID) | `id TEXT PRIMARY KEY` |
| Foreign Key | `<table>_id` | `user_id`, `shift_config_id` |
| Timestamp | `<action>_at` | `created_at`, `updated_at` |
| Boolean | `is_<condition>` | `is_active`, `is_working_day` |

### Tipe Data Umum

| Penggunaan | Tipe PostgreSQL |
|------------|-----------------|
| ID | `TEXT` (UUID string) |
| Tanggal | `DATE` |
| Waktu | `TIMESTAMP` |
| Jam (string) | `TEXT` (format HH:mm) |
| Koordinat | `DOUBLE PRECISION` |
| JSON/Object | `JSONB` |

---

## 2. Diagram Relasi

```
┌─────────────────┐
│    divisions    │◀────────────────────┐
│ (Bidang/Divisi) │                     │
└────────┬────────┘                     │
         │                              │
         │ 1:N                          │
         ▼                              │
┌─────────────────┐     1:N      ┌──────┴────────┐
│    positions    │◀─────────────│     users     │
│   (Jabatan)     │              │   (Pengguna)  │
└─────────────────┘              └───────┬───────┘
                                         │
                    ┌────────────────────┼────────────────────┐
                    │                    │                    │
                    ▼ 1:N                ▼ 1:N                ▼ 1:N
          ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
          │     shifts      │  │ attendance_     │  │ leave_requests  │
          │   (Jadwal)      │  │ records         │  │   (Cuti)        │
          └────────┬────────┘  │ (Kehadiran)     │  └────────┬────────┘
                   │           └─────────────────┘           │
         ┌─────────┼─────────┐                               │
         │         │         │                     ┌─────────▼─────────┐
         ▼         ▼         ▼                     │ leave_balances    │
  ┌───────────┐ ┌───────────┐ ┌───────────────┐   │ (Saldo Cuti)      │
  │ work_     │ │ shift_    │ │ work_         │   └───────────────────┘
  │ schedules │ │ configs   │ │ locations     │
  │(Container)│ │ (Shift)   │ │ (Lokasi)      │
  └───────────┘ └───────────┘ └───────────────┘
```

---

## 3. Tabel Master Data

### 3.1 divisions (Bidang/Divisi)

Menyimpan data master bidang atau divisi organisasi.

| Kolom | Tipe | Nullable | Deskripsi |
|-------|------|----------|-----------|
| `id` | TEXT | NOT NULL | Primary key (UUID) |
| `name` | TEXT | NOT NULL | Nama divisi (unique) |
| `is_active` | BOOLEAN | NOT NULL | Status aktif (default: true) |
| `created_at` | TIMESTAMP | NOT NULL | Waktu pembuatan |

**Constraint:**
- `divisions_name_unique` - Nama divisi harus unik

---

### 3.2 positions (Jabatan)

Menyimpan data master jabatan pegawai.

| Kolom | Tipe | Nullable | Deskripsi |
|-------|------|----------|-----------|
| `id` | TEXT | NOT NULL | Primary key (UUID) |
| `name` | TEXT | NOT NULL | Nama jabatan (unique) |
| `is_active` | BOOLEAN | NOT NULL | Status aktif (default: true) |
| `created_at` | TIMESTAMP | NOT NULL | Waktu pembuatan |

**Constraint:**
- `positions_name_unique` - Nama jabatan harus unik

---

## 4. Tabel Pengguna

### 4.1 users (Pengguna)

Menyimpan data pengguna sistem (admin dan pegawai).

| Kolom | Tipe | Nullable | Deskripsi |
|-------|------|----------|-----------|
| `id` | TEXT | NOT NULL | Primary key (UUID) |
| `username` | TEXT | NOT NULL | Username untuk login (unique) |
| `full_name` | TEXT | NOT NULL | Nama lengkap |
| `password_hash` | TEXT | NOT NULL | Hash password (Argon2) |
| `role` | ENUM | NOT NULL | Role: 'admin' atau 'employee' |
| `division_id` | TEXT | NULL | FK ke divisions |
| `position_id` | TEXT | NULL | FK ke positions |
| `is_active` | BOOLEAN | NOT NULL | Status akun aktif |
| `must_change_password` | BOOLEAN | NOT NULL | Wajib ganti password? |
| `auth_version` | INTEGER | NOT NULL | Versi auth (untuk invalidasi session) |
| `created_at` | TIMESTAMP | NOT NULL | Waktu pembuatan |
| `updated_at` | TIMESTAMP | NOT NULL | Waktu update terakhir |

**Enum role:**
```sql
CREATE TYPE role AS ENUM ('admin', 'employee');
```

**Constraint:**
- `users_username_unique` - Username harus unik

---

## 5. Tabel Kehadiran

### 5.1 attendance_records (Record Kehadiran)

Menyimpan data kehadiran harian setiap pegawai.

| Kolom | Tipe | Nullable | Deskripsi |
|-------|------|----------|-----------|
| `id` | TEXT | NOT NULL | Primary key (UUID) |
| `user_id` | TEXT | NOT NULL | FK ke users |
| `date` | DATE | NOT NULL | Tanggal kehadiran |
| `year` | INTEGER | NOT NULL | Tahun (untuk partisi) |
| `check_in_at` | TIMESTAMP | NULL | Waktu check-in |
| `check_out_at` | TIMESTAMP | NULL | Waktu check-out |
| `status` | ENUM | NOT NULL | Status kehadiran |
| `original_status` | ENUM | NULL | Status sebelum override |
| `late_minutes` | INTEGER | NULL | Menit keterlambatan |
| `early_minutes` | INTEGER | NULL | Menit pulang awal |
| `location_label` | TEXT | NULL | Label lokasi check-in |
| `check_in_photo_url` | TEXT | NULL | URL foto check-in |
| `check_out_photo_url` | TEXT | NULL | URL foto check-out |
| `late_check_in` | BOOLEAN | NOT NULL | Flag: terlambat check-in |
| `early_check_out` | BOOLEAN | NOT NULL | Flag: pulang awal |
| `missing_check_out` | BOOLEAN | NOT NULL | Flag: tidak check-out |
| `outside_location` | BOOLEAN | NOT NULL | Flag: di luar lokasi |
| `original_late_check_in` | BOOLEAN | NULL | Flag asli sebelum override |
| `original_early_check_out` | BOOLEAN | NULL | Flag asli sebelum override |
| `original_missing_check_out` | BOOLEAN | NULL | Flag asli sebelum override |
| `original_outside_location` | BOOLEAN | NULL | Flag asli sebelum override |
| `is_mock_location_detected` | BOOLEAN | NULL | Terdeteksi lokasi palsu? |
| `gps_accuracy` | REAL | NULL | Akurasi GPS (meter) |
| `device_info` | JSONB | NULL | Info perangkat |
| `overridden_by_admin` | BOOLEAN | NOT NULL | Di-override admin? |
| `overridden_at` | TIMESTAMP | NULL | Waktu override |
| `overridden_by` | TEXT | NULL | FK ke users (admin) |
| `override_reason` | TEXT | NULL | Alasan override |
| `created_at` | TIMESTAMP | NOT NULL | Waktu pembuatan |
| `updated_at` | TIMESTAMP | NOT NULL | Waktu update |

**Enum attendance_status:**
```sql
CREATE TYPE attendance_status AS ENUM ('present', 'late', 'excused', 'absent');
```

**Constraint & Index:**
- `attendance_user_date_unique` - Satu record per user per tanggal
- `attendance_year_idx` - Index pada kolom year
- `attendance_user_year_idx` - Index pada (user_id, year)
- `attendance_user_date_idx` - Index pada (user_id, date)
- `attendance_date_status_idx` - Index pada (date, status)

---

### 5.2 attendance_override_log (Log Override)

Menyimpan riwayat perubahan status kehadiran oleh admin.

| Kolom | Tipe | Nullable | Deskripsi |
|-------|------|----------|-----------|
| `id` | TEXT | NOT NULL | Primary key |
| `attendance_id` | TEXT | NOT NULL | FK ke attendance_records |
| `admin_id` | TEXT | NOT NULL | FK ke users (admin) |
| `previous_status` | TEXT | NOT NULL | Status sebelumnya |
| `new_status` | TEXT | NOT NULL | Status baru |
| `reason` | TEXT | NOT NULL | Alasan perubahan |
| `created_at` | TIMESTAMP | NOT NULL | Waktu perubahan |

---

## 6. Tabel Jadwal & Shift

### 6.1 work_schedules (Container Jadwal)

Menyimpan container jadwal kerja bulanan.

| Kolom | Tipe | Nullable | Deskripsi |
|-------|------|----------|-----------|
| `id` | TEXT | NOT NULL | Primary key |
| `month` | TEXT | NOT NULL | Bulan (format: YYYY-MM) |
| `year` | INTEGER | NOT NULL | Tahun |
| `status` | TEXT | NOT NULL | Status: draft/published |
| `created_at` | TIMESTAMP | NOT NULL | Waktu pembuatan |

---

### 6.2 shift_configs (Konfigurasi Shift)

Menyimpan template konfigurasi shift kerja.

| Kolom | Tipe | Nullable | Deskripsi |
|-------|------|----------|-----------|
| `id` | TEXT | NOT NULL | Primary key |
| `name` | TEXT | NOT NULL | Nama shift (unique) |
| `start_time` | TEXT | NULL | Jam mulai (format: HH:mm) |
| `end_time` | TEXT | NULL | Jam selesai (format: HH:mm) |
| `grace_minutes` | INTEGER | NOT NULL | Toleransi keterlambatan (menit) |
| `crosses_midnight` | BOOLEAN | NOT NULL | Melewati tengah malam? |
| `is_working_day` | BOOLEAN | NOT NULL | Hari kerja atau libur? |
| `color` | TEXT | NOT NULL | Warna untuk UI (hex) |
| `is_active` | BOOLEAN | NOT NULL | Status aktif |
| `created_at` | TIMESTAMP | NOT NULL | Waktu pembuatan |
| `updated_at` | TIMESTAMP | NOT NULL | Waktu update |

**Constraint:**
- `shift_configs_name_unique` - Nama shift harus unik

---

### 6.3 work_locations (Lokasi Kerja)

Menyimpan data lokasi kerja dengan geofence.

| Kolom | Tipe | Nullable | Deskripsi |
|-------|------|----------|-----------|
| `id` | TEXT | NOT NULL | Primary key |
| `name` | TEXT | NOT NULL | Nama lokasi (unique) |
| `latitude` | DOUBLE PRECISION | NOT NULL | Koordinat latitude |
| `longitude` | DOUBLE PRECISION | NOT NULL | Koordinat longitude |
| `radius_meters` | INTEGER | NOT NULL | Radius geofence (meter) |
| `is_active` | BOOLEAN | NOT NULL | Status aktif |
| `created_at` | TIMESTAMP | NOT NULL | Waktu pembuatan |
| `updated_at` | TIMESTAMP | NOT NULL | Waktu update |

---

### 6.4 shifts (Jadwal Harian)

Menyimpan jadwal kerja harian per pegawai.

| Kolom | Tipe | Nullable | Deskripsi |
|-------|------|----------|-----------|
| `id` | TEXT | NOT NULL | Primary key |
| `schedule_id` | TEXT | NOT NULL | FK ke work_schedules |
| `user_id` | TEXT | NOT NULL | FK ke users |
| `date` | DATE | NOT NULL | Tanggal |
| `year` | INTEGER | NOT NULL | Tahun |
| `shift_config_id` | TEXT | NULL | FK ke shift_configs |
| `location_id` | TEXT | NULL | FK ke work_locations |
| `created_at` | TIMESTAMP | NOT NULL | Waktu pembuatan |
| `updated_at` | TIMESTAMP | NOT NULL | Waktu update |

**Constraint & Index:**
- `shifts_user_date_unique` - Satu shift per user per tanggal
- `shifts_year_idx` - Index pada year
- `shifts_user_year_idx` - Index pada (user_id, year)

---

## 7. Tabel Cuti & Izin

### 7.1 leave_requests (Pengajuan Cuti)

Menyimpan pengajuan cuti/izin pegawai.

| Kolom | Tipe | Nullable | Deskripsi |
|-------|------|----------|-----------|
| `id` | TEXT | NOT NULL | Primary key |
| `user_id` | TEXT | NOT NULL | FK ke users |
| `type` | ENUM | NOT NULL | Jenis cuti |
| `start_date` | DATE | NOT NULL | Tanggal mulai |
| `end_date` | DATE | NOT NULL | Tanggal selesai |
| `year` | INTEGER | NOT NULL | Tahun |
| `reason` | TEXT | NULL | Alasan pengajuan |
| `document_url` | TEXT | NULL | URL dokumen pendukung |
| `status` | ENUM | NOT NULL | Status pengajuan |
| `deduction_status` | ENUM | NULL | Status deduksi saldo |
| `deducted_days` | INTEGER | NULL | Jumlah hari yang dideduksi |
| `decided_by` | TEXT | NULL | FK ke users (admin) |
| `decided_at` | TIMESTAMP | NULL | Waktu keputusan |
| `decision_note` | TEXT | NULL | Catatan keputusan |
| `created_at` | TIMESTAMP | NOT NULL | Waktu pembuatan |
| `updated_at` | TIMESTAMP | NOT NULL | Waktu update |

**Enum leave_type:**
```sql
CREATE TYPE leave_type AS ENUM (
  'permission',    -- izin
  'sick',          -- sakit
  'annual_leave',  -- cuti tahunan
  'official_duty'  -- tugas luar
);
```

**Enum leave_status:**
```sql
CREATE TYPE leave_status AS ENUM ('pending', 'approved', 'rejected');
```

**Enum deduction_status:**
```sql
CREATE TYPE deduction_status AS ENUM ('provisional', 'final');
```

---

### 7.2 leave_balances (Saldo Cuti)

Menyimpan saldo cuti tahunan per pegawai.

| Kolom | Tipe | Nullable | Deskripsi |
|-------|------|----------|-----------|
| `id` | TEXT | NOT NULL | Primary key |
| `user_id` | TEXT | NOT NULL | FK ke users |
| `year` | INTEGER | NOT NULL | Tahun |
| `total` | INTEGER | NOT NULL | Total cuti (default: 12) |
| `used` | INTEGER | NOT NULL | Cuti terpakai |
| `created_at` | TIMESTAMP | NOT NULL | Waktu pembuatan |
| `updated_at` | TIMESTAMP | NOT NULL | Waktu update |

**Constraint:**
- `leave_balance_user_year_unique` - Satu record per user per tahun

---

### 7.3 leave_deduction_details (Detail Deduksi)

Menyimpan detail deduksi per hari untuk cuti.

| Kolom | Tipe | Nullable | Deskripsi |
|-------|------|----------|-----------|
| `id` | TEXT | NOT NULL | Primary key |
| `leave_request_id` | TEXT | NOT NULL | FK ke leave_requests |
| `date` | DATE | NOT NULL | Tanggal |
| `schedule_status` | ENUM | NOT NULL | Status jadwal hari itu |
| `deducted` | BOOLEAN | NOT NULL | Apakah dideduksi? |
| `evaluated_at` | TIMESTAMP | NOT NULL | Waktu evaluasi |

**Enum schedule_status:**
```sql
CREATE TYPE schedule_status AS ENUM ('WORKING', 'OFF', 'NOT_DEFINED');
```

---

## 8. Tabel Analitik

### 8.1 analytics_snapshots (Snapshot Analitik)

Menyimpan hasil agregasi bulanan untuk keperluan analitik.

| Kolom | Tipe | Nullable | Deskripsi |
|-------|------|----------|-----------|
| `id` | TEXT | NOT NULL | Primary key |
| `year` | INTEGER | NOT NULL | Tahun |
| `month` | INTEGER | NOT NULL | Bulan (1-12) |
| `type` | ENUM | NOT NULL | Jenis snapshot |
| `payload` | JSONB | NOT NULL | Data snapshot |
| `generated_at` | TIMESTAMP | NOT NULL | Waktu generate |
| `source_version` | TEXT | NULL | Versi source code |
| `created_at` | TIMESTAMP | NOT NULL | Waktu pembuatan |

**Enum analytics_snapshot_type:**
```sql
CREATE TYPE analytics_snapshot_type AS ENUM (
  'late_summary',
  'absence_recap',
  'anomaly_list',
  'monthly_attendance',
  'employee_performance'
);
```

---

## 9. Tabel Kebijakan

### 9.1 policy_rules (Aturan Kebijakan)

Menyimpan definisi aturan kebijakan disipliner.

| Kolom | Tipe | Nullable | Deskripsi |
|-------|------|----------|-----------|
| `id` | TEXT | NOT NULL | Primary key |
| `code` | TEXT | NOT NULL | Kode aturan (unique) |
| `description` | TEXT | NOT NULL | Deskripsi aturan |
| `threshold` | JSONB | NOT NULL | Threshold trigger |
| `action_hint` | TEXT | NOT NULL | Saran tindakan |
| `is_active` | BOOLEAN | NOT NULL | Status aktif |
| `created_at` | TIMESTAMP | NOT NULL | Waktu pembuatan |
| `updated_at` | TIMESTAMP | NOT NULL | Waktu update |

**Contoh threshold JSONB:**
```json
{
  "late_days": { "gte": 3 },
  "period": "month"
}
```

---

### 9.2 policy_action_proposals (Proposal Tindakan)

Menyimpan proposal tindakan yang dihasilkan sistem.

| Kolom | Tipe | Nullable | Deskripsi |
|-------|------|----------|-----------|
| `id` | TEXT | NOT NULL | Primary key |
| `policy_id` | TEXT | NOT NULL | FK ke policy_rules |
| `user_id` | TEXT | NOT NULL | FK ke users (target) |
| `period` | TEXT | NOT NULL | Periode (YYYY-MM) |
| `year` | INTEGER | NOT NULL | Tahun |
| `evidence` | JSONB | NOT NULL | Data bukti |
| `status` | ENUM | NOT NULL | Status proposal |
| `created_at` | TIMESTAMP | NOT NULL | Waktu pembuatan |
| `updated_at` | TIMESTAMP | NOT NULL | Waktu update |

**Enum policy_proposal_status:**
```sql
CREATE TYPE policy_proposal_status AS ENUM (
  'pending',
  'approved',
  'dismissed',
  'superseded'
);
```

---

### 9.3 policy_action_logs (Log Tindakan)

Menyimpan log eksekusi tindakan oleh admin.

| Kolom | Tipe | Nullable | Deskripsi |
|-------|------|----------|-----------|
| `id` | TEXT | NOT NULL | Primary key |
| `proposal_id` | TEXT | NOT NULL | FK ke proposals |
| `admin_id` | TEXT | NOT NULL | FK ke users (admin) |
| `action_taken` | ENUM | NOT NULL | Jenis tindakan |
| `note` | TEXT | NULL | Catatan tambahan |
| `executed_at` | TIMESTAMP | NOT NULL | Waktu eksekusi |

**Enum policy_action_type:**
```sql
CREATE TYPE policy_action_type AS ENUM (
  'verbal_warning',    -- teguran lisan
  'written_warning',   -- surat peringatan
  'internal_note',     -- catatan internal
  'performance_review', -- evaluasi kinerja
  'other'              -- lainnya
);
```

---

## 10. Tabel Sistem

### 10.1 audit_logs (Log Audit)

Menyimpan log aktivitas sistem untuk keperluan audit.

| Kolom | Tipe | Nullable | Deskripsi |
|-------|------|----------|-----------|
| `id` | TEXT | NOT NULL | Primary key |
| `actor_id` | TEXT | NULL | FK ke users (pelaku) |
| `action` | TEXT | NOT NULL | Jenis aksi |
| `entity_type` | TEXT | NULL | Tipe entitas yang diubah |
| `entity_id` | TEXT | NULL | ID entitas yang diubah |
| `metadata` | JSONB | NULL | Data tambahan |
| `created_at` | TIMESTAMP | NOT NULL | Waktu aksi |

---

### 10.2 system_settings (Pengaturan Sistem)

Menyimpan konfigurasi sistem yang dapat diubah runtime.

| Kolom | Tipe | Nullable | Deskripsi |
|-------|------|----------|-----------|
| `key` | TEXT | NOT NULL | Primary key (nama setting) |
| `value` | TEXT | NOT NULL | Nilai setting |
| `description` | TEXT | NULL | Deskripsi setting |
| `updated_at` | TIMESTAMP | NOT NULL | Waktu update terakhir |
| `updated_by` | TEXT | NULL | FK ke users (admin) |

---

## Lampiran: Perintah Migrasi

### Membuat Migrasi Baru

```bash
# Setelah mengubah src/db/schema.ts
drizzle-kit generate
```

### Menjalankan Migrasi

```bash
pnpm migrate
```

### Melihat Status Migrasi

```bash
drizzle-kit status
```

---

*Dokumen ini diperbarui terakhir: Januari 2026*
