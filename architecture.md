# Arsitektur — DeskBrowser (Fase 1)

## 1. Keputusan engine (ADR-001)

| opsi | ekstensi Chrome | iterasi awal | catatan |
| --- | --- | --- | --- |
| **A. GeckoView** (Mozilla) | WebExtensions (kompatibel dengan sebagian besar API `chrome.*`; MV2 + MV3 sebagian) lewat `WebExtensionController` | cepat: dependensi Gradle biasa, build di laptop | engine bukan Chromium; CWS bukan sumber resmi; **ekstensi tak bertanda tangan perlu di-spike** |
| B. Fork Chromium Android (jalur Kiwi) | penuh (kalau patch ekstensi dipertahankan sendiri) | lambat: checkout >100 GB, build di Linux, berjam-jam, patch harus dirawat tiap rilis | cocok sebagai target jangka panjang |
| C. Android WebView | **tidak ada** | tercepat | tidak bisa menjalankan ekstensi; intersepsi request terbatas → tidak memenuhi syarat inti |

**Keputusan:** Track A (GeckoView) untuk Fase 1–4, di belakang antarmuka `BrowserEngine`, sehingga Track B (Chromium) bisa dicolok belakangan tanpa menulis ulang UI. WebView ditolak karena tidak bisa memenuhi syarat ekstensi.

Spike wajib di awal Fase 4 (bukan asumsi): (1) apakah GeckoView rilis mau memasang ekstensi tak bertanda tangan dari penyimpanan app, (2) cakupan API `chrome.*` untuk 10 ekstensi populer, (3) ToS Chrome Web Store untuk unduh CRX dari browser non-Chrome. Jika (1) gagal, "Load unpacked" bergantung pada jalur built-in/dev-signing atau pindah ke Track B.

Catatan lingkungan: sandbox ini tidak bisa mengunduh dari `dl.google.com` / `maven.mozilla.org`, jadi kode ditulis di sini tetapi **dikompilasi di Android Studio di mesin Anda**.

## 2. Pembagian tugas 11 skill

| skill | peran di proyek ini | fase | catatan penyesuaian |
| --- | --- | --- | --- |
| replica-recon | peta layar/alur/fitur dari sumber publik | 1 ✅ | `replica/recon.md`, `features.csv` |
| replica-architect | stack, model data, "API" internal, urutan build | 1 ✅ | dokumen ini; schema SQL → entitas Room (tanpa Postgres) |
| replica-design | token desain tablet + spesifikasi komponen (TabStrip, Omnibox, dll.) | 2 | token → `Theme.kt`; `contrast.py` untuk cek WCAG |
| replica-build | bangun shell → vertical slice → semua layar | 2–3 | UI Compose, bukan Next.js |
| replica-backend | diadaptasi: lapisan data lokal (Room/DataStore), pembaruan filter list, keamanan ekstensi | 3–4 | tanpa Stripe/auth/email; sinkronisasi = fase lanjut |
| replica-test | rencana uji dari alur F01–F08, uji instrumentasi tablet | 2–4 | Playwright → Compose UI test + Espresso + uji keyboard/mouse |
| replica-diff | skor paritas fitur (`parity.py`) + diff layout screenshot | tiap akhir fase | screenshot emulator tablet 1280x800 |
| replica-entrepreneur | riset keluhan pengguna browser tablet → rencana perbaikan | setelah Fase 4 | hanya review publik, tidak ada kutipan palsu |
| replica-brand | nama final, palet, logo, `sweep.py` bersihkan sisa "Brave/Kiwi/Firefox" | sebelum rilis | **wajib**; nama/logo Mozilla, Brave, Kiwi tidak boleh dipakai |
| replica-launch | halaman landing, listing Play Store | rilis | lint `listing.py` |
| replica-deploy | preflight + penandatanganan AAB/APK, Play Console | rilis | Vercel/DNS tidak relevan; keystore dipegang pengguna |

Aturan lintas-skill: tidak menyalin kode/aset Brave, Kiwi, Chrome, atau Firefox. Yang dipakai hanya **dependensi berlisensi** (GeckoView — MPL-2.0; adblock-rust — MPL-2.0; filter list sesuai lisensinya) dengan atribusi di layar "Tentang".

