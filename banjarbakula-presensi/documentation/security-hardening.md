# Security Audit & Hardening Guide

Dokumen ini merinci temuan audit keamanan dan rekomendasi pengerasan (hardening) untuk aplikasi Banjarbakula Presensi. Audit ini disusun menggunakan keahlian dari **Specialized Security Auditor** dan **Security Hardening Orchestrator** untuk memastikan strategi pertahanan berlapis (defense-in-depth).

## Metodologi Audit (Skills Used)

Audit ini dilakukan dengan mengaktifkan dan mengikuti protokol dari:
1.  **security-auditor**: Digunakan untuk meninjau logika autentikasi (NextAuth v5), otorisasi (Year Context), dan pola akses database (Drizzle ORM).
2.  **security-scanning-security-hardening**: Digunakan untuk merancang strategi pengerasan infrastruktur, header keamanan, dan validasi input Server Actions.

---

## Temuan Utama (Critical & High)

### 1. Validasi Wajah di Sisi Klien (Critical)
**Temuan:** Fungsi `performAttendanceAction` di `src/app/actions/attendance.ts` menerima parameter `faceVerified` dan `faceMatchScore` langsung dari klien dan menyimpannya ke database tanpa verifikasi ulang di server.
**Risiko:** User dapat memanipulasi request untuk mengirimkan `faceVerified: true` secara manual meskipun wajah tidak cocok atau menggunakan foto (spoofing).
**Rekomendasi:**
- Kirim `faceDescriptor` dari klien ke server.
- Lakukan perbandingan descriptor di server menggunakan fungsi `isFaceMatch` yang sudah ada di `src/app/actions/face.ts`.
- Gunakan data descriptor yang tersimpan di database user sebagai referensi verifikasi.

### 2. Kurangnya Deteksi Keaktifan (Liveness Detection) (High)
**Temuan:** Sistem verifikasi wajah saat ini hanya membandingkan embedding (descriptor) tanpa mengecek apakah subjek adalah manusia hidup (bukan foto/video).
**Risiko:** Penipuan kehadiran menggunakan foto pegawai lain yang ditempel di depan kamera.
**Rekomendasi:**
- Implementasi tantangan acak di sisi klien (misal: "kedipkan mata", "tengok kanan/kiri").
- Gunakan library yang mendukung deteksi keaktifan atau analisis tekstur kulit jika memungkinkan.

### 3. Rate Limiting In-Memory (Medium)
**Temuan:** Implementasi `rateLimit` di `src/lib/rate-limit.ts` menggunakan `Map` di memori server.
**Risiko:** Data rate limit akan hilang setiap kali server restart atau redeploy. Pada lingkungan multi-instance (seperti Vercel atau Kubernetes), rate limit tidak tersinkronisasi antar instance.
**Rekomendasi:**
- Gunakan shared store seperti **Redis** untuk menyimpan data rate limit di lingkungan produksi.
- Implementasi library seperti `upstash/ratelimit` jika menggunakan platform serverless.

---

## Rekomendasi Pengerasan (Hardening)

### 1. Keamanan Header HTTP
Tambahkan security headers di `next.config.ts` untuk mencegah serangan umum seperti XSS dan Clickjacking. Meskipun `proxy.ts` menangani routing, header di level framework tetap krusial.

```typescript
// next.config.ts
const nextConfig = {
  async headers() {
    return [
      {
        source: '/(.*)',
        headers: [
          { key: 'X-Content-Type-Options', value: 'nosniff' },
          { key: 'X-Frame-Options', value: 'DENY' },
          { key: 'X-XSS-Protection', value: '1; mode=block' },
          { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
          { key: 'Permissions-Policy', value: 'camera=(self), microphone=(), geolocation=(self)' },
        ],
      },
    ];
  },
};
```

### 2. Validasi Input Server Actions
Gunakan schema Zod untuk memvalidasi setiap input yang masuk ke Server Actions. Saat ini beberapa action seperti `performAttendanceAction` belum memvalidasi tipe data secara ketat.

### 3. Perlindungan Data PII (Personally Identifiable Information)
**Descriptor Wajah** dan **Foto Wajah** adalah data biometrik yang sensitif.
- **Enkripsi:** Pertimbangkan untuk mengenkripsi kolom `face_descriptor` di database.
- **Storage:** Pastikan foto wajah (jika nanti diimplementasikan) disimpan di bucket privat dengan akses terkontrol (Presigned URLs).

### 4. Audit Trail yang Lebih Detail
Tambahkan informasi berikut ke `audit_logs` untuk mempermudah investigasi insiden:
- Alamat IP (Remote IP)
- User Agent (Browser/Device info)
- Metadata perubahan (data lama vs data baru)

### 5. Penguatan Otorisasi Admin
Pastikan semua fungsi administratif (seperti `resetUserPassword`) selalu mengecek `adminYearContext` jika memang admin tersebut hanya berwenang pada periode tertentu, guna mencegah eskalasi wewenang lintas tahun anggaran.

---

## Daftar Periksa (Checklist) Implementasi

- [x] Konfirmasi `src/proxy.ts` sebagai middleware aktif (Next.js 16)
- [x] Implementasi Security Headers di `next.config.ts`
- [x] Pindahkan verifikasi wajah ke Server-Side logic (`performAttendanceAction`)
- [x] Tambahkan validasi Zod untuk Presensi Server Actions
- [ ] Migrasi Rate Limiter ke Redis (Produksi)
- [x] Tambahkan validasi Zod di SEMUA Server Actions (Sisa: admin, employee, dll)
- [ ] Review kebijakan retensi data biometrik
