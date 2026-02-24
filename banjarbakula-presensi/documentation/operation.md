# Dokumentasi Operasional

## Panduan Pemulihan Bencana & Operasional

Dokumen ini menjelaskan prosedur operasional untuk sistem absensi.

---

## 1. Strategi Backup

### Kebijakan Backup Database

| Jenis | Frekuensi | Retensi |
|-------|-----------|---------|
| Backup Penuh | Harian (jam 2 pagi) | 30 hari lokal |
| Incremental | Per jam (opsional) | 7 hari |
| Salinan Off-site | Mingguan | 1 salinan |

### Backup via Admin Panel

Aplikasi menyediakan fitur backup/restore melalui panel admin:

1. Login sebagai administrator
2. Buka menu **User → Advance → Backup Database**
3. Atau akses langsung: `/admin/config/database`
4. Klik **Download Backup** untuk mengunduh file backup JSON

File backup mencakup:
- Data master (Bidang, Jabatan, Lokasi Kerja, Shift Kerja)
- Data pegawai (Users)
- Rekap absensi (Attendance Records)
- Data cuti dan izin (Leave Requests, Leave Balances)
- Jadwal kerja (Work Schedules, Shifts)
- Log audit (Audit Logs)

### Backup Manual via CLI

Untuk backup langsung dari database PostgreSQL:

```bash
# Export menggunakan pg_dump
pg_dump -h $DB_HOST -U $DB_USER -d $DB_NAME -F c -f backup_$(date +%Y%m%d).dump
```

---

## 2. Prosedur Restore

### Prasyarat
- Akses ke file backup (.json atau .dump)
- Kredensial database (untuk CLI)
- Estimasi downtime: 5-15 menit tergantung ukuran data

### Restore via Admin Panel

1. Login sebagai administrator
2. Buka menu **User → Advance → Restore Database**
3. Atau akses langsung: `/admin/config/database?tab=restore`
4. Pilih file backup JSON
5. Klik **Restore Database** dan konfirmasi

**PERINGATAN:**
- Restore akan **MENIMPA SELURUH DATA** yang ada
- Admin users tidak akan terhapus
- Audit log baru akan dibuat setelah restore

### Restore Manual via CLI

```bash
# 1. Hentikan aplikasi (jika self-hosted)
# 2. Restore dari backup
pg_restore -h $DB_HOST -U $DB_USER -d $DB_NAME -c backup_YYYYMMDD.dump

# 3. Verifikasi integritas data
psql -h $DB_HOST -U $DB_USER -c "SELECT COUNT(*) FROM users;"
psql -h $DB_HOST -U $DB_USER -c "SELECT COUNT(*) FROM attendance_records;"

# 4. Restart aplikasi
```

### Estimasi Kehilangan Data
- Backup penuh: Maksimal 24 jam data
- Dengan backup incremental per jam: Maksimal 1 jam data

---

## 3. Keamanan Migrasi

### Aturan
1. Semua migrasi WAJIB reversible
2. JANGAN gunakan `DROP COLUMN` tanpa backup terlebih dahulu
3. Gunakan pola: tambah → migrasi → hapus

### Pola Penghapusan Kolom yang Aman

```sql
-- Langkah 1: Tambah kolom baru
ALTER TABLE users ADD COLUMN email_baru TEXT;

-- Langkah 2: Migrasi data
UPDATE users SET email_baru = email;

-- Langkah 3: Update aplikasi untuk menggunakan kolom baru
-- (Deploy perubahan kode)

-- Langkah 4: Hapus kolom lama (setelah verifikasi)
ALTER TABLE users DROP COLUMN email;
```

---

## 4. Feature Flags (Toggle Fitur)

### Daftar Flag yang Tersedia

| Flag | Variabel ENV | Default |
|------|--------------|---------|
| Mesin Anomali | `FEATURE_ANOMALY_ENGINE` | true |
| Deteksi Wajah | `FEATURE_FACE_DETECTION` | false |
| Export Massal | `FEATURE_MASS_EXPORT` | true |
| Deteksi Lokasi Palsu | `FEATURE_MOCK_LOCATION_DETECTION` | true |
| Mesin Kebijakan | `FEATURE_POLICY_ENGINE` | true |
| Cache Analitik | `FEATURE_ANALYTICS_CACHE` | true |

