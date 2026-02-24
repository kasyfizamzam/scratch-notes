# Panduan Kontribusi

Dokumen ini berisi panduan bagi developer yang ingin berkontribusi atau melakukan pengembangan pada proyek Banjarbakula Presensi.

---

## Daftar Isi

1. [Persiapan Lingkungan](#1-persiapan-lingkungan)
2. [Struktur Kode](#2-struktur-kode)
3. [Konvensi Penulisan Kode](#3-konvensi-penulisan-kode)
4. [Alur Kerja Pengembangan](#4-alur-kerja-pengembangan)
5. [Menambahkan Fitur Baru](#5-menambahkan-fitur-baru)
6. [Pengujian](#6-pengujian)
7. [Dokumentasi](#7-dokumentasi)

---

## 1. Persiapan Lingkungan

### 1.1 Prasyarat

Pastikan sistem Anda telah terinstall:

| Software | Versi Minimum | Verifikasi |
|----------|---------------|------------|
| Node.js | 20.x | `node -v` |
| pnpm | 9.x | `pnpm -v` |
| PostgreSQL | 16.x | `psql --version` |
| Docker (opsional) | Latest | `docker -v` |

### 1.2 Clone dan Setup

```bash
# Clone repository
git clone https://github.com/your-org/banjarbakula-presensi.git
cd banjarbakula-presensi

# Install dependencies
pnpm install

# Salin file environment
cp .env.example .env
```

### 1.3 Konfigurasi Environment

Edit file `.env` dengan konfigurasi berikut:

```env
# Database
DATABASE_URL="postgres://username:password@localhost:5432/banjarbakula_presensi"

# Autentikasi
AUTH_SECRET="generate-dengan-openssl-rand-base64-32"
JWT_SECRET="secret-jwt-anda"

# URL Aplikasi
NEXT_PUBLIC_APP_URL="http://localhost:3000"

# Mode Development
NODE_ENV="development"
```

### 1.4 Setup Database

**Opsi A: Menggunakan Docker**
```bash
docker-compose up -d
```

**Opsi B: PostgreSQL Lokal**
```bash
createdb banjarbakula_presensi
```

### 1.5 Migrasi Database

```bash
# Jalankan migrasi
pnpm migrate

# (Opsional) Seed data awal
pnpm tsx src/scripts/seed-admin.ts
```

### 1.6 Menjalankan Server Development

```bash
pnpm dev
```

Aplikasi akan berjalan di `http://localhost:3000`

---

## 2. Struktur Kode

### 2.1 Direktori Utama

```
src/
├── app/                    # Next.js App Router (halaman & API)
├── components/             # Komponen React
├── db/                     # Database layer (Drizzle ORM)
├── lib/                    # Logika bisnis & utilitas
├── hooks/                  # Custom React hooks
├── types/                  # TypeScript type definitions
└── scripts/                # Script utilitas (seed, migration helper)
```

### 2.2 Konvensi Penamaan File

| Jenis | Konvensi | Contoh |
|-------|----------|--------|
| Halaman | `page.tsx` | `app/admin/page.tsx` |
| Layout | `layout.tsx` | `app/admin/layout.tsx` |
| Server Action | `kebab-case.ts` | `actions/employee.ts` |
| Komponen | `PascalCase.tsx` | `EmployeeTable.tsx` |
| Utilitas | `kebab-case.ts` | `date-utils.ts` |
| Hook | `use-camelCase.ts` | `use-debounce.ts` |

---

## 3. Konvensi Penulisan Kode

### 3.1 Komentar

**Wajib menggunakan Bahasa Indonesia** untuk semua komentar dalam codebase.

```typescript
// ✅ Benar
// Validasi lokasi pegawai terhadap geofence kantor
function validateGeofence() { ... }

// ❌ Salah
// Validate employee location against office geofence
function validateGeofence() { ... }
```

### 3.2 Teks UI

**Wajib menggunakan Bahasa Indonesia** untuk semua teks yang terlihat di antarmuka pengguna.

```tsx
// ✅ Benar
<Button>Simpan Data</Button>
<Label>Nama Lengkap</Label>

// ❌ Salah
<Button>Save Data</Button>
<Label>Full Name</Label>
```

### 3.3 Format Kode

Proyek ini menggunakan **Biome** untuk linting dan formatting.

```bash
# Cek lint errors
pnpm lint

# Format kode
pnpm format
```

### 3.4 TypeScript Strict

Semua kode harus ditulis dengan TypeScript strict mode. Hindari penggunaan `any` kecuali sangat diperlukan.

```typescript
// ✅ Benar
function processData(data: EmployeeData): ProcessedResult {
  return { ... };
}

// ❌ Hindari
function processData(data: any): any {
  return { ... };
}
```

### 3.5 Import Order

Import harus diurutkan sesuai konvensi Biome (otomatis diformat):

1. External libraries (react, next, dll)
2. Internal aliases (@/lib, @/components)
3. Relative imports (./local)

---

## 4. Alur Kerja Pengembangan
 
 Detail lengkap mengenai workflow Git dan Pull Request dapat dilihat di documentasi terpisah:
 
 👉 **[Panduan Workflow Git & Pull Request](./git-workflow.md)**
 
 Dokumen tersebut mencakup:
 - Struktur Branch (main, development, feature)
 - Langkah pembuatan Feature Branch
 - Standar Commit Message (Conventional Commits)
 - Proses Pull Request dan Code Review
 - Deployment ke Production


---

## 5. Menambahkan Fitur Baru

### 5.1 Menambahkan Halaman Admin Baru

**Langkah 1:** Buat folder halaman

```
src/app/admin/nama-fitur/
├── page.tsx              # Server Component (halaman utama)
└── components/           # Client Components
    └── NamaFiturClient.tsx
```

**Langkah 2:** Implementasi Server Component

```typescript
// src/app/admin/nama-fitur/page.tsx
import { requireAdminYearContextOrRedirect } from "@/lib/admin-auth";
import { NamaFiturClient } from "./components/NamaFiturClient";

export default async function NamaFiturPage() {
  const { year } = await requireAdminYearContextOrRedirect();
  
  // Fetch data di server
  const data = await fetchData(year);
  
  return <NamaFiturClient data={data} />;
}
```

**Langkah 3:** Tambahkan ke sidebar navigation

```typescript
// src/components/admin/layout/sidebar-config.ts
export const sidebarItems = [
  // ... existing items
  {
    title: "Nama Fitur",
    href: "/admin/nama-fitur",
    icon: IconName,
  },
];
```

### 5.2 Menambahkan Server Action Baru

**Langkah 1:** Buat file action

```typescript
// src/app/actions/nama-fitur.ts
"use server";

import { db } from "@/db";
import { requireAdminYearContext } from "@/lib/admin-auth";
import { z } from "zod";

// Definisi schema validasi
const inputSchema = z.object({
  field1: z.string().min(1),
  field2: z.number().positive(),
});

export async function createNamaFitur(data: z.infer<typeof inputSchema>) {
  // 1. Validasi autentikasi
  const { user, year } = await requireAdminYearContext();
  
  // 2. Validasi input
  const validated = inputSchema.parse(data);
  
  // 3. Eksekusi logika bisnis
  const result = await db.insert(tableName).values({
    ...validated,
    year,
    createdBy: user.id,
  });
  
  // 4. Return hasil
  return { success: true, data: result };
}
```

### 5.3 Menambahkan Perubahan Database

**Langkah 1:** Edit schema

```typescript
// src/db/schema.ts
export const namaTable = pgTable("nama_table", {
  id: text().primaryKey(),
  field1: text().notNull(),
  field2: integer().notNull(),
  createdAt: timestamp("created_at").defaultNow().notNull(),
});
```

**Langkah 2:** Generate migration

```bash
drizzle-kit generate
```

**Langkah 3:** Review & jalankan migration

```bash
pnpm migrate
```

---

## 6. Pengujian

### 6.1 Type Check

```bash
npx tsc --noEmit
```

### 6.2 Lint Check

```bash
pnpm lint
```

### 6.3 Build Test

```bash
pnpm build
```

### 6.4 Manual Testing Checklist

Sebelum membuat PR, pastikan:

- [ ] Tidak ada error TypeScript
- [ ] Tidak ada lint errors
- [ ] Build berhasil
- [ ] Fitur berjalan sesuai spesifikasi
- [ ] UI responsive di mobile dan desktop
- [ ] Tidak ada regresi pada fitur existing

---

## 7. Dokumentasi

### 7.1 Dokumentasi Wajib

Setiap fitur baru harus dilengkapi:

1. **Komentar kode** - Jelaskan logika bisnis yang kompleks
2. **Update API docs** - Jika menambah Server Action baru
3. **Update README** - Jika fitu signifikan

### 7.2 Lokasi Dokumentasi

| Jenis | Lokasi |
|-------|--------|
| API Reference | `docs/api.md` |
| Arsitektur | `docs/arsitektur.md` |
| Deployment | `docs/deployment.md` |
| Operasional | `docs/operation.md` |
| Kontribusi | `docs/kontribusi.md` |
| Database Schema | `docs/database.md` |

---

## Tips & Best Practices

### Server Components vs Client Components

```tsx
// ✅ Server Component - untuk fetch data
export default async function Page() {
  const data = await db.select().from(table);
  return <ClientComponent data={data} />;
}

// ✅ Client Component - untuk interaktivitas
"use client";
export function ClientComponent({ data }) {
  const [state, setState] = useState();
  return <button onClick={() => setState(...)}>Click</button>;
}
```

### Hindari N+1 Query

```typescript
// ❌ Salah - N+1 query
for (const user of users) {
  const attendance = await db.select().from(records).where(eq(userId, user.id));
}

// ✅ Benar - Batch query
const attendances = await db.select().from(records).where(inArray(userId, userIds));
const attendanceMap = new Map(attendances.map(a => [a.userId, a]));
```

### Gunakan Feature Flags

```typescript
import { isFeatureEnabled } from "@/lib/feature-flags";

if (isFeatureEnabled("newFeature")) {
  // Fitur baru
} else {
  // Fallback
}
```

---

*Terima kasih telah berkontribusi pada proyek ini!*