## 3. Struktur proyek
```
deskbrowser/
├─ replica/                 recon, features.csv, architecture.md, (design/, brand, parity, ...)
├─ gradle/libs.versions.toml
├─ settings.gradle.kts · build.gradle.kts
└─ app/
   ├─ build.gradle.kts
   └─ src/main/
      ├─ AndroidManifest.xml
      ├─ assets/shields/     ekstensi built-in "shields" + filter list awal
      └─ kotlin/dev/deskbrowser/
         ├─ engine/          BrowserEngine, EngineSession (abstraksi) + gecko/ (implementasi Fase 3)
         ├─ tabs/            Tab, TabStore (Room), TabController
         ├─ desktop/         DesktopModeManager (UA, viewport), KeyShortcutRouter
         ├─ adblock/         ContentBlocker, FilterListRepository, HttpsUpgrader
         ├─ extensions/      ExtensionManager, CrxParser, ExtensionRegistry, ApiBridge
         ├─ data/            Room DB, DataStore (SiteSetting, History, ...)
         └─ ui/              Compose: TabStrip, Omnibox, ExtensionToolbar, ContextMenu, screens
```

## 4. Dependensi (ringkas; versi pasti di `libs.versions.toml`)
Kotlin 2.x + Compose (Material3, window-size-class), AndroidX Window (posture/multi-window), Room, DataStore, Coroutines/Flow, GeckoView dari `https://maven.mozilla.org/maven2/` (versi diisi dari repo Mozilla), adblock-rust via JNI (Fase 3, modul `:adblock-native` terpisah).

## 5. Extension Loader (Fase 4, antarmuka ditetapkan sekarang)
Alur: sumber (CWS URL / file .crx / .zip / folder) → `CrxParser` (buang header CRX2/CRX3, validasi ZIP) → baca `manifest.json` (v2/v3) → `PermissionReviewer` (tampil di S06) → `ExtensionManager.install()` ke engine → catat di `ExtensionRegistry` (Room) → ikon muncul di `ExtensionToolbar`.
- `ApiBridge`: pemetaan `chrome.*` ↔ API engine; daftar API yang tidak didukung ditampilkan di S04 (tidak diam-diam gagal).
- Popup (`action.default_popup`) dibuka sebagai `PopupWindow` yang dibatasi lebar jendela (aman untuk split-screen).

## 6. Ad-Block Interceptor (Fase 3, antarmuka ditetapkan sekarang)
Jalur utama: ekstensi built-in "shields" memakai `webRequest`/DNR di engine, memanggil `ContentBlocker` native (adblock-rust via JNI) lewat native messaging; keputusan di-cache per (host, tipe resource). Tambahan: ETP bawaan engine, `HttpsUpgrader`, penyetelan anti-fingerprinting lewat preferensi engine, aturan kosmetik (hide element) + penutup banner cookie. Daftar filter diperbarui berkala (WorkManager, ETag). Ada saklar per situs (S07) untuk situs yang rusak.

## 7. Risiko teratas
1. Ekstensi tak bertanda tangan/CRX di GeckoView + kecocokan API Chrome (spike Fase 4).
2. ToS Chrome Web Store untuk pemasangan dari browser non-Chrome.
3. Latensi pemblokiran request lewat jembatan native; filter list merusak situs.
4. Lisensi/atribusi dan merek dagang (wajib `replica-brand` sebelum rilis).

## 8. Urutan build
1. **Fase 2 — vertical slice:** shell tablet (TabStrip + Omnibox + WebContent) dengan engine palsu → engine Gecko → buka URL desktop mode.
2. **Fase 2 — must-have UI:** tab pin/drag/close, ExtensionToolbar kosong, context menu, shortcut keyboard, split-screen.
3. **Fase 3:** desktop UA/viewport, HTTPS upgrade, ad/tracker block, Shields S07.
4. **Fase 4:** CrxParser, S04/S06, ApiBridge, popup, instal CWS.
