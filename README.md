# X-A Avengers — Jadwal, Catatan Tugas & Chat

Website kelas X-A SMAN 1 Mlati: jadwal semester 1 TA 2026/2027 (lengkap dengan nama guru & jam), catatan tugas per pelajaran, chat kelas, login, dan Admin Panel untuk Azka.

## Isi folder
| File | Fungsi |
|---|---|
| `index.html` | Seluruh website |
| `config.js` | Tempat menempel konfigurasi Firebase |
| `bg.jpg`, `logo.jpg` | Background (di-blur oleh CSS) dan logo |

## Dua mode
- **Mode demo lokal** (`config.js` masih kosong): langsung jalan, tapi akun, catatan, dan chat hanya tersimpan di browser itu sendiri. Cocok untuk mencoba tampilan. **Chat tidak tersambung antar-HP.**
- **Mode Firebase** (`config.js` diisi): login, catatan, dan chat tersimpan online sehingga Azka dan teman-teman bisa berkomunikasi dua arah dari HP masing-masing. Gratis di paket Spark untuk pemakaian sekelas.

GitHub Pages hanya bisa menyajikan file statis, jadi untuk chat antar-perangkat dibutuhkan layanan online seperti Firebase.

---

## Langkah 1 — Buat proyek Firebase
1. Buka <https://console.firebase.google.com> → **Add project** → beri nama (mis. `xa-avengers`). Google Analytics boleh dimatikan.
2. **Build → Authentication → Get started → Sign-in method → Email/Password → Enable → Save.**
3. **Build → Firestore Database → Create database** → pilih lokasi (mis. `asia-southeast2` Jakarta) → mode **Production**.

## Langkah 2 — Tempel aturan keamanan
Di **Firestore Database → Rules**, hapus isinya, tempel kode di bawah, lalu **Publish**:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    function signedIn() { return request.auth != null; }

    // Pemilik web = akun dengan username "azka"
    function isOwner() {
      return signedIn() && request.auth.token.email == 'azka@xa-avengers.example.com';
    }

    function approved() {
      return isOwner() || (signedIn()
        && exists(/databases/$(database)/documents/users/$(request.auth.uid))
        && get(/databases/$(database)/documents/users/$(request.auth.uid)).data.status == 'approved');
    }

    function validText() {
      return request.resource.data.text is string
        && request.resource.data.text.size() > 0
        && request.resource.data.text.size() <= 500
        && request.resource.data.uid == request.auth.uid;
    }

    match /users/{id} {
      allow get: if signedIn() && (request.auth.uid == id || isOwner());
      allow list: if isOwner();
      // Pendaftar baru hanya boleh membuat profil berstatus "pending"
      allow create: if isOwner() || (signedIn() && request.auth.uid == id
        && request.resource.data.keys().hasOnly(['username','role','status','createdAt'])
        && request.resource.data.role == 'member'
        && request.resource.data.status == 'pending');
      allow update, delete: if isOwner();
    }

    match /notes/{id} {
      allow read: if approved();
      allow create: if approved() && validText();
      allow delete: if approved() && (resource.data.uid == request.auth.uid || isOwner());
    }

    match /messages/{id} {
      allow read: if approved();
      allow create: if approved() && validText();
      allow delete: if approved() && (resource.data.uid == request.auth.uid || isOwner());
    }
  }
}
```

## Langkah 3 — Ambil konfigurasi web
1. Di Firebase: **Project settings (ikon roda gigi) → General → Your apps → ikon `</>` (Web)** → daftarkan app → salin objek `firebaseConfig`.
2. Buka `config.js` dan isi `apiKey`, `authDomain`, `projectId`, `storageBucket`, `messagingSenderId`, `appId` dengan nilai tersebut.

## Langkah 4 — Pasang di GitHub Pages
1. Buat repository baru (Public), unggah **semua file** di folder ini (`index.html`, `config.js`, `bg.jpg`, `logo.jpg`, `README.md`).
2. **Settings → Pages → Branch: `main` / `(root)` → Save.**
3. Tunggu ±1 menit; web aktif di `https://USERNAME.github.io/NAMA-REPO/`.

## Langkah 5 — WAJIB: Azka daftar pertama
Segera setelah web aktif dan **sebelum link dibagikan**, buka web → tab **Daftar** → buat username **`azka`** dengan password pilihan Azka. Akun dengan username `azka` otomatis menjadi **pemilik** dan memiliki tab **Admin**. Setelah username itu terpakai, tidak ada orang lain yang bisa mengambilnya.

## Cara kerja akses
- Orang lain **Daftar** → statusnya *menunggu persetujuan*; mereka belum bisa melihat catatan/chat. Azka menyetujui atau menolak di tab **Admin**.
- Atau Azka membuat **username + password** untuk teman di Admin → **Buat akun** (langsung aktif), lalu memberikannya ke teman.
- Azka bisa **memblokir** / **mengaktifkan** kembali anggota kapan saja.
- Login tersimpan di perangkat (tidak perlu masuk ulang). Setiap anggota bisa ganti password sendiri lewat tombol 🔑.

## Batasan yang perlu diketahui
- **Lupa password**: tidak ada reset lewat email (akun memakai username, bukan email asli). Jalan keluarnya: buat akun baru dengan username lain lewat Admin Panel, dan blokir akun lama.
- Catatan tugas bersifat **bersama** (terlihat oleh semua anggota yang disetujui); tanda centang "selesai" bersifat pribadi per perangkat.
- Nama pemilik (`azka`) tertanam di aturan Firestore pada langkah 2. Jika ingin username pemilik lain, ubah `azka@xa-avengers.example.com` di aturan **dan** konstanta `OWNER` di `index.html`.
- Jadwal berasal dari PDF "Jadwal Guru Mengajar Semester 1 TA 2026/2027" (berlaku mulai 10 Agustus 2026). Jika jadwal berubah, edit data `DAYS` dan `TEACHERS` di bagian atas script pada `index.html`.