### Sub-Konfigurasi Feature Flags

| Flag | Variabel ENV | Default |
|------|--------------|---------|
| Maks anomali per run | `FEATURE_ANOMALY_MAX_PER_RUN` | 100 |
| Wajib foto check-in | `FEATURE_REQUIRE_PHOTO_CHECKIN` | false |
| Wajib foto check-out | `FEATURE_REQUIRE_PHOTO_CHECKOUT` | false |
| Maks baris export | `FEATURE_EXPORT_MAX_ROWS` | 10000 |
| Cooldown export (menit) | `FEATURE_EXPORT_COOLDOWN_MIN` | 5 |
| Auto flag lokasi palsu | `FEATURE_AUTO_FLAG_MOCK_LOCATION` | true |
| Auto buat proposal | `FEATURE_AUTO_CREATE_PROPOSALS` | false |
| Debug analitik | `ANALYTICS_DEBUG` | false |

### Menonaktifkan Fitur Darurat

Untuk menonaktifkan fitur di production:

```bash
# Tambahkan ke .env atau environment hosting
FEATURE_ANOMALY_ENGINE=false
FEATURE_POLICY_ENGINE=false

# Restart aplikasi
```

---


---

## 5. Rate Limits

Rate limiting sudah **AKTIF** untuk mencegah penyalahgunaan:

| Operasi | Limit | Window | Status |
|---------|-------|--------|--------|
| Login | 5 percobaan | 15 menit | ✅ Aktif |
| Submit Absensi | 20 submission | 1 jam | ✅ Aktif |
| API Umum | 200 request | 1 menit | Dikonfigurasi |
| Export | 5 export | 10 menit | Dikonfigurasi |
| Reset Password | 3 percobaan | 1 jam | Dikonfigurasi |

> **Catatan:** Limit absensi disesuaikan untuk 70+ user bersamaan (20 submit/jam per user).

---

## 6. Danger Zone (Mode Debug)

Halaman "Danger Zone" (`/admin/config/danger`) menyediakan alat untuk penghapusan data massal. Fitur ini dilindungi dengan **Konfirmasi Ganda** (wajib mengetik frasa konfirmasi).

### Fitur Tersedia

1. **Hapus Data Pegawai**
   - Menghapus semua user dengan role `employee`.
   - Menghapus data terkait: Absensi, Jadwal, Cuti, Log Override.
   - **AMAN:** User Admin tidak akan terhapus.

2. **Hapus Jadwal Kerja**
   - Menghapus semua data `shifts` dan `workSchedules`.
   - **AMAN:** Konfigurasi Shift (`shiftConfigs`) tidak terhapus.

3. **Hapus Absensi & Cuti**
   - Menghapus `attendanceRecords`, `leaveRequests`, `attendanceOverrideLog`.
   - Berguna untuk reset data transaksi sebelum go-live.

### Kapan Menggunakan?
- **Development:** Reset data dummy.
- **Pre-Go-Live:** Membersihkan data testing sebelum start production.
- **Maintenance:** Reset total (sangat jarang).

> ⚠️ **PERINGATAN:** Selalu lakukan **Backup Database** sebelum melakukan aksi apa pun di Danger Zone.

---

## 7. Observabilitas

### Log yang Wajib Dicatat

- ✅ Kegagalan autentikasi
- ✅ Operasi export
- ✅ Override absensi oleh admin
- ✅ Persetujuan/penolakan kebijakan
- ✅ Restore database

### Metrik Performa

- Latensi login
- Waktu muat dashboard
- Waktu submit absensi

### Mode Debug

```bash
# Aktifkan debug logging
OPS_DEBUG=true
ANALYTICS_DEBUG=true
```

---

## 7. Skenario Bencana

### Skenario 1: Database Down Saat Jam Check-in

**Gejala:** Pengguna tidak bisa check-in/out

**Respons:**
1. Aktifkan mode absensi manual (backup berbasis kertas)
2. Hubungi administrator database
3. Setelah pulih, admin memasukkan override manual

