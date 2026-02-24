# Panduan Deployment - Banjarbakula Presensi

Dokumen ini menjelaskan cara men-deploy aplikasi Banjarbakula Presensi ke server pribadi (VPS) atau server kantor.

Anda dapat menjalankan beberapa instance aplikasi ini di server yang berbeda asalkan setiap instance memiliki:
1.  **Database Sendiri** (atau database sama tapi beda nama DB).
2.  **Environment Variables Sendiri** (terutama `NEXTAUTH_URL` dan `NEXTAUTH_SECRET`).

---

## 🏗️ 1. Persiapan Server

Pastikan server Anda (Ubuntu/Debian recommended) sudah terinstall:

1.  **Node.js** (Versi 20 LTS atau terbaru)
    ```bash
    curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
    sudo apt-get install -y nodejs
    ```

2.  **PostgreSQL** (Database)
    ```bash
    sudo apt install postgresql postgresql-contrib
    ```

3.  **PNPM** (Package Manager)
    ```bash
    npm install -g pnpm
    ```

4.  **PM2** (Process Manager untuk menjalankan aplikasi di background)
    ```bash
    npm install -g pm2
    ```

5.  **Nginx** (Web Server / Reverse Proxy)
    ```bash
    sudo apt install nginx
    ```

---

## 🗄️ 2. Setup Database

Buat database baru untuk aplikasi presensi.

```bash
sudo -u postgres psql
```

```sql
CREATE DATABASE banjarbakula_presensi;
CREATE USER presensi_user WITH ENCRYPTED PASSWORD 'password_rahasia_anda';
GRANT ALL PRIVILEGES ON DATABASE banjarbakula_presensi TO presensi_user;
\q
```

---

## 🚀 3. Instalasi Aplikasi

1.  **Clone Repository**
    ```bash
    git clone https://github.com/kasyfizamzam/banjarbakula-presensi.git
    cd banjarbakula-presensi
    ```

2.  **Install Dependencies**
    ```bash
    pnpm install
    ```

3.  **Setup Environment Variables**
    Salin file contoh `.env.example` ke `.env.production` (atau simply `.env`).
    ```bash
    cp .env.example .env
    nano .env
    ```

    Isi konfigurasi penting:
    ```env
    DATABASE_URL="postgresql://presensi_user:password_rahasia_anda@localhost:5432/banjarbakula_presensi"
    NEXTAUTH_URL="http://103.xxx.xxx.xxx" # Ganti dengan IP Publik VPS Anda
    NEXTAUTH_SECRET="buat_string_random_panjang_disini"
    
    # Konfigurasi Storage (MinIO/S3) jika pakai upload foto
    S3_ENDPOINT="..."
    S3_ACCESS_KEY="..."
    S3_SECRET_KEY="..."
    ```

4.  **Migrasi Database**
    Jalankan migrasi untuk membuat tabel-tabel di database.
    ```bash
    pnpm migrate
    # Atau jika ingin seed data awal (admin default, shift default)
    pnpm db:seed
    ```

5.  **Build Aplikasi**
    ```bash
    pnpm build
    ```

---

## ▶️ 4. Menjalankan Aplikasi (PM2)

Gunakan PM2 agar aplikasi tetap berjalan di background meskipun terminal ditutup.

```bash
pm2 start npm --name "banjarbakula-presensi" -- start
pm2 save
pm2 startup
```

Aplikasi sekarang berjalan di port default `3000`.

---

## 🌐 5. Setup Nginx (Reverse Proxy)

Agar aplikasi bisa diakses via domain (bukan IP:Port) dan menggunakan HTTPS.

1.  **Buat Config Nginx**
    ```bash
    sudo nano /etc/nginx/sites-available/presensi
    ```

2.  **Isi Config**
    Ganti `presensi.domain-anda.com` dengan domain Anda.
    ```nginx
    server {
        listen 80;
        server_name _; # Gunakan underscore jika hanya akses via IP

        location / {
            proxy_pass http://localhost:3000;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_cache_bypass $http_upgrade;
        }
    }
    ```

    > [!IMPORTANT]
    > **Akses via IP & HTTPS**: Fitur PWA (Service Worker) dan Web Push **WAJIB** menggunakan HTTPS. Jika Anda hanya menggunakan IP, disarankan untuk tetap menggunakan Domain (gratis via DuckDNS/Freenom jika ada) atau mengonfigurasi Self-Signed Certificate, meskipun user akan melihat peringatan "Not Secure".

3.  **Aktifkan Config**
    ```bash
    sudo ln -s /etc/nginx/sites-available/presensi /etc/nginx/sites-enabled/
    sudo nginx -t
    sudo systemctl restart nginx
    ```

4.  **Setup HTTPS (SSL Gratis dengan Certbot)**
    ```bash
    sudo apt install certbot python3-certbot-nginx
    sudo certbot --nginx -d presensi.domain-anda.com
    ```

---

## 🔄 Update Aplikasi

Jika ada update terbaru dari repository:

```bash
cd ~/banjarbakula-presensi
git pull origin main
pnpm install
pnpm migrate
pnpm build
pm2 restart banjarbakula-presensi
```
