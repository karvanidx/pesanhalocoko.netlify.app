# Arsitektur & Alur Program — PesanHaloCoko

Blueprint lengkap aplikasi single-file PWA ini. Ditulis untuk referensi pengembangan dan onboarding.

---

## 1. Data Layer (Static Constants)

```
┌─────────────────────────────────────────────────────────┐
│                    PRICE_MAP (23 item)                    │
│  id → { pcs, price, discontinued? }                     │
│  Contoh: 1 → { pcs: 60, price: 93000 }                 │
│         7 → { pcs: 50, price: 77500, discontinued }     │
├─────────────────────────────────────────────────────────┤
│                   initialDB (23 item)                    │
│  { id, name, price (per-pcs), qty: 0 }                  │
│  Contoh: { name: "SuMiCo Milo", price: 2000 }           │
└─────────────────────────────────────────────────────────┘
```

`PRICE_MAP` menyimpan metadata fisik produk (isi per dus, harga per dus, status discontinued). `initialDB` menyimpan data katalog (nama, harga satuan). Keduanya digabung saat Alpine `init()` — setiap item di `initialDB` di-*enrich* dengan `pcs`, `pricePerDus`, dan `discontinued` dari `PRICE_MAP`.

---

## 2. Alpine State — `app()` Component

### State

| Variable | Type | Default | Keterangan |
|---|---|---|---|
| `currentTab` | string | `'order'` | Tab aktif: `'order'` atau `'history'` |
| `store` | object | `{ name:'', address:'', date:'YYYY-MM-DD' }` | Data toko/agen |
| `items` | array | 23 item (initialDB + PRICE_MAP) | Katalog produk dengan stok metadata |
| `result` | string | `''` | Teks WhatsApp yang di-generate |
| `history` | array | `[]` | Riwayat pesanan (max 50 entry) |
| `showHistoryDetail` | number\|null | `null` | ID entry history yang sedang di-expand |
| `toast` | object | `{ show:false, message:'', timeout:null }` | State notifikasi toast |
| `collapsedGroups` | object | `{}` | Key=price, value=true jika group di-fold |
| `showInstallPill` | boolean | `false` | Visibility custom install prompt |
| `deferredPrompt` | object\|null | `null` | Referensi BeforeInstallPromptEvent |

### Getters (Computed)

| Getter | Return | Rumus |
|---|---|---|
| `totalDus` | number | Σ qty semua item |
| `totalPcs` | number | Σ (qty × pcs per dus) |
| `totalPrice` | number | Σ (qty × price per dus) |
| `productGroups` | array | items dikelompokkan per `price`, diurutkan ascending |

### Methods

| Method | Trigger | Efek |
|---|---|---|
| `init()` | Alpine mount | Restore localStorage, pasang listener PWA, watch tab |
| `save()` | Setiap perubahan store/qty | Persist store+items ke `hc_app_v5_final` |
| `saveHistory()` | Setelah generate/delete | Persist history ke `hc_history` |
| `adjust(id, val)` | Tap +/- stepper | Ubah qty ±1 (min 0), vibrate(10), save() |
| `clear()` | Klik "Reset Pilihan" | Reset semua qty ke 0, save() |
| `generate()` | Klik "Buat Pesanan" | Format teks WA, simpan history, copy, buka modal |
| `copyText(text, fromGenerate)` | Generate / salin ulang | Clipboard API, vibrate, toast |
| `toggleGroup(price)` | Klik chevron group header | Toggle collapsedGroups[price] |
| `groupActiveCount(group)` | Reaktif (di template) | Hitung item dengan qty>0 dalam satu group |
| `installApp()` | Klik pill "Pasang" | Trigger deferredPrompt.prompt() |
| `showToast(msg)` | Copy / error | Tampilkan toast 2.5 detik |
| `formatRupiah(num)` | Reaktif (di template) | `150000` → `"Rp 150.000"` |
| `formatDate(dateStr)` | generate() | `"2026-06-29"` → `"29-06-2026"` |
| `deleteHistoryEntry(id)` | Klik delete di history | Filter + saveHistory() |
| `clearHistory()` | Klik "Hapus Semua" | Kosongkan history + saveHistory() |

---

