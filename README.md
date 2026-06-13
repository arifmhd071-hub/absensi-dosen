# Absensi Dosen — Panduan Deploy (GitHub + Netlify + Firebase + APK)

Paket ini berisi:
```
index.html      -> halaman utama aplikasi (PWA)
manifest.json   -> manifest PWA (dipakai saat install/jadi APK)
sw.js           -> service worker (dukungan offline)
icons/          -> ikon aplikasi (192x192 & 512x512)
netlify.toml    -> konfigurasi deploy Netlify
```

Urutan kerja: **Firebase → GitHub → Netlify → APK**.

---

## 1. Setup Firebase (database & sinkronisasi)

1. Buka https://console.firebase.google.com → **Add project** → beri nama (mis. `absensi-dosen`) → ikuti wizard sampai selesai.
2. Di sidebar kiri, masuk ke **Build > Firestore Database** → **Create database** → pilih lokasi (mis. `asia-southeast2`) → mulai dengan mode **test mode** dulu agar mudah (nanti perketat rules-nya, lihat bagian Keamanan di bawah).
3. Masuk ke **Project settings** (ikon gerigi) → scroll ke **Your apps** → klik ikon **Web (`</>`)** → beri nickname (mis. `absensi-web`) → **Register app**.
4. Firebase akan menampilkan objek `firebaseConfig` seperti ini:
   ```js
   const firebaseConfig = {
     apiKey: "AIza...",
     authDomain: "absensi-dosen.firebaseapp.com",
     projectId: "absensi-dosen",
     storageBucket: "absensi-dosen.appspot.com",
     messagingSenderId: "1234567890",
     appId: "1:1234567890:web:abcdef123456"
   };
   ```
5. Buka file **`index.html`**, cari blok:
   ```js
   window.__FIREBASE_CONFIG__ = {
     apiKey: "",
     authDomain: "",
     projectId: "",
     storageBucket: "",
     messagingSenderId: "",
     appId: ""
   };
   ```
   Ganti nilai kosong tersebut dengan nilai dari `firebaseConfig` Anda. Simpan file.

   > Jika field ini dibiarkan kosong, aplikasi akan tetap berjalan menggunakan `localStorage` (data tersimpan lokal di perangkat saja, tidak tersinkron).

### Keamanan Firestore (PENTING sebelum publik)
Default "test mode" Firestore akan **expired otomatis dalam 30 hari** dan rules-nya terbuka untuk siapa saja. Untuk produksi, minimal gunakan rules berikut di **Firestore > Rules**:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /{document=**} {
      allow read, write: if true; // sementara - ganti dengan auth di production
    }
  }
}
```

Untuk keamanan lebih baik, tambahkan Firebase Authentication (misalnya login admin) dan ubah rules menjadi `allow read, write: if request.auth != null;`. Ini di luar lingkup paket ini, tapi disebutkan agar tidak lupa — aplikasi absensi sebaiknya tidak punya database yang bisa ditulis siapa saja secara publik.

---

## 2. Upload ke GitHub

1. Buat repository baru di https://github.com/new (mis. nama `absensi-dosen`), set ke **Public** (Netlify free tier paling mudah dengan repo public, tapi private juga didukung).
2. Upload semua file dalam paket ini ke repo tersebut. Dua cara:

   **Cara mudah (lewat browser, tanpa Git):**
   - Di halaman repo, klik **Add file > Upload files**.
   - Drag semua file & folder (`index.html`, `manifest.json`, `sw.js`, `icons/`, `netlify.toml`) ke area upload.
   - Klik **Commit changes**.

   **Cara via Git (terminal):**
   ```bash
   git init
   git add .
   git commit -m "Initial commit - Absensi Dosen"
   git branch -M main
   git remote add origin https://github.com/USERNAME/absensi-dosen.git
   git push -u origin main
   ```

---

## 3. Deploy ke Netlify

1. Buka https://app.netlify.com → **Add new site > Import an existing project**.
2. Pilih **GitHub**, beri izin akses, lalu pilih repo `absensi-dosen`.
3. Pada pengaturan build:
   - **Build command**: kosongkan (tidak perlu build, ini static site).
   - **Publish directory**: `.` (root) — `netlify.toml` sudah mengatur ini juga.
4. Klik **Deploy site**. Tunggu beberapa detik, Netlify akan memberi URL seperti `https://nama-acak.netlify.app`.
5. (Opsional) Di **Site settings > Domain management**, ubah subdomain menjadi sesuatu yang lebih rapi, misalnya `absensi-dosen.netlify.app`.
6. **Setiap kali Anda push perubahan ke GitHub, Netlify otomatis re-deploy** — jadi update aplikasi cukup dengan commit ke repo.

