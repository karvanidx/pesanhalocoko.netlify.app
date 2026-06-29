# PesanHaloCoko — Order Manager

Aplikasi **unofficial** untuk mempermudah agen/toko dalam merekap pesanan produk es krim HaloCoko secara otomatis. Hasil generate berupa teks siap kirim ke WhatsApp.

**[pesanhalocoko.netlify.app](https://pesanhalocoko.netlify.app)**

---

## Fitur

- 🛒 **Katalog 23 produk** — dikelompokkan per harga satuan, dengan stepper +/- per item
- 📋 **Form toko** — nama, alamat, tanggal pengiriman (auto-save ke localStorage)
- ✨ **Generate teks WhatsApp** — format terstruktur, satu klik copy
- 📁 **Fold group harga** — collapse/expand produk per kategori, animasi 60fps
- 📱 **PWA installable** — bisa dipasang di homescreen, offline support
- 📜 **Riwayat pesanan** — tersimpan lokal, bisa di-copy ulang

## Tech Stack

| | |
|---|---|
| **UI** | Alpine.js 3.x + Tailwind CSS (CDN) |
| **Offline** | Service Worker (cache-first strategy) |
| **PWA** | Web App Manifest + `beforeinstallprompt` custom UI |
| **Hosting** | Netlify (static, auto-deploy dari `main`) |
| **Build step** | Tidak ada |

## Menjalankan Lokal

```bash
# Serve root directory dengan static server apa pun:
npx serve .
# atau
python3 -m http.server 8000
```

Buka `http://localhost:3000` (atau port yang digunakan).

**Catatan:** `beforeinstallprompt` hanya bekerja di `localhost` atau `https://`. Tidak akan muncul di IP LAN (192.168.x.x).

## Struktur Proyek

```
├── index.html          # Seluruh aplikasi (HTML + CSS + JS)
├── sw.js               # Service Worker (offline cache)
├── manifest.json       # Web App Manifest (PWA)
├── pesanhalocoko-*.png # Icon PWA (192px + 512px)
├── og-image.png        # Open Graph image
├── previews/            # UI preview sebelum implementasi
│   └── subtotal-layout.html
├── notes/              # Catatan dan dokumentasi
│   └── 29-06-2026_pwa-without-framework.md
└── README.md
```

## Catatan Teknis

- **PWA Install Prompt** — implementasi custom lewat `beforeinstallprompt`. Detail proses trial-and-error (6+ iterasi) dan lesson learned: [`notes/29-06-2026_pwa-without-framework.md`](notes/29-06-2026_pwa-without-framework.md)
- **Animasi fold** — menggunakan Alpine `x-collapse` plugin (height-based, 60fps), bukan CSS `max-height` hack
- **Subtotal inline** — fixed height container (`h-5`) untuk mencegah layout shift saat quantity berubah

---

© Ervan Kurniawan · [Telegram](https://t.me/karvanidx) · [Trakteer](https://trakteer.id/ervankurniawan41)
