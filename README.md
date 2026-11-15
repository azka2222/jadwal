# X-A Avengers — Jadwal, Tugas, Uang Kas & Chat

Website kelas X-A SMAN 1 Mlati: jadwal semester 1 TA 2026/2027 (nama guru & jam), tugas per pelajaran, uang kas 36 siswa, chat kelas, login, peran, dan Admin Panel untuk Azka.

## Isi folder
| File | Fungsi |
|---|---|
| `index.html` | Seluruh website |
| `config.js` | Tempat menempel konfigurasi Firebase |
| `firestore.rules` | Aturan keamanan database (isi sama dengan langkah 2) |
| `manifest.webmanifest`, `sw.js`, `icon-192.png`, `icon-512.png` | Agar web bisa dipasang di layar utama HP seperti aplikasi |
| `bg.jpg`, `logo.jpg` | Background (di-blur oleh CSS) dan logo |

**Unggah semua file ini ke repository GitHub yang sama.**

## Dua mode
- **Mode demo lokal** (`config.js` kosong): langsung jalan, tapi akun, tugas, kas, dan chat hanya tersimpan di browser itu sendiri. **Tidak tersambung antar-HP.**
- **Mode Firebase** (`config.js` diisi): semua data tersimpan online dan tersinkron. Gratis (paket Spark) untuk pemakaian sekelas.

---

## Peran dan hak akses
| Peran | Lihat jadwal, chat | Tambah/edit/hapus **tugas** | Menu **Uang Kas** | **Admin Panel** & ubah peran |
|---|:-:|:-:|:-:|:-:|
| 👑 Pemilik (`azka`) | ✅ | ✅ | ✅ | ✅ |
| ⭐ Ketua Kelas | ✅ | ✅ | ✅ | ❌ |
| 📝 Sekretaris | ✅ | ✅ | ❌ | ❌ |
| 💰 Bendahara | ✅ | ❌ | ✅ | ❌ |
| 👤 Anggota | ✅ | ❌ (hanya centang selesai) | ❌ | ❌ |

- **Hanya Azka** yang bisa mengubah peran, lewat tab **Admin** (dropdown peran di samping tiap anggota). Aturan ini dijaga oleh Firestore, bukan hanya tampilan.
- Ketua Kelas juga boleh menghapus pesan chat siapa pun (seperti Azka).
- Tab **Kas** hanya muncul untuk Bendahara, Ketua Kelas, dan Azka.

## Tugas dan peringatan
- Sekretaris / Ketua / Azka menambah tugas di kartu pelajaran. Tugas bisa diedit (✎) dan dihapus (✕).
- **Peringatan:** begitu ada tugas baru, semua yang sedang membuka web langsung melihat kotak kuning "📌 Tugas baru" dengan tombol **Lihat**.
- **Tanda:** kartu pelajaran yang punya tugas diberi garis kuning dan label **📌 N tugas**; tugas yang belum kamu lihat diberi label merah **BARU**; tab hari menampilkan angka jumlah tugas.
- Tombol 🔔 di bagian atas mengaktifkan notifikasi sistem (muncul bila web sedang terbuka atau berjalan di latar).
- Centang "selesai" bersifat pribadi per perangkat.

## Uang Kas (36 siswa)
- Pilih minggu (‹ ›), ketuk tombol **Belum / ✓ Lunas** di tiap siswa. Ada ringkasan saldo, total masuk, total keluar, bar progres, dan filter "Belum bayar".
- **Catat pengeluaran** dengan keterangan dan jumlah; saldo otomatis berkurang.
- **⚙️ Pengaturan:** ubah iuran per minggu (awal Rp 5.000) dan isi **nama 36 siswa** (satu nama per baris, urut absen). Sebelum diisi, nama tampil sebagai "Siswa 1" … "Siswa 36".

## Tetap masuk (tidak perlu login ulang)
Login diingat di perangkat sampai menekan tombol 🚪 Keluar. Web **tidak bisa memakai alamat IP** untuk mengenali pengguna (web statis tidak bisa membaca IP, dan IP HP berganti-ganti serta dipakai bersama banyak orang, sehingga tidak aman). Cara yang dipakai: perangkat mengingat login.
Agar lebih nyaman, pasang di layar utama: **Android/Chrome** → menu ⋮ → *Install app*; **iPhone/Safari** → Bagikan → *Tambahkan ke Layar Utama*.

---

## Langkah 1 — Buat proyek Firebase
1. Buka <https://console.firebase.google.com> → **Add project** → beri nama (mis. `xa-avengers`). Google Analytics boleh dimatikan.
2. **Build → Authentication → Get started → Sign-in method → Email/Password → Enable → Save.**
3. **Build → Firestore Database → Create database** → pilih lokasi (mis. `asia-southeast2` Jakarta) → mode **Production**.