## 3. Alur Utama — Dari Buka Sampai Pesanan Jadi

```
BUKA HALAMAN
  │
  ├─1─ tailwind.config (CDN) diinisialisasi
  ├─2─ Alpine.js + Collapse plugin dimuat (defer)
  │     Urutan: Collapse plugin → Alpine core (sesuai docs)
  ├─3─ Global listener beforeinstallprompt dipasang
  │     (top-level script, sebelum Alpine mount)
  │
  ▼
ALPINE MOUNT — init()
  │
  ├─ Check: apakah sudah standalone mode? → skip install pill
  ├─ Check: __deferredPrompt sudah ada? → tampilkan pill
  ├─ Listen: beforeinstallprompt (fallback untuk visit berikutnya)
  │
  ├─ Restore: localStorage 'hc_app_v5_final' → store + items.qty
  ├─ Restore: localStorage 'hc_history' → history[]
  │
  └─ Watch: currentTab → scrollTo(top, instant)

═══════════════════════════════════════════════════════════
TAB "PESAN" (currentTab='order')
═══════════════════════════════════════════════════════════
  │
  │  FORM TOKO (auto-save setiap input)
  │  ├─ Input nama toko  → x-model="store.name"   → @input → save()
  │  ├─ Input alamat     → x-model="store.address" → @input → save()
  │  └─ Input tanggal    → x-model="store.date"    → @input → save()
  │     Default: today (new Date().toISOString().split('T')[0])
  │
  │  ┌─ KATALOG PRODUK (per productGroups) ─────────────────┐
  │  │                                                        │
  │  │  GROUP HEADER                                          │
  │  │  ├─ "Rp 2.000 / Pcs" label                            │
  │  │  ├─ Horizontal line (flex-1)                           │
  │  │  │   └─ Jika folded + ada item: overlay "X dipesan"   │
  │  │  └─ [▼] fold button (36×36px touch target)            │
  │  │                                                        │
  │  │  PRODUCT ROW                                           │
  │  │  ├─ Nama produk + "Stop Produksi" badge (jika ada)    │
  │  │  ├─ "Isi X pcs/dus" + subtotal inline "= Rp XX.XXX"  │
  │  │  │   └─ Fixed height (h-5) → no layout shift          │
  │  │  └─ Stepper [-][qty][+]                               │
  │  │                                                        │
  │  │  INTERAKSI                                             │
  │  │  ├─ KLIK [+] → adjust(id, +1)                         │
  │  │  │   ├─ navigator.vibrate(10) — haptic feedback       │
  │  │  │   ├─ item.qty++ → save() → localStorage            │
  │  │  │   └─ Subtitel muncul inline (x-transition.opacity) │
  │  │  │                                                      │
  │  │  ├─ KLIK [-] → adjust(id, -1)                         │
  │  │  │   └─ qty min 0, hilang subtitel saat qty=0         │
  │  │  │                                                      │
  │  │  ├─ KLIK [▼] → toggleGroup(price)                     │
  │  │  │   └─ x-collapse.duration.300ms — 60fps animation   │
  │  │  │                                                      │
  │  │  └─ PRODUK DISCONTINUED                                │
  │  │      ├─ Row: opacity-50 + grayscale                   │
  │  │      └─ Stepper: hidden (x-show="!item.discontinued") │
  │  │                                                        │
  │  └────────────────────────────────────────────────────────┘
  │
  │  CHECKOUT BAR (muncul saat totalDus > 0)
  │  ├─ Fixed bottom-14, z-30 (di BELAKANG nav bar z-40)
  │  ├─ Slide up/down dengan translate-y-full (no opacity)
  │  │   Easing: cubic-bezier(0,0,0.2,1) enter / (0.4,0,1,1) leave
  │  │
  │  ├─ totalDus < 6:
  │  │   └─ Bar abu-abu, tombol "Kurang X Dus" (disabled)
  │  │
  │  └─ totalDus ≥ 6:
  │      └─ Bar gelap (bg-slate-900), tombol "Buat Pesanan" → generate()
  │
  │  ┌─ generate() FLOW ─────────────────────────────────────┐
  │  │                                                        │
  │  │  Check: totalDus < 6? → return (tidak akan terjadi)    │
  │  │                                                        │
  │  │  Check: nama/alamat kosong? → confirm dialog          │
  │  │  ├─ Cancel → scroll to top + focus input               │
  │  │  └─ OK → lanjut (pakai '-')                            │
  │  │                                                        │
  │  │  FORMAT TEKS WHATSAPP:                                 │
  │  │  ┌──────────────────────────────┐                     │
  │  │  │ *HaloCoko Order*            │                     │
  │  │  │ ═════════════════           │                     │
  │  │  │ Toko   : Toko Berkah        │                     │
  │  │  │ Alamat : Jl. Melati No. 5   │                     │
  │  │  │ Waktu  : 29-06-2026        │                     │
  │  │  │                              │                     │
  │  │  │ *List Produk:*               │                     │
  │  │  │ 1. SuMiCo Milo = 3 Dus     │                     │
  │  │  │ 2. Yogurt Krispi = 2 Dus   │                     │
  │  │  │                              │                     │
  │  │  │ *TOTAL: 5 Dus*              │                     │
  │  │  └──────────────────────────────┘                     │
  │  │                                                        │
  │  │  SIMPAN HISTORY:                                       │
  │  │  ├─ Buat historyEntry { id:Date.now(), ... }          │
  │  │  ├─ history.unshift(entry) — entry baru di atas        │
  │  │  └─ Slice max 50 entry → saveHistory()                │
  │  │                                                        │
  │  │  OUTPUT:                                               │
  │  │  ├─ copyText(txt, fromGenerate=true) → clipboard      │
  │  │  │   └─ No toast (fromGenerate=true)                  │
  │  │  └─ result-modal.showModal() — preview monospace      │
  │  │                                                        │
  │  └────────────────────────────────────────────────────────┘
  │
  ▼
═══════════════════════════════════════════════════════════
TAB "RIWAYAT" (currentTab='history')
═══════════════════════════════════════════════════════════
  │
  ├─ EMPTY STATE (history.length === 0):
  │   └─ Icon dokumen + "Belum Ada Riwayat"
  │
  └─ HISTORY LIST:
      ├─ Entry card (left accent bar, bg-primary)
      ├─ Info: nama toko, tanggal (format DD-MM-YYYY)
      ├─ Summary: total dus, total pcs, total harga
      ├─ "Lihat Rincian" → expand detail per item (x-show toggle)
      ├─ "Salin Ulang" → copyText(entry.text) + toast
      └─ Delete icon → confirm → filter + saveHistory()
```

