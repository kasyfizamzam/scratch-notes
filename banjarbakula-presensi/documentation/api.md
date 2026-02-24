# Dokumentasi API Server Action

Dokumen ini berisi spesifikasi teknis untuk Server Action yang digunakan dalam aplikasi Banjarbakula Presensi. Server Action ini berfungsi sebagai antarmuka antara komponen Client Side dan logika bisnis di sisi server.

## 1. Autentikasi (`src/app/actions/auth.ts`)

Menangani proses login, register, dan manajemen sesi pengguna umum.

### `login(data: LoginInput)`
Melakukan autentikasi pengguna menggunakan kredensial yang diberikan.
- **Input**: Object `LoginInput`
  - `username`: string (wajib)
  - `password`: string (wajib)
- **Output**: `Promise<{ success?: boolean; error?: string }>`
- **Deskripsi**: Memverifikasi username dan password. Jika valid, membuat sesi baru. Jika gagal, mengembalikan pesan error.

### `register(data: RegisterInput)`
Mendaftarkan akun baru untuk pengguna.
- **Input**: Object `RegisterInput`
  - `fullName`: string (wajib)
  - `username`: string (wajib)
  - `password`: string (wajib)
  - `divisionId`: string (opsional)
- **Output**: `Promise<{ success?: boolean; error?: string }>`
- **Deskripsi**: Membuat pengguna baru di database. Melakukan validasi username unik sebelum pembuatan.

### `logout()`
Mengakhiri sesi pengguna saat ini.
- **Input**: Tidak ada
- **Output**: `Promise<void>`
- **Deskripsi**: Menghapus cookie sesi dan melakukan redirect ke halaman login.

## 2. Absensi (`src/app/actions/attendance.ts`)

Menangani pencatatan kehadiran harian karyawan.

### `checkIn(location: { lat: number; lng: number })`
Mencatat waktu masuk (clock in) karyawan.
- **Input**: Object lokasi
  - `lat`: number (latitude)
  - `lng`: number (longitude)
- **Output**: `Promise<Result>`
- **Deskripsi**: Memvalidasi lokasi karyawan terhadap lokasi kantor yang diizinkan (Geofencing). Jika valid, mencatat waktu check-in. Gagal jika pengguna sudah check-in hari ini atau berada di luar radius.

### `checkOut(location: { lat: number; lng: number })`
Mencatat waktu pulang (clock out) karyawan.
- **Input**: Object lokasi
  - `lat`: number (latitude)
  - `lng`: number (longitude)
- **Output**: `Promise<Result>`
- **Deskripsi**: Memvalidasi lokasi dan mencatat waktu check-out. Menghitung durasi kerja dan status kehadiran (tepat waktu, pulang awal, dll).

### `getAttendanceHistory(date: Date)`
Mengambil riwayat absensi untuk tanggal tertentu.
- **Input**: `date` (Date object)
- **Output**: Data absensi (detail check-in/out, status)
- **Deskripsi**: Mengembalikan record absensi pengguna yang sedang login untuk bulan/tanggal yang diminta.

## 3. Cuti & Izin (`src/app/actions/leave.ts`)

Menangani pengajuan permohonan ketidakhadiran.

### `requestLeave(data: LeaveRequestInput)`
Mengajukan permohonan cuti, sakit, atau izin.
- **Input**: Object `LeaveRequestInput`
  - `type`: string ('annual', 'sick', 'permission', 'official_duty', dll)
  - `startDate`: Date
  - `endDate`: Date
  - `reason`: string
  - `attachment`: string (opsional, URL file)
- **Output**: `Promise<{ success: boolean; error?: string }>`
- **Deskripsi**: Validasi saldo cuti (jika tipe tahunan) dan membuat request baru. Mengirim notifikasi ke admin/atasan.

### `cancelLeave(id: string)`
Membatalkan permohonan cuti yang berstatus pending.
- **Input**: `id` (UUID request)
- **Output**: `Promise<{ success: boolean }>`
- **Deskripsi**: Menghapus atau mengubah status request menjadi dibatalkan. Hanya bisa dilakukan jika status masih pending.

## 4. Jadwal (`src/app/actions/schedule.ts`)

### `getSchedule(month: Date)`
Mengambil jadwal kerja personal karyawan.
- **Input**: `month` (Date)
- **Output**: Array jadwal harian
- **Deskripsi**: Mengembalikan daftar shift kerja untuk satu bulan penuh, termasuk hari libur dan konfigurasi shift spesifik.

## 5. Administrasi - Manajemen Karyawan (`src/app/actions/employee.ts`)

Action khusus untuk role Admin.

### `createEmployee(data: EmployeeInput)`
Menambahkan karyawan baru ke dalam sistem.
- **Input**: Object `EmployeeInput` (nama, username, divisi, jabatan, dll)
- **Output**: `Promise<{ success: boolean }>`
- **Deskripsi**: Admin membuat akun karyawan. Password awal di-generate otomatis atau default.

### `updateEmployee(id: string, data: EmployeeInput)`
Memperbarui data karyawan yang ada.
- **Input**: `id`, `data`
- **Output**: `Promise<{ success: boolean }>`

### `toggleEmployeeStatus(id: string, isActive: boolean)`
Mengaktifkan atau menonaktifkan akun karyawan.
- **Input**: `id`, `isActive`
- **Output**: `Promise<{ success: boolean }>`

### `resetEmployeePassword(id: string)`
Mereset password karyawan ke default.
- **Input**: `id`
- **Output**: `Promise<{ success: boolean }>`
- **Deskripsi**: Mengubah password karyawan menjadi default dan menandai akun untuk wajib ganti password saat login berikutnya.

## 6. Administrasi - Laporan & Override

### `getAttendanceReport(...)` (`src/app/actions/report.ts`)
Mengambil rekapitulasi kehadiran untuk pelaporan.
- **Input**: Range tanggal, filter departemen
- **Output**: Data agregat kehadiran (jumlah hadir, terlambat, sakit, cuti, alpha)

### `overrideAttendanceStatus(...)` (`src/app/actions/attendance-override.ts`)
Mengubah status kehadiran karyawan secara manual oleh admin.
- **Input**: `attendanceId`, `status` baru, `reason`
- **Output**: `Promise<{ success: boolean }>`
- **Deskripsi**: Admin dapat mengoreksi status kehadiran (misal: dari Alpha menjadi Sakit jika ada surat susulan). Mencatat log audit perubahan ini.

### `submitBulkAttendance(...)` (`src/app/actions/bulk-attendance.ts`)
Melakukan input absensi massal.
- **Input**: List karyawan, status, tanggal
- **Output**: `Promise<{ result: BulkResult }>`
- **Deskripsi**: Berguna untuk input data manual saat sistem error atau kejadian khusus (misal: penugasan luar kota massal).

## 7. Referensi Data (`src/app/actions/*.ts`)

Action untuk mengambil data referensi (Master Data).

- **`getDivisions()`** (`division.ts`): Daftar departemen/divisi.
- **`getPositions()`** (`position.ts`): Daftar jabatan.
- **`getWorkLocations()`** (`work-location.ts`): Daftar lokasi kantor yang valid untuk geofencing.
- **`getShiftConfigs()`** (`shift-config.ts`): Daftar konfigurasi jam kerja (Shift Pagi, Shift Malam, Non-Shift).

---
*Catatan: Semua Server Action dilindungi oleh validasi sesi dan role (RBAC). Action administratif memerlukan role `admin`.*
