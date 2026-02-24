# Arsitektur Aplikasi

Dokumen ini menjelaskan arsitektur teknis aplikasi Banjarbakula Presensi secara mendalam untuk membantu developer memahami struktur dan alur kerja sistem.

---

## Daftar Isi

1. [Ikhtisar Arsitektur](#1-ikhtisar-arsitektur)
2. [Lapisan Aplikasi](#2-lapisan-aplikasi)
3. [Struktur Direktori](#3-struktur-direktori)
4. [Alur Data](#4-alur-data)
5. [Sistem Autentikasi](#5-sistem-autentikasi)
6. [Sistem Kehadiran](#6-sistem-kehadiran)
7. [Manajemen Jadwal](#7-manajemen-jadwal)
8. [Mesin Anomali & Kebijakan](#8-mesin-anomali--kebijakan)

---

## 1. Ikhtisar Arsitektur

Aplikasi ini dibangun menggunakan arsitektur monolitik modern berbasis Next.js App Router dengan pendekatan Server Components dan Server Actions.

### Teknologi Utama

| Komponen | Teknologi | Versi |
|----------|-----------|-------|
| Framework | Next.js (App Router) | 16.x |
| Runtime | React Server Components | 19.x |
| Database | PostgreSQL | 16.x |
| ORM | Drizzle ORM | 0.45.x |
| Autentikasi | NextAuth.js v5 | 5.0.0-beta |
| Styling | Tailwind CSS | 4.x |
| UI Components | Radix UI + shadcn/ui | Latest |

### Diagram Arsitektur

```
┌──────────────────────────────────────────────────────────────────┐
│                        LAPISAN KLIEN                             │
│  ┌────────────────────┐        ┌────────────────────┐           │
│  │   PWA (Pegawai)    │        │  Dashboard Admin   │           │
│  │   /app/(app)/*     │        │   /app/admin/*     │           │
│  │                    │        │                    │           │
│  │  • Check-in/out    │        │  • Kelola Pegawai  │           │
│  │  • Riwayat         │        │  • Kelola Jadwal   │           │
│  │  • Pengajuan Cuti  │        │  • Rekap Kehadiran │           │
│  └─────────┬──────────┘        └─────────┬──────────┘           │
│            │                              │                      │
│            └──────────────┬───────────────┘                      │
│                           ▼                                      │
├──────────────────────────────────────────────────────────────────┤
│                       LAPISAN SERVER                             │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                 Next.js App Router                         │  │
│  │                                                            │  │
│  │  ┌─────────────────┐  ┌─────────────────┐                 │  │
│  │  │ Server Actions  │  │   API Routes    │                 │  │
│  │  │ /app/actions/*  │  │   /app/api/*    │                 │  │
│  │  └────────┬────────┘  └────────┬────────┘                 │  │
│  │           │                    │                          │  │
│  │           └────────┬───────────┘                          │  │
│  │                    ▼                                      │  │
│  │  ┌────────────────────────────────────────────────────┐  │  │
│  │  │              Lapisan Logika Bisnis                  │  │  │
│  │  │              /src/lib/*                             │  │  │
│  │  │                                                     │  │  │
│  │  │  • attendance-status.ts  (Derivasi Status)         │  │  │
│  │  │  • attendance-population.ts (Populasi Data)        │  │  │
│  │  │  • anomaly-detection.ts  (Deteksi Pola)            │  │  │
│  │  │  • policy-engine.ts      (Mesin Kebijakan)         │  │  │
│  │  │  • leave-deduction.ts    (Kalkulasi Cuti)          │  │  │
│  │  │  • feature-flags.ts      (Toggle Fitur)            │  │  │
│  │  └────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              │                                   │
├──────────────────────────────▼───────────────────────────────────┤
│                        LAPISAN DATA                              │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                     Drizzle ORM                            │  │
│  │                     /src/db/*                              │  │
│  │  • schema.ts      (Definisi Tabel)                        │  │
│  │  • index.ts       (Koneksi Database)                      │  │
│  │  • migrations/*   (File Migrasi SQL)                      │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              ▼                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                    PostgreSQL 16                           │  │
│  └───────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 2. Lapisan Aplikasi

### 2.1 Lapisan Presentasi (Client)

Lapisan ini menangani tampilan antarmuka pengguna dan interaksi.

**Komponen Utama:**
- **Server Components**: Komponen yang dirender di server, ideal untuk fetching data
- **Client Components**: Komponen interaktif dengan state dan event handlers
- **UI Library**: shadcn/ui berbasis Radix UI primitives

**Konvensi:**
```tsx
// Server Component (default)
export default async function Page() {
  const data = await fetchData();
  return <ClientComponent data={data} />;
}

// Client Component
"use client";
export function ClientComponent({ data }) {
  const [state, setState] = useState();
  return <div>...</div>;
}
```

### 2.2 Lapisan Server Actions

Server Actions adalah fungsi asinkron yang berjalan di server dan dipanggil langsung dari komponen klien.

**Lokasi:** `/src/app/actions/`

**Contoh Implementasi:**
```typescript
"use server";

import { db } from "@/db";
import { requireAuth } from "@/lib/auth";

export async function createEmployee(data: EmployeeInput) {
  // 1. Validasi autentikasi
  const session = await requireAuth();
  
  // 2. Validasi role
  if (session.user.role !== "admin") {
    throw new Error("Unauthorized");
  }
  
  // 3. Validasi input
  const validated = employeeSchema.parse(data);
  
  // 4. Eksekusi query database
  await db.insert(users).values({...});
  
  // 5. Kembalikan hasil
  return { success: true };
}
```

### 2.3 Lapisan Logika Bisnis

Lapisan ini berisi logika bisnis murni yang terpisah dari framework.

**Lokasi:** `/src/lib/`

**Modul Utama:**

| File | Fungsi |
|------|--------|
| `attendance-status.ts` | Menentukan status kehadiran berdasarkan jadwal dan record |
| `attendance-population.ts` | Mengambil data populasi kehadiran semua pegawai |
| `anomaly-detection.ts` | Mendeteksi pola anomali (terlambat berulang, alpha) |
| `policy-engine.ts` | Mengevaluasi aturan kebijakan disipliner |
| `leave-deduction.ts` | Menghitung deduksi saldo cuti |
| `feature-flags.ts` | Mengontrol toggle fitur via environment variable |

### 2.4 Lapisan Akses Data

Lapisan ini menangani komunikasi dengan database PostgreSQL menggunakan Drizzle ORM.

**Lokasi:** `/src/db/`

**Komponen:**
- `schema.ts` - Definisi skema tabel dengan tipe TypeScript
- `index.ts` - Instance koneksi database
- `migrations/` - File migrasi SQL yang di-generate

---

## 3. Struktur Direktori

```
src/
├── app/                          # Next.js App Router
│   ├── (admin-auth)/             # Layout khusus auth admin
│   ├── (app)/                    # Halaman PWA pegawai
│   │   ├── attendance/           # Halaman presensi
│   │   ├── history/              # Riwayat kehadiran
│   │   ├── leave/                # Pengajuan cuti
│   │   └── home/                 # Dashboard pegawai
│   ├── (auth)/                   # Halaman login
│   ├── actions/                  # Server Actions
│   │   ├── auth.ts               # Login, logout
│   │   ├── attendance.ts         # Check-in, check-out
│   │   ├── schedule.ts           # Manajemen jadwal
│   │   ├── employee.ts           # CRUD pegawai
│   │   ├── leave.ts              # Manajemen cuti
│   │   └── ...
│   ├── admin/                    # Dashboard admin
│   │   ├── attendance/           # Rekap kehadiran
│   │   ├── employees/            # Daftar pegawai
│   │   ├── schedules/            # Manajemen jadwal
│   │   ├── leaves/               # Approval cuti
│   │   └── config/               # Konfigurasi sistem
│   └── api/                      # API Routes (REST)
│
├── components/                   # Komponen React
│   ├── admin/                    # Komponen khusus admin
│   │   ├── attendance/           # Tabel kehadiran
│   │   ├── schedule/             # Kalender jadwal
│   │   └── tables/               # Tabel data
│   ├── pwa/                      # Komponen PWA
│   │   ├── attendance/           # UI check-in
│   │   └── home/                 # Dashboard pegawai
│   ├── shared/                   # Komponen shared
│   └── ui/                       # shadcn/ui components
│
├── db/                           # Database layer
│   ├── schema.ts                 # Definisi skema Drizzle
│   ├── index.ts                  # Koneksi database
│   └── migrations/               # SQL migrations
│
├── lib/                          # Utilitas & logika bisnis
│   ├── attendance-status.ts      # Engine derivasi status
│   ├── attendance-population.ts  # Fetcher populasi
│   ├── anomaly-detection.ts      # Deteksi anomali
│   ├── policy-engine.ts          # Mesin kebijakan
│   ├── feature-flags.ts          # Toggle fitur
│   ├── utils.ts                  # Helper functions
│   └── date-utils.ts             # Utilitas tanggal/waktu
│
├── hooks/                        # Custom React hooks
└── types/                        # TypeScript type definitions
```

---

## 4. Alur Data

### 4.1 Alur Check-in Pegawai

```
┌─────────┐     ┌───────────────┐     ┌─────────────────┐
│ Pegawai │ ──▶ │   PWA Client  │ ──▶ │ Server Action   │
│         │     │               │     │ attendance.ts   │
└─────────┘     └───────────────┘     └────────┬────────┘
                                               │
                                               ▼
                      ┌────────────────────────────────────┐
                      │         Validasi                   │
                      │ 1. Autentikasi (session valid?)    │
                      │ 2. Jadwal (ada shift hari ini?)    │
                      │ 3. Geofence (dalam radius?)        │
                      │ 4. Mock GPS (lokasi asli?)         │
                      │ 5. Duplikat (belum check-in?)      │
                      └────────────────┬───────────────────┘
                                       │
                             ┌─────────▼─────────┐
                             │ INSERT / UPDATE   │
                             │ attendance_records│
                             └─────────┬─────────┘
                                       │
                             ┌─────────▼─────────┐
                             │  Hitung Status    │
                             │  • PRESENT        │
                             │  • LATE (+menit)  │
                             └─────────┬─────────┘
                                       │
                             ┌─────────▼─────────┐
                             │ Return Response   │
                             │ { success: true } │
                             └───────────────────┘
```

### 4.2 Alur Derivasi Status Kehadiran

Fungsi `deriveAttendanceStatus()` menentukan status kehadiran berdasarkan berbagai input:

```
┌─────────────────────────────────────────────────────────────┐
│                 deriveAttendanceStatus()                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  INPUT:                                                     │
│  • ShiftData (jadwal hari ini)                              │
│  • AttendanceRecord (data check-in/out)                     │
│  • LeaveRequest (data cuti jika ada)                        │
│  • dateStr (tanggal yang dievaluasi)                        │
│                                                             │
│  LOGIKA PRIORITAS:                                          │
│                                                             │
│  1. Cek CUTI DISETUJUI                                      │
│     └─ Jika ada → Return LEAVE/SICK/PERMISSION              │
│                                                             │
│  2. Cek JADWAL                                              │
│     └─ Jika tidak ada jadwal → Return NO_SCHEDULE           │
│                                                             │
│  3. Cek RECORD KEHADIRAN                                    │
│     ├─ Tidak ada record + tanggal lampau → Return ABSENT    │
│     ├─ Tidak ada record + hari ini/depan → NOT_CHECKED_IN   │
│     └─ Ada record → Map ke status:                          │
│         ├─ present → PRESENT                                │
│         ├─ late → LATE                                      │
│         ├─ excused → LEAVE                                  │
│         └─ absent → ABSENT                                  │
│                                                             │
│  4. Cek ADMIN OVERRIDE                                      │
│     └─ Parse prefix override_reason untuk status khusus     │
│                                                             │
│  OUTPUT: DerivedAttendanceResult                            │
│  • status: DerivedAttendanceStatus                          │
│  • shiftName, shiftStartTime, shiftEndTime                  │
│  • scheduledLocationName                                    │
│  • checkInAt, checkOutAt                                    │
│  • lateMinutes, earlyMinutes                                │
│  • locationLabel, gpsAccuracy                               │
│  • overriddenByAdmin, overrideReason                        │
│  • recordId                                                 │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. Sistem Autentikasi

### 5.1 Arsitektur Auth

Sistem menggunakan NextAuth.js v5 dengan strategi Credentials (username + password).

```
┌──────────────┐     ┌───────────────┐     ┌─────────────────┐
│    Login     │ ──▶ │ NextAuth.js   │ ──▶ │ PostgreSQL      │
│    Form      │     │ Credentials   │     │ users table     │
└──────────────┘     └───────────────┘     └─────────────────┘
                            │
                            ▼
              ┌──────────────────────────┐
              │    JWT Session Token     │
              │    (httpOnly cookie)     │
              └──────────────────────────┘
```

### 5.2 File Konfigurasi

| File | Fungsi |
|------|--------|
| `/src/auth.ts` | Konfigurasi utama NextAuth |
| `/src/auth.config.ts` | Opsi autentikasi (providers, callbacks) |
| `/src/lib/admin-auth.ts` | Helper untuk validasi role admin |

### 5.3 Proteksi Route

```typescript
// Untuk Server Components
import { auth } from "@/auth";

export default async function AdminPage() {
  const session = await auth();
  
  if (!session || session.user.role !== "admin") {
    redirect("/forbidden");
  }
  
  return <Dashboard />;
}

// Untuk Server Actions
import { requireAdminYearContext } from "@/lib/admin-auth";

export async function adminAction() {
  const { user, year } = await requireAdminYearContext();
  // year = konteks tahun kerja saat ini
}
```

---

## 6. Sistem Kehadiran

### 6.1 Tabel Database

```sql
-- attendance_records
CREATE TABLE attendance_records (
  id TEXT PRIMARY KEY,
  user_id TEXT REFERENCES users(id),
  date DATE NOT NULL,
  year INTEGER NOT NULL,
  
  check_in_at TIMESTAMP,
  check_out_at TIMESTAMP,
  
  status attendance_status DEFAULT 'present',
  late_minutes INTEGER,
  early_minutes INTEGER,
  
  location_label TEXT,
  gps_accuracy REAL,
  is_mock_location_detected BOOLEAN,
  
  -- Flag system
  late_check_in BOOLEAN,
  early_check_out BOOLEAN,
  missing_check_out BOOLEAN,
  outside_location BOOLEAN,
  
  -- Admin override
  overridden_by_admin BOOLEAN,
  override_reason TEXT,
  
  UNIQUE(user_id, date)
);
```

### 6.2 Status Kehadiran

| Status | Deskripsi |
|--------|-----------|
| `PRESENT` | Hadir tepat waktu |
| `LATE` | Hadir terlambat |
| `ABSENT` | Tidak hadir tanpa keterangan (alpha) |
| `SICK` | Sakit (dengan surat/cuti) |
| `LEAVE` | Cuti tahunan |
| `PERMISSION` | Izin |
| `OUTSIDE_DUTY` | Tugas luar |
| `NO_SCHEDULE` | Tidak ada jadwal/libur |
| `NOT_CHECKED_IN` | Belum absen (hari ini/masa depan) |

### 6.3 Validasi Geofence

```typescript
// /src/lib/geolocation.ts
function isWithinGeofence(
  userLat: number,
  userLng: number,
  locationLat: number,
  locationLng: number,
  radiusMeters: number
): boolean {
  const distance = calculateHaversineDistance(
    userLat, userLng, locationLat, locationLng
  );
  return distance <= radiusMeters;
}
```

---

## 7. Manajemen Jadwal

### 7.1 Hierarki Data Jadwal

```
WorkSchedule (container bulanan)
  └── Shifts (jadwal harian per pegawai)
        ├── ShiftConfig (konfigurasi jam kerja)
        └── WorkLocation (lokasi kerja)
```

### 7.2 Tabel Database

```sql
-- work_schedules (container)
CREATE TABLE work_schedules (
  id TEXT PRIMARY KEY,
  month TEXT NOT NULL,      -- format: 2026-01
  year INTEGER NOT NULL,
  status TEXT DEFAULT 'draft'
);

-- shift_configs (template shift)
CREATE TABLE shift_configs (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  start_time TEXT,          -- format: 08:00
  end_time TEXT,            -- format: 17:00
  grace_minutes INTEGER,
  crosses_midnight BOOLEAN, -- untuk shift malam
  is_working_day BOOLEAN,
  color TEXT
);

-- shifts (jadwal aktual)
CREATE TABLE shifts (
  id TEXT PRIMARY KEY,
  schedule_id TEXT REFERENCES work_schedules(id),
  user_id TEXT REFERENCES users(id),
  date DATE NOT NULL,
  year INTEGER NOT NULL,
  shift_config_id TEXT REFERENCES shift_configs(id),
  location_id TEXT REFERENCES work_locations(id),
  UNIQUE(user_id, date)
);
```

### 7.3 Alur Assign Jadwal

```
Admin memilih:
  • Rentang tanggal
  • Daftar pegawai
  • Shift
  • Lokasi kerja
        │
        ▼
┌───────────────────────┐
│ bulkAssignSchedules() │
│                       │
│ 1. Validasi input     │
│ 2. Cari/buat schedule │
│ 3. Upsert shifts      │
│ 4. Return summary     │
└───────────────────────┘
```

---

## 8. Mesin Anomali & Kebijakan

### 8.1 Deteksi Anomali

Sistem secara otomatis mendeteksi pola-pola bermasalah:

| Anomali | Threshold Default | Deskripsi |
|---------|-------------------|-----------|
| `late_streak` | 3x berturut-turut | Terlambat berturut-turut |
| `absent_streak` | 2x berturut-turut | Alpha berturut-turut |
| `monthly_late` | ≥5x/bulan | Terlambat berkali-kali dalam sebulan |
| `monthly_absent` | ≥3x/bulan | Alpha berkali-kali dalam sebulan |

**File:** `/src/lib/anomaly-detection.ts`

### 8.2 Mesin Kebijakan

Sistem dapat menghasilkan proposal tindakan berdasarkan aturan kebijakan:

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│ Deteksi Anomali │ ──▶ │ Evaluasi Policy  │ ──▶ │ Buat Proposal   │
│                 │     │ Rule             │     │ Tindakan        │
└─────────────────┘     └──────────────────┘     └─────────────────┘
                                                         │
                                                         ▼
                                              ┌─────────────────────┐
                                              │ Admin Review        │
                                              │ • Approve           │
                                              │ • Dismiss           │
                                              └─────────────────────┘
```

**Tabel:** `policy_rules`, `policy_action_proposals`, `policy_action_logs`

---

## Catatan Tambahan

- Semua timestamp disimpan dalam UTC, konversi ke timezone lokal dilakukan di frontend
- Konteks tahun (`year`) digunakan untuk memfilter data per tahun anggaran
- Feature flags memungkinkan toggle fitur tanpa deployment ulang
- Audit log mencatat semua perubahan sensitif untuk keperluan audit

---

*Dokumen ini diperbarui terakhir: Januari 2026*