## Langkah 2 — Tempel aturan keamanan (WAJIB diperbarui)
Di **Firestore Database → Rules**, hapus isinya, tempel kode di bawah, lalu **Publish**. Jika sebelumnya sudah memakai aturan lama, **ganti dengan yang ini** — tanpa aturan baru, peran, tugas, dan kas tidak akan berfungsi.

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    function signedIn() { return request.auth != null; }

    // Pemilik web = akun dengan username "azka"
    function isOwner() {
      return signedIn() && request.auth.token.email == 'azka@xa-avengers.example.com';
    }
    function hasProfile() {
      return signedIn() && exists(/databases/$(database)/documents/users/$(request.auth.uid));
    }
    function me() {
      return get(/databases/$(database)/documents/users/$(request.auth.uid)).data;
    }
    function approved() { return isOwner() || (hasProfile() && me().status == 'approved'); }
    function hasRole(roles) { return isOwner() || (approved() && me().role in roles); }

    function canTasks() { return hasRole(['ketua', 'sekretaris']); }
    function canKas()   { return hasRole(['ketua', 'bendahara']); }

    function validText(max) {
      return request.resource.data.text is string
        && request.resource.data.text.size() > 0
        && request.resource.data.text.size() <= max;
    }

    // Profil & peran: HANYA azka yang boleh mengubah peran/status.
    match /users/{id} {
      allow get: if signedIn() && (request.auth.uid == id || isOwner());
      allow list: if isOwner();
      allow create: if isOwner() || (signedIn() && request.auth.uid == id
        && request.resource.data.keys().hasOnly(['username', 'role', 'status', 'createdAt'])
        && request.resource.data.username == request.auth.token.email.split('@')[0]
        && request.resource.data.role == 'member'
        && request.resource.data.status == 'pending');
      allow update, delete: if isOwner();
    }

    // Tugas: dilihat semua anggota; ditambah/diedit/dihapus azka, ketua, sekretaris.
    match /notes/{id} {
      allow read: if approved();
      allow create: if canTasks() && validText(500)
        && request.resource.data.keys().hasOnly(['key', 'text', 'by', 'uid', 'ts'])
        && request.resource.data.uid == request.auth.uid;
      allow update: if canTasks() && validText(500)
        && request.resource.data.diff(resource.data).affectedKeys().hasOnly(['text', 'editedBy', 'editedAt']);
      allow delete: if canTasks();
    }

    // Chat: semua anggota; hapus pesan sendiri, atau azka/ketua menghapus pesan siapa pun.
    match /messages/{id} {
      allow read: if approved();
      allow create: if approved() && validText(500)
        && request.resource.data.uid == request.auth.uid;
      allow delete: if approved() && (resource.data.uid == request.auth.uid || hasRole(['ketua']));
    }

    // Uang kas: hanya azka, ketua, bendahara.
    match /kas/config {
      allow read: if canKas();
      allow create, update: if canKas()
        && request.resource.data.names is list
        && request.resource.data.names.size() == 36
        && request.resource.data.amount is number
        && request.resource.data.amount >= 0
        && request.resource.data.amount <= 1000000;
    }
    match /kasPayments/{id} {
      allow read, delete: if canKas();
      allow create, update: if canKas()
        && request.resource.data.sid is int
        && request.resource.data.sid >= 0 && request.resource.data.sid < 36
        && request.resource.data.week is string && request.resource.data.week.size() == 10
        && request.resource.data.amount is number && request.resource.data.amount > 0;
    }
    match /kasExpenses/{id} {
      allow read, delete: if canKas();
      allow create: if canKas() && validText(100)
        && request.resource.data.amount is number && request.resource.data.amount > 0;
    }
  }
}
```

## Langkah 3 — Ambil konfigurasi web
**Project settings (ikon ⚙️) → General → Your apps** → pada aplikasi web, buka **SDK setup and configuration → Config**, lalu salin enam nilainya (`apiKey`, `authDomain`, `projectId`, `storageBucket`, `messagingSenderId`, `appId`) ke `config.js`. (Halaman *App Check* bukan tempatnya, jangan diaktifkan.)

## Langkah 4 — Pasang di GitHub Pages
1. Buat repository (Public), unggah semua file di folder ini.
2. **Settings → Pages → Branch: `main` / `(root)` → Save.**
3. Tunggu ±1 menit; web aktif di `https://USERNAME.github.io/NAMA-REPO/`.

## Langkah 5 — WAJIB: Azka daftar pertama
Setelah web aktif dan **sebelum link dibagikan**, buka web → tab **Daftar** → buat username **`azka`**. Akun ini otomatis menjadi **pemilik**. Setelah username itu terpakai, tidak ada orang lain yang bisa mengambilnya.

## Cara kerja akses
- Orang lain **Daftar** → *menunggu persetujuan*; belum bisa melihat apa pun. Azka menyetujui/menolak di tab **Admin**.
- Atau Azka membuat **username + password + peran** untuk teman di Admin → **Buat akun** (langsung aktif).
- Azka bisa **memblokir/mengaktifkan** anggota dan mengganti **peran** kapan saja. Perubahan peran langsung berlaku di perangkat anggota itu.
- Setiap anggota bisa ganti password sendiri lewat tombol 🔑.

## Batasan yang perlu diketahui
- **Notifikasi saat web ditutup total** tidak tersedia (itu butuh server notifikasi tersendiri). Namun saat dibuka kembali, tanda 📌 dan **BARU** langsung terlihat. Di iPhone, notifikasi sistem hanya bisa jika web sudah dipasang di Layar Utama (iOS 16.4+).
- **Lupa password**: tidak ada reset lewat email (akun memakai username). Jalan keluarnya: Azka membuat akun baru lewat Admin Panel dan memblokir akun lama.
- Kas dan tugas tidak punya "undo"; hapus dengan hati-hati.
- Username pemilik (`azka`) tertanam di aturan Firestore dan konstanta `OWNER` di `index.html`. Jika ingin username lain, ubah **keduanya**.
- Jadwal berasal dari PDF "Jadwal Guru Mengajar Semester 1 TA 2026/2027". Jika berubah, edit `DAYS` dan `TEACHERS` di awal script `index.html`.