---

## 4. Side Systems

### 4.1 Service Worker (`sw.js`)

```
┌──────────────────────────────────────────────────────────┐
│  CACHE_NAME: 'halocoko-cache-v2'                         │
│                                                          │
│  INSTALL: cache index.html, manifest.json, icon (192+512)│
│           self.skipWaiting() — langsung aktif            │
│                                                          │
│  FETCH:   cache-first strategy                           │
│           caches.match(request) → cache hit? return      │
│           cache miss → fetch(network)                    │
│                                                          │
│  ACTIVATE: hapus cache lama (versi sebelumnya)            │
│                                                          │
│  REGISTER: <script> di index.html, baris sebelum </body> │
└──────────────────────────────────────────────────────────┘
```

### 4.2 PWA Install Flow

```
┌──────────────────────────────────────────────────────────┐
│  Global listener (top-level script):                     │
│  window.addEventListener('beforeinstallprompt', e => {   │
│    e.preventDefault()  ← blokir native mini-infobar       │
│    simpan e → window.__deferredPrompt                    │
│    tampilkan pill "Pasang" di header                     │
│  })                                                      │
│                                                          │
│  Alpine init():                                          │
│  - Check window.__deferredPrompt → jika ada, tampilkan   │
│  - Listen beforeinstallprompt (fallback visit berikutnya)│
│  - Check display-mode: standalone → skip                 │
│                                                          │
│  User klik pill → installApp():                          │
│  - deferredPrompt.prompt() → browser dialog native       │
│  - userChoice.then() → hide pill jika accepted           │
│  - deferredPrompt = null (hanya bisa dipakai sekali)     │
│                                                          │
│  ⚠️  Visit 1: native prompt mungkin lolos (race condition)│
│  ✅ Visit 2+: custom pill muncul (SW caching → load cepat)│
└──────────────────────────────────────────────────────────┘
```