> Pastikan setelah deploy Anda buka URL Netlify-nya dan cek indikator status di pojok kiri atas aplikasi: jika menunjukkan **"Tersambung ke Firebase"**, berarti sinkronisasi sudah aktif.

---

## 4. Bungkus jadi APK (Android)

Cara paling mudah dan resmi (didukung Google) adalah **Trusted Web Activity (TWA)** menggunakan **PWABuilder** — tidak perlu install apa pun di komputer.

### Opsi A — PWABuilder (paling mudah, via browser)
1. Buka https://www.pwabuilder.com
2. Masukkan URL Netlify Anda (mis. `https://absensi-dosen.netlify.app`) → klik **Start**.
3. PWABuilder akan mengecek manifest, service worker, dan ikon (semua sudah disiapkan di paket ini). Skor sebaiknya hijau/lengkap.
4. Klik **Package for stores** → pilih **Android**.
5. Isi:
   - **Package ID**: mis. `com.namakampus.absensidosen`
   - **App name**: `Absensi Dosen`
   - Biarkan opsi lain default (signing key bisa dibuat otomatis oleh PWABuilder — simpan file `.keystore`/`.jks` yang dihasilkan, dibutuhkan untuk update aplikasi nanti).
6. Klik **Generate** → unduh paket. Anda akan mendapatkan file **`.apk`** (untuk install langsung/testing) dan **`.aab`** (untuk upload ke Google Play Store).
7. File `.apk` bisa langsung di-transfer ke HP Android dan di-install (aktifkan "Install dari sumber tidak diketahui" di pengaturan Android).

### Opsi B — Bubblewrap CLI (jika butuh kontrol lebih, via terminal)
```bash
npm install -g @bubblewrap/cli
bubblewrap init --manifest https://absensi-dosen.netlify.app/manifest.json
bubblewrap build
```
Ini akan menghasilkan `app-release-signed.apk` di folder project. Membutuhkan Java JDK & Android SDK terinstall (Bubblewrap akan menawarkan untuk mengunduhnya otomatis saat `init`).

### Catatan tentang TWA
- TWA pada dasarnya membuka website Anda dalam tampilan tanpa address bar (full-screen, seperti aplikasi native), menggunakan Chrome di belakangnya. Internet tetap dibutuhkan saat pertama kali load, tapi service worker (`sw.js`) memungkinkan beberapa bagian tetap bisa diakses offline.
- Karena memakai Firestore, fitur sinkronisasi data tetap memerlukan koneksi internet aktif (Firestore juga punya offline cache bawaan untuk data yang sudah pernah dimuat).

---

## 5. Checklist sebelum go-live
- [ ] `firebaseConfig` sudah diisi di `index.html`
- [ ] Firestore Rules sudah diperketat (bukan test mode terbuka)
- [ ] Repo GitHub sudah berisi semua file (`index.html`, `manifest.json`, `sw.js`, `icons/`, `netlify.toml`)
- [ ] Site Netlify sudah live dan indikator "Tersambung ke Firebase" muncul
- [ ] APK sudah digenerate via PWABuilder dan diuji di HP Android
