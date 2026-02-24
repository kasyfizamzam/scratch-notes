# Setup Database dengan WSL2 dan DBngin

Panduan ini ditujukan untuk developer yang menggunakan **Windows Subsystem for Linux (WSL2)** untuk menjalankan kode, namun menggunakan **DBngin** (di Windows) untuk manajemen database PostgreSQL.

## Masalah

WSL2 berjalan di jaringan terpisah (virtual network) dan tidak bisa mengakses `localhost` Windows secara langsung untuk semua port. Selain itu, Windows Firewall seringkali memblokir koneksi masuk dari WSL2 ke service yang berjalan di Windows.

## Solusi (The "Hack")

### 1. Dapatkan IP Host Windows

Dari terminal WSL2, jalankan perintah berikut untuk melihat IP host Windows (biasanya dari `/etc/resolv.conf`):

```bash
cat /etc/resolv.conf
# Output: nameserver 172.xx.xx.xx
```
Atau gunakan PowerShell di Windows: `ipconfig` (cari adapter vEthernet WSL).

### 2. Konfigurasi DBngin (PostgreSQL)

Secara default, DBngin hanya listen di `127.0.0.1` (localhost). Kita perlu mengubahnya agar listen di semua interface.

1.  Buka DBngin, stop server PostgreSQL.
2.  Klik kanan nama server -> **Show in Finder/Explorer** (buka folder data).
3.  Edit file `postgresql.conf`:
    ```ini
    # Ubah baris listen_addresses
    listen_addresses = '*'
    ```
4.  Edit file `pg_hba.conf`, tambahkan baris berikut di paling bawah untuk mengizinkan koneksi dari WSL:
    ```ini
    # Allow WSL access
    host    all             all             0.0.0.0/0            md5
    ```
5.  Start kembali server di DBngin.

### 3. Konfigurasi Windows Firewall

Anda perlu membuka port 5432 (atau port Postgres Anda) di Windows Firewall untuk traffic dari WSL.

Buka **PowerShell sebagai Administrator** dan jalankan:

```powershell
New-NetFirewallRule -DisplayName "WSL Postgres" -Direction Inbound -LocalPort 5432 -Action Allow -Protocol TCP
```

### 4. Update Environment Variable

Di file `.env` project (di dalam WSL), gunakan IP Windows yang didapat di langkah 1, bukan `localhost`.

**Contoh:**
Jika IP host adalah `172.31.224.1`:

```env
DATABASE_URL="postgres://postgres:password@172.31.224.1:5432/nama_database"
```

> **Tips:** IP WSL/Host bisa berubah setiap restart. Jika koneksi gagal tiba-tiba, cek ulang IP Anda.