---

### Skenario 2: Internet Kantor Mati

**Gejala:** Perangkat mobile tidak bisa terhubung ke server

**Respons:**
1. Pengguna bisa menggunakan mobile data
2. Implementasikan dukungan PWA offline (peningkatan masa depan)
3. Admin memasukkan catatan absensi secara retroaktif

---

### Skenario 3: GPS Gagal

**Gejala:** Lokasi tidak bisa terdeteksi

**Respons:**
1. Izinkan check-in tanpa lokasi (tandai untuk review)
2. Admin memverifikasi dan menyetujui/menolak
3. JANGAN auto-reject

---

### Skenario 4: Jam Perangkat Salah

**Gejala:** Waktu check-in terlihat salah

**Respons:**
1. Sistem menggunakan WAKTU SERVER, bukan waktu perangkat
2. Waktu perangkat dicatat tapi tidak digunakan untuk kalkulasi
3. Tidak perlu tindakan - sistem sudah resilient

---

## 8. Pemisahan Environment

| Environment | Tujuan | Data |
|-------------|--------|------|
| Development | Testing lokal | Data dummy |
| Staging | Testing pre-production | Mirror skema production |
| Production | Sistem live | Data riil saja |

### Aturan

- ⚠️ JANGAN PERNAH gunakan data dummy di production
- ⚠️ Staging harus mirror skema production dengan tepat
- ⚠️ Test semua migrasi di staging sebelum production

---

## 9. Waktu & Timezone

### Aturan Backend

- Semua timestamp disimpan dalam UTC
- Konversi timezone terjadi di frontend
- Konteks tahun disimpan per sesi admin

### Konversi Frontend

```typescript
// Konversi UTC ke waktu lokal
const localTime = new Date(utcTime).toLocaleString('id-ID', {
  timeZone: 'Asia/Jakarta'
});
```

---

## 10. Kontak Darurat

| Peran | Tanggung Jawab |
|-------|----------------|
| System Admin | Masalah server/hosting |
| Database Admin | Pemulihan data, migrasi |
| Application Dev | Perbaikan bug, feature flags |

---

## 11. Fitur Absensi Massal (Bulk Attendance)

Fitur ini memungkinkan administrator untuk memasukkan data kehadiran (hadir, sakit, izin, dll) untuk banyak pegawai sekaligus dalam satu waktu. Fitur ini sangat berguna untuk:
- Kondisi darurat (internet mati, server down) dimana absensi dicatat manual lalu diinput belakangan.
- Penugasan luar kota massal (satu tim dinas luar).
- Input data rapelan/migrasi.

### Cara Menggunakan

1. **Akses Menu**
   - Masuk ke Admin Panel.
   - Navigasi ke **Konfigurasi** -> **Absensi Massal** (atau akses `/admin/config/bulk-attendance`).

2. **Langkah-langkah Input**
   - **Pilih Tanggal:** Tentukan tanggal kehadiran yang ingin diinput.
   - **Pilih Status:** Pilih jenis status (Hadir, Sakit, Izin, Cuti, atau Tugas Luar).
   - **Pilih Pegawai:** Centang daftar pegawai yang ingin diinputkan. Anda bisa mencari berdasarkan nama atau filter per divisi.
   - **Submit:** Klik tombol simpan.

3. **Catatan Penting**
   - Sistem akan membuat data absensi baru untuk pegawai yang dipilih.
   - Jika pegawai sudah memiliki data absensi pada tanggal tersebut, data lama akan **tertimpa (override)** oleh data baru ini.
   - Semua aksi ini tercatat di **Audit Log** sebagai "Bulk Attendance" oleh admin yang melakukannya.

---

## Checklist Sebelum Go-Live

- [ ] Lakukan drill restore minimal sekali
- [ ] Verifikasi semua feature flags berfungsi
- [ ] Konfirmasi jadwal backup aktif
- [ ] Dokumentasikan kontak darurat
- [ ] Test alur override absensi manual
- [ ] Verifikasi log tercatat dengan benar
- [ ] Test fitur backup/restore via admin panel
