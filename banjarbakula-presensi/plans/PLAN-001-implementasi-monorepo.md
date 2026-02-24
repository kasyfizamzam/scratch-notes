# Rencana Implementasi Migrasi Monorepo

> **Untuk Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Tujuan:** Migrasi monolith Next.js yang ada ke struktur Polylith monorepo (Turborepo + pnpm workspaces) untuk meningkatkan modularitas dan performa build.

**Arsitektur:**
- **Monorepo Tool:** Turborepo
- **Package Manager:** pnpm dengan workspaces
- **Apps:** `apps/web` (aplikasi Next.js yang ada)
- **Packages:**
    - `packages/ui`: Komponen UI Bersama (Tailwind + Radix)
    - `packages/database`: Skema Drizzle ORM & klien
    - `packages/config`: TSConfig, ESLint Bersama
    - `packages/schema`: Skema Zod

**Tech Stack:** Next.js, TypeScript, Drizzle ORM, Tailwind CSS, pnpm.

---

### Tugas 1: Inisialisasi Monorepo

**File:**
- Buat: `pnpm-workspace.yaml`
- Buat: `turbo.json`
- Buat: `package.json` (root)
- Buat: `.gitignore` (root)

**Langkah 1: Buat `package.json` Root**
```json
{
  "name": "banjarbakula-monorepo",
  "version": "0.0.0",
  "private": true,
  "scripts": {
    "build": "turbo build",
    "dev": "turbo dev",
    "lint": "turbo lint",
    "format": "prettier --write \"**/*.{ts,tsx,md}\""
  },
  "devDependencies": {
    "turbo": "^2.3.0",
    "prettier": "^3.0.0"
  },
  "packageManager": "pnpm@9.0.0"
}
```

**Langkah 2: Definisikan Workspace**
Buat `pnpm-workspace.yaml`:
```yaml
packages:
  - "apps/*"
  - "packages/*"
```

**Langkah 3: Konfigurasi Turbo**
Buat `turbo.json`:
```json
{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": [".next/**", "!.next/cache/**"]
    },
    "lint": {},
    "dev": {
      "cache": false,
      "persistent": true
    }
  }
}
```

**Langkah 4: Perbarui Gitignore**
Perbarui `.gitignore` agar mengabaikan folder build monorepo:
```text
.turbo
node_modules
dist
.next
```

**Langkah 5: Install Dependensi**
Jalankan: `pnpm install`

---

### Tugas 2: Pindahkan Aplikasi Web

**File:**
- Buat: direktori `apps/web`
- Pindahkan: File source root ke `apps/web`
- Modifikasi: `apps/web/package.json`

**Langkah 1: Buat Direktori**
Jalankan: `mkdir -p apps/web`

**Langkah 2: Pindahkan File Proyek**
Jalankan:
```bash
mv public apps/web/
mv src apps/web/
mv .env* apps/web/
mv next.config.mjs apps/web/
mv tailwind.config.ts apps/web/
mv postcss.config.mjs apps/web/
mv tsconfig.json apps/web/
mv biome.json apps/web/
mv components.json apps/web/
mv drizzle.config.ts apps/web/
mv package.json apps/web/
```

**Langkah 3: Perbarui Nama Paket**
Modifikasi `apps/web/package.json`:
```json
{
  "name": "@banjarbakula/web",
  "version": "0.1.0"
}
```

**Langkah 4: Verifikasi Perpindahan**
Jalankan: `cd apps/web && pnpm install && pnpm build`
Ekspektasi: Build berhasil.

---

### Tugas 3: Ekstraksi Paket Database

**File:**
- Buat: `packages/database/package.json`
- Buat: `packages/database/tsconfig.json`
- Pindahkan: `apps/web/src/db` -> `packages/database/src`
- Buat: `packages/database/src/index.ts`

**Langkah 1: Setup Package.json**
Buat `packages/database/package.json`:
```json
{
  "name": "@banjarbakula/database",
  "version": "0.0.0",
  "main": "./src/index.ts",
  "types": "./src/index.ts",
  "exports": {
    ".": "./src/index.ts"
  },
  "scripts": {
    "db:push": "drizzle-kit push",
    "db:studio": "drizzle-kit studio"
  },
  "dependencies": {
    "drizzle-orm": "^0.30.0",
    "postgres": "^3.4.0"
  },
  "devDependencies": {
    "drizzle-kit": "^0.20.0",
    "typescript": "^5.0.0"
  }
}
```

**Langkah 2: Pindahkan Kode DB**
Jalankan: `mkdir -p packages/database/src`
Jalankan: `mv apps/web/src/db/* packages/database/src/`

**Langkah 3: Ekspor Skema**
Buat `packages/database/src/index.ts`:
```ts
export * from "./schema";
export * from "./client"; // Pastikan logika koneksi klien dapat diadaptasi
```

**Langkah 4: Konsumsi di Aplikasi Web**
- Di `apps/web/package.json`, tambahkan dependensi `"@banjarbakula/database": "workspace:*"`.
- Perbarui impor di `apps/web/src/app/actions/*.ts`:
  `import { db } from "@banjarbakula/database";`

---

### Tugas 4: Ekstraksi Komponen UI Bersama

**File:**
- Buat: `packages/ui/package.json`
- Buat: `packages/ui/src/button.tsx`
- Pindahkan: `apps/web/src/components/ui` -> `packages/ui/src`

**Langkah 1: Setup Package.json**
Buat `packages/ui/package.json`:
```json
{
  "name": "@banjarbakula/ui",
  "version": "0.0.0",
  "main": "./src/index.ts",
  "types": "./src/index.ts",
  "exports": {
    ".": "./src/index.ts"
  },
  "peerDependencies": {
    "react": "^19.0.0"
  }
}
```

**Langkah 2: Pindahkan Komponen UI**
Jalankan: `mkdir -p packages/ui/src`
Jalankan: `mv apps/web/src/components/ui/* packages/ui/src/`
*(Catatan: Pastikan `utils.ts` untuk helper `cn` juga dipindahkan/dibagikan)*

**Langkah 3: Ekspor Komponen**
Buat `packages/ui/src/index.ts`:
```ts
export * from "./button";
export * from "./input";
// ... ekspor komponen lainnya
```

**Langkah 4: Konsumsi di Aplikasi Web**
- Di `apps/web/package.json`, tambahkan `"@banjarbakula/ui": "workspace:*"`
- Perbarui impor: `import { Button } from "@banjarbakula/ui";`

---

### Tugas 5: Transpilasi & Konfigurasi

**File:**
- Modifikasi: `apps/web/next.config.mjs`

**Langkah 1: Perbarui Konfigurasi Next**
```js
const nextConfig = {
  transpilePackages: ["@banjarbakula/ui", "@banjarbakula/database"],
  // ... konfigurasi lainnya
};
export default nextConfig;
```

**Langkah 2: Verifikasi Build**
Jalankan: `pnpm build` (dari root).
Ekspektasi: Semua paket dan aplikasi berhasil di-build.
