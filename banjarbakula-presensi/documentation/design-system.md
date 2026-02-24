# Dokumentasi Design System Banjarbakula

> **Dibuat oleh:** `ui-ux-designer` skill
> **Referensi Teknis:** `src/app/globals.css` (Konfigurasi Tailwind v4)

## 1. Filosofi Desain
Design system ini mengadopsi estetika **"Paper"** yang terinspirasi dari Perplexity.ai. Fokus utama adalah kejernihan visual dan kenyamanan membaca. Antarmuka menggunakan palet warna kontras tinggi namun lembut untuk mengurangi kelelahan mata, memberikan kesan profesional namun tetap modern.

---

## 2. Palet Warna (Color Palette)

Kami menggunakan variabel CSS native untuk mendukung mode Gelap/Terang secara dinamis.

### 2.1 Latar Belakang & Permukaan (Backgrounds)
| Token | Light Mode (Hex) | Dark Mode (Hex) | Penggunaan |
| :--- | :--- | :--- | :--- |
| `background` | `#FBFAF4` (Paper White) | `#0D1117` (Deep Space) | Latar belakang halaman utama |
| `secondary` | `#F3F3EE` | `#161B22` | Sidebar, Panel sekunder |
| `card` | `#FFFFFF` | `#161B22` | Kartu konten, Modal |
| `popover` | `#FFFFFF` | `#161B22` | Dropdown, Tooltip |

### 2.2 Teks & Konten
| Token | Light Mode (Hex) | Dark Mode (Hex) | Penggunaan |
| :--- | :--- | :--- | :--- |
| `foreground` | `#091717` (Off Black) | `#E6EDF3` | Teks utama |
| `muted-foreground` | `#65676B` | `#8B949E` | Teks sekunder, caption |
| `accent-foreground`| `#091717` | `#E6EDF3` | Teks di atas background accent |

### 2.3 Interaktif & Aksen
| Token | Light Mode (Hex) | Dark Mode (Hex) | Penggunaan |
| :--- | :--- | :--- | :--- |
| `primary` | `#20808D` (Turquoise) | `#2EA5B5` | Tombol utama, State aktif |
| `primary-foreground`| `#FFFFFF` | `#FFFFFF` | Teks di atas tombol primary |
| `accent` | `#EFF6F7` | `#1A2A2D` | State hover, highlight halus |
| `destructive` | `#DC2626` | `#F85149` | Pesan error, aksi hapus |
| `ring` | `#20808D` | `#2EA5B5` | Ring fokus pada input |

---

## 3. Tipografi

**Keluarga Font:**
- **Sans Serif:** Public Sans (`var(--font-public-sans)`) - Digunakan untuk UI utama.
- **Monospace:** Geist Mono (`var(--font-geist-mono)`) - Digunakan untuk kode atau data tabular.

**Skala:**
Menggunakan utilitas bawaan Tailwind (`text-sm`, `text-base`, `text-lg`, `text-xl`, dst).

---

## 4. Pola Komponen UI

### 4.1 Tombol (Button)
- **Primary:** `bg-primary text-primary-foreground hover:bg-primary/90`
- **Secondary:** `bg-secondary text-secondary-foreground hover:bg-secondary/80`
- **Ghost:** `hover:bg-accent hover:text-accent-foreground`
- **Destructive:** `bg-destructive text-destructive-foreground hover:bg-destructive/90`

### 4.2 Input Form
- **Border:** `border-input` (Warna: `#E8E8E3` / `#30363D`)
- **Fokus:** `ring-2 ring-ring ring-offset-2`
- **Background:** `bg-transparent` (agar menyatu dengan card/modal)

### 4.3 Kartu (Card)
- **Container:** `bg-card text-card-foreground border border-border shadow-sm rounded-xl`
- **Header:** `p-6` (Padding standar)
- **Content:** `p-6 pt-0`

### 4.4 Sidebar
Aplikasi menggunakan konfigurasi sidebar yang spesifik:
- **Background:** `bg-sidebar` (`#F3F3EE` / `#0D1117`)
- **Item Aktif:** `bg-sidebar-accent text-sidebar-accent-foreground`
- **Border:** `border-sidebar-border`

---

## 5. Panduan Implementasi (Tailwind v4)

Proyek ini menggunakan **Tailwind CSS v4** dengan konfigurasi zero-runtime (`@theme inline` di `globals.css`).

**Cara Penggunaan:**
Gunakan class utility Tailwind seperti biasa. Nilai warna akan otomatis menyesuaikan dengan tema aktif (Light/Dark) karena terhubung langsung ke CSS Variables.

**Contoh Kode:**
```tsx
<div className="bg-background text-foreground min-h-screen">
  <div className="bg-card border border-border rounded-xl p-4 shadow-sm">
    <h1 className="text-2xl font-bold text-primary">Judul Halaman</h1>
    <p className="text-muted-foreground">Deskripsi singkat di sini.</p>
    <button className="bg-primary text-primary-foreground px-4 py-2 rounded-md mt-4 transition-colors hover:bg-primary/90">
      Simpan Perubahan
    </button>
  </div>
</div>
```
