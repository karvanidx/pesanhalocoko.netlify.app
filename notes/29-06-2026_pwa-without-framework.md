# Membangun PWA Installable Tanpa Framework — Pelajaran dari PesanHaloCoko

Catatan pribadi berdasarkan pengalaman membangun fitur install prompt di [PesanHaloCoko](https://pesanhalocoko.netlify.app) — sebuah PWA satu file HTML tanpa build step, tanpa framework (hanya Alpine.js via CDN).

---

## 1. Kenapa PWA Tanpa Framework?

**Konteks proyek:** satu file `index.html` (±550 baris), Alpine.js + Tailwind via CDN, deploy static ke Netlify. Nol build step.

Untuk aplikasi skala kecil yang dikerjakan solo, pendekatan ini valid:
- Tidak butuh bundler (Vite/Webpack)
- Tidak butuh package manager
- Semua dependensi dari CDN
- Struktur proyek tetap _flat_ dan bisa diedit dari text editor apa pun

PWA tidak memerlukan React/Vue/Svelte. **Syarat PWA installable murni standar browser** — manifest, service worker, HTTPS. Framework tidak relevan.

---

## 2. Tiga Syarat Minimal PWA Installable

| Syarat | File | Keterangan |
|---|---|---|
| **Web App Manifest** | `manifest.json` | Minimal: `name`, `icons` (192+512px), `start_url`, `display` |
| **Service Worker** | `sw.js` | Minimal: fetch handler (cache-first), harus di-register dari halaman |
| **HTTPS** | — | Wajib. Netlify/Vercel/GitHub Pages menyediakan gratis |

Tanpa salah satu dari tiga ini, `beforeinstallprompt` **tidak akan pernah fire**.

### 2.1 Anatomi `manifest.json`

```json
{
  "name": "Nama Aplikasi",
  "short_name": "Nama Pendek",
  "description": "Deskripsi (promotional field — enhance dialog)",
  "start_url": "/",
  "scope": "/",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#3A0110",
  "orientation": "portrait",
  "icons": [
    { "src": "icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "icon-512.png", "sizes": "512x512", "type": "image/png", "purpose": "maskable" }
  ]
}
```

Field _promotional_ (`description`, `screenshots`, `categories`) tidak wajib untuk installability, tapi **meningkatkan kualitas dialog install native** Chrome Android — dari mini-infobar kecil jadi dialog besar seperti App Store.

### 2.2 Anatomi Service Worker (`sw.js`)

```javascript
const CACHE_NAME = 'app-cache-v1';
const urlsToCache = ['./', './index.html', './manifest.json'];

self.addEventListener('install', (event) => {
  event.waitUntil(caches.open(CACHE_NAME).then(cache => cache.addAll(urlsToCache)));
  self.skipWaiting();
});

self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request).then(response => response || fetch(event.request))
  );
});

self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then(names =>
      Promise.all(names.filter(n => n !== CACHE_NAME).map(n => caches.delete(n)))
    )
  );
});
```

**Register dari halaman** (jangan lupa — ini yang sering missed):

```html
<script>
  if ('serviceWorker' in navigator) {
    navigator.serviceWorker.register('/sw.js');
  }
</script>
```

---

## 3. `beforeinstallprompt` — Pattern yang Benar

### 3.1 Cara Kerja Event

1. Browser mendeteksi situs memenuhi kriteria installable
2. Browser me-fire event `beforeinstallprompt` di `window`
3. **Kita bisa tangkap event ini untuk menampilkan UI custom**
4. User klik tombol install → kita panggil `deferredPrompt.prompt()`
5. Browser menampilkan dialog install native

### 3.2 Aturan Penting

- Listener **HARUS** di top-level script — **bukan** di dalam callback/inisialisasi framework
- Panggil `e.preventDefault()` untuk **memblokir** native prompt default
- Simpan referensi event (`deferredPrompt`) — hanya bisa dipakai **sekali**
- Setelah `prompt()` dipanggil, null-kan `deferredPrompt`

### 3.3 Pola Final (TypeScript-style pseudocode)

```javascript
// ═══ TOP-LEVEL (bukan di dalam framework init!) ═══
window.__deferredPrompt = null;

window.addEventListener('beforeinstallprompt', (e) => {
  e.preventDefault();                    // Blokir native mini-infobar
  window.__deferredPrompt = e;           // Simpan untuk dipakai nanti
  showCustomInstallButton();             // Tampilkan UI custom kita
});

// ═══ Alpine init (untuk integrasi state) ═══
Alpine.data('app', () => ({
  deferredPrompt: null,
  showInstallPill: false,

  init() {
    // Cek apakah event sudah fire sebelum Alpine mount
    if (window.__deferredPrompt) {
      this.deferredPrompt = window.__deferredPrompt;
      this.showInstallPill = true;
    }
    // Tetap listen untuk event yang fire setelah mount
    window.addEventListener('beforeinstallprompt', (e) => {
      e.preventDefault();
      this.deferredPrompt = e;
      this.showInstallPill = true;
    });
  },

  installApp() {
    if (!this.deferredPrompt) return;
    this.deferredPrompt.prompt();        // Trigger dialog install native
    this.deferredPrompt = null;          // Hanya bisa dipakai sekali
    this.showInstallPill = false;
  }
}));
```

HTML untuk UI custom (contoh pill di header):

```html
<button @click="installApp()" x-show="showInstallPill"
  x-transition:enter="transition ease-out duration-500"
  x-transition:enter-start="opacity-0 -translate-x-4 scale-90"
  x-transition:enter-end="opacity-100 translate-x-0 scale-100"
  class="flex items-center gap-1.5 bg-blue-600 text-white pl-2 pr-3 py-1.5 rounded-full text-xs font-bold">
  <svg><!-- download icon --></svg>
  <span>Pasang</span>
</button>
```

---

## 4. Pola yang Kami Coba (dan Kenapa Gagal)

Ini catatan dari 6+ iterasi push yang gagal sebelum akhirnya berhasil.

### Attempt 1 ❌ — Listener di Dalam Alpine `init()`

```javascript
// ❌ SALAH: terlalu lambat
Alpine.data('app', () => ({
  init() {
    window.addEventListener('beforeinstallprompt', (e) => { ... });
  }
}));
```

**Masalah:** `beforeinstallprompt` bisa fire **sebelum** Alpine selesai mount. Begitu Alpine siap, event sudah lewat — listener kita tidak pernah menangkapnya. Hasil: native prompt muncul (visit 1), atau tidak ada prompt sama sekali.

### Attempt 2 ❌ — Global + Alpine Listener Double

```javascript
// Global listener
window.addEventListener('beforeinstallprompt', (e) => {
  window.__deferredPrompt = e;
});
// Alpine listener (terpisah)
window.addEventListener('beforeinstallprompt', (e) => {
  this.deferredPrompt = e;
  this.showInstallPill = true;
});
```

**Masalah:** Dua listener menangkap event yang sama. State Alpine bisa di-overwrite oleh timing yang tidak terprediksi. Hasil: pill muncul lalu hilang (flashing).

### Attempt 3 ❌ — `appinstalled` Listener

```javascript
window.addEventListener('appinstalled', () => {
  this.showInstallPill = false;
});
```

**Masalah:** `appinstalled` fire saat user sudah pernah install PWA (via native prompt atau manual). Begitu halaman load, event ini bisa fire dan langsung menyembunyikan pill. Hasil: pill muncul <1 detik lalu hilang.

### Final ✅ — Global Top-Level + Alpine Check + No appinstalled

```javascript
// TOP-LEVEL (sebelum Alpine)
window.__deferredPrompt = null;
window.addEventListener('beforeinstallprompt', (e) => {
  e.preventDefault();
  window.__deferredPrompt = e;
  // Sync ke Alpine kalau sudah mounted
  const alpine = document.querySelector('[x-data]').__x?.$data;
  if (alpine) { alpine.deferredPrompt = e; alpine.showInstallPill = true; }
});

// Alpine init
init() {
  if (window.__deferredPrompt) {
    this.deferredPrompt = window.__deferredPrompt;
    this.showInstallPill = true;
  }
  window.addEventListener('beforeinstallprompt', (e) => {
    e.preventDefault();
    this.deferredPrompt = e;
    this.showInstallPill = true;
  });
}
```

**Mengapa berhasil:**
- Global listener menangkap event **bahkan sebelum** Alpine mount
- Alpine init mengecek global state saat pertama kali jalan
- Alpine listener sebagai _fallback_ untuk event yang fire setelah mount
- Tidak ada `appinstalled` — pill tetap muncul setiap visit (sesuai kebutuhan)

---

## 5. Kenapa Visit Pertama Sering Muncul Native Prompt?

```
Visit 1:  Browser deteksi PWA → fire beforeinstallprompt
          Tapi script kita BELUM selesai load/parse
          → e.preventDefault() TIDAK terpanggil
          → Native prompt muncul (default browser behavior)

Visit 2+: Service worker aktif, semua file di-cache
          Script load LEBIH CEPAT dari event browser
          → Listener terpasang sebelum event fire
          → e.preventDefault() berhasil memblokir native
          → Hanya custom pill yang muncul ✓
```

Ini **race condition alami** antara browser dan script load time. Service worker caching di visit kedua dan seterusnya mempercepat load → listener kita menang.

---

## 6. Checklist Debug: "Kenapa Prompt Gak Muncul?"

Gunakan DevTools (desktop atau remote debugging Android):

1. **Application > Manifest** — pastikan semua field terdeteksi, tidak ada error
2. **Application > Service Workers** — pastikan status "activated and running"
3. **Network** — pastikan halaman diakses via `https://` (atau `localhost`)
4. **Console** — cari error terkait manifest atau SW
5. Ketik di Console: `window.matchMedia('(display-mode: standalone)').matches`
   - `true` = user sudah install, prompt tidak akan muncul
6. Coba **clear site data** (Settings > Privacy > Clear browsing data) — kadang Chrome "mengingat" user sudah menolak install prompt
7. Coba dalam **Incognito mode** — mem-bypass cache install state

---

## 7. Referensi

- [web.dev — Installation Prompt](https://web.dev/learn/pwa/installation-prompt)
- [MDN — Trigger Installation Prompt](https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps/How_to/Trigger_install_prompt)
- [web.dev — PWA Installation Criteria](https://web.dev/learn/pwa/installation)
- Kode final: [`index.html`](../index.html), [`sw.js`](../sw.js), [`manifest.json`](../manifest.json)