### 4.3 LocalStorage

| Key | Struktur | Keterangan |
|---|---|---|
| `hc_app_v5_final` | `{ store, items[] }` | Auto-save setiap perubahan store/qty |
| `hc_history` | `[{ id, storeName, date, dateFmt, items, text, ... }]` | Max 50 entry, newest first |

### 4.4 Modals & Overlays

| Element | Type | Z-index | Keterangan |
|---|---|---|---|
| Toast | Fixed, top-center | 50 | Auto-dismiss 2.5s, checkmark icon |
| Result Modal | `<dialog>` | Native | Preview teks WA (terminal style, monospace) |
| Info Modal | `<dialog>` | Native | Profil + changelog + link Telegram/Trakteer |
| Checkout Bar | Fixed, bottom-14 | 30 | Di belakang nav bar (z-40) |
| Bottom Nav | Fixed, bottom-0 | 40 | Di depan checkout bar |
| Install Pill | In header | — | Slide-in dari kanan, shadow |

---

## 5. Data Flow

```
                  ┌──────────────┐
                  │   User Tap   │
                  └──────┬───────┘
                         │
              ┌──────────▼──────────┐
              │   Alpine Method     │
              │ (adjust/generate/   │
              │  toggleGroup/clear) │
              └──────────┬──────────┘
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
    ┌──────────┐  ┌────────────┐  ┌──────────┐
    │  State   │  │localStorage│  │   DOM    │
    │ (Proxy)  │  │ (persist)  │  │ (render) │
    └────┬─────┘  └────────────┘  └──────────┘
         │
    ┌────▼─────┐
    │ Reactive │  Alpine Proxy deteksi perubahan,
    │ Re-render│  update hanya binding yang terpengaruh
    └──────────┘
```

Alpine.js menggunakan JavaScript `Proxy` — tidak ada Virtual DOM, tidak ada diffing. Setiap perubahan state langsung memicu re-render binding DOM yang terkait (`x-text`, `x-show`, `:class`, dsb).

---

## 6. Detail Interaksi yang Mungkin Tidak Terlihat

| Interaksi | Detail |
|---|---|
| **Haptic feedback** | `navigator.vibrate(10)` di tap stepper ±, `vibrate(20)` saat copy sukses |
| **Auto-save** | Setiap perubahan `store` atau `items.qty` langsung persist via `save()` |
| **Forced reactivity** | `toggleGroup()` pakai spread `{ ...this.collapsedGroups }` — Proxy tidak mendeteksi `delete` |
| **Deadline awareness** | Checkout bar threshold 6 dus — bar berubah dari abu-abu ke gelap |
| **Discontinued items** | Tetap di katalog (grayscale, no stepper) — awareness tanpa menghapus produk |
| **Format Rupiah** | Regex `/\B(?=(\d{3})+(?!\d))/g` — thousand separator native JS, tanpa library |
| **Race condition PWA** | Visit 1: native prompt lolos (script belum load) • Visit 2+: custom pill (SW cache) |
| **No layout shift** | Subtitel produk di `h-5` fixed height + inline; checkout bar pure translateX |
| **Fold indicator** | "X produk dipesan" hanya muncul saat folded + qty>0 — progressive disclosure |
| **Plugin load order** | Collapse plugin SEBELUM Alpine core — sesuai requirement docs Alpine |

---

## 7. Tech Stack & Dependencies

| Layer | Teknologi | Load |
|---|---|---|
| Reactivity | Alpine.js 3.x | CDN (defer) |
| Animation (fold) | @alpinejs/collapse | CDN (defer, before Alpine) |
| Styling | Tailwind CSS | CDN (sync, blocking head) |
| Offline | Service Worker | `sw.js` (registered inline) |
| Install | Web App Manifest | `manifest.json` (link rel in head) |
| Hosting | Netlify | Static deploy, auto from `main` |
| Build | — | No build step |
| Package manager | — | No dependencies |

---

_Ditulis 10-07-2026 — referensi untuk pengembangan dan onboarding._
