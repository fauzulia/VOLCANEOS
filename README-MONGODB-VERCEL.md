# Setup MongoDB Atlas untuk VULCANEOS (Vercel)

Panduan ini menjelaskan koneksi MongoDB Atlas untuk **registrasi akun, profil pengguna, dan forum diskusi**, baik saat dijalankan lokal maupun setelah di-deploy ke Vercel.

Gunakan `MONGODB_URI` atau `MONGODB_URL` sebagai environment variable. Jika keduanya disetel, aplikasi memprioritaskan `MONGODB_URI`. Collection dan index dibuat otomatis oleh aplikasi.

---

## 1. Siapkan cluster di MongoDB Atlas

1. Masuk ke [cloud.mongodb.com](https://cloud.mongodb.com).
2. Buat project, lalu klik **Build a Database**.
3. Pilih **M0 Free**, provider **AWS**, region **Singapore** (paling dekat ke Indonesia).
4. Tunggu sampai cluster selesai dibuat (beberapa menit).

### Buat user database

1. Buka **Security → Database Access**.
2. Klik **Add New Database User**.
3. Pilih **Password** sebagai metode autentikasi.
4. Isi username, misalnya `forum-user`, dan simpan passwordnya.
5. Pilih role **Read and write** yang dibatasi ke database `vulcaneos`.
6. Klik **Add User**.

Catatan penting: ini **bukan** akun login Atlas Anda. Login Atlas (termasuk lewat Google) hanya untuk membuka dashboard; aplikasi tidak memakai akun Google atau password Google. Kredensial yang dipakai aplikasi selalu username dan password dari **Database Access** seperti langkah di atas. Jika belum ada user di daftar itu, aplikasi tidak akan pernah bisa login ke database.

### Izinkan akses dari Vercel

1. Buka **Security → Network Access**.
2. Klik **Add IP Address**.
3. Izinkan egress IP yang digunakan Vercel. Jika tidak memiliki IP egress statis, `0.0.0.0/0` dapat dipakai, tetapi mengizinkan percobaan koneksi dari semua IP.
4. Batasi hak Database User ke `vulcaneos`, gunakan password kuat, dan hapus akses luas tersebut setelah kebutuhan deployment berakhir atau saat sudah dapat memakai IP statis.

Alasannya: fungsi Vercel memakai IP keluar yang berubah-ubah, sehingga tidak bisa didaftarkan satu per satu. Kalau nanti tersedia static IP (misalnya Vercel Secure Compute), rentangnya bisa dipersempit.

---

## 2. Ambil connection string

1. Buka cluster → **Connect**.
2. Pilih **Drivers** → **Node.js**.
3. Salin connection string. Bentuknya:

```
mongodb+srv://forum-user:PASSWORD@cluster0.xxxxx.mongodb.net/?retryWrites=true&w=majority
```

4. Tambahkan nama database (`vulcaneos`) sebelum tanda tanya, dan **selalu** sertakan `authSource=admin`:

```
mongodb+srv://forum-user:PASSWORD@cluster0.xxxxx.mongodb.net/vulcaneos?retryWrites=true&w=majority&authSource=admin
```

> Penting: user database Atlas dibuat di database `admin`. Jika nama database ditulis di path tanpa `authSource=admin`, driver akan mencoba login ke database tersebut dan hasilnya `Authentication failed`. Alternatifnya, kosongkan path dan set `MONGODB_DB=vulcaneos`; dengan begitu authSource kembali ke `admin`.

Ketentuan:

- Ganti `PASSWORD` dengan password user database, tanpa tanda kurung `< >`.
- Jika password mengandung karakter `@ : / # ? %`, URL-encode dulu:
  `@` → `%40`, `:` → `%3A`, `/` → `%2F`, `#` → `%23`, `?` → `%3F`, `%` → `%25`.
- Nama database boleh diganti; aplikasi juga bisa memakai `MONGODB_DB` untuk memaksanya.

---

## 3. Pasang di Vercel

1. Buka [vercel.com](https://vercel.com) → project VULCANEOS.
2. Masuk **Settings → Environment Variables**.
3. Klik **Add New**:
   - **Key**: `MONGODB_URI` (atau `MONGODB_URL`)
   - **Value**: connection string lengkap dari langkah 2
   - **Environments**: centang **Production** dan **Preview** (tambahkan **Development** bila memakai `vercel dev`)
4. Simpan.
5. Buka **Deployments** → deployment terbaru → menu `⋯` → **Redeploy**.

Environment variable baru hanya terbaca setelah redeploy. Tanpa redeploy, `/api/forum` akan tetap membalas 503.

---

## 4. Jalankan di komputer (lokal)

### Install dependency

```bash
npm install
```

Perintah ini membaca `package.json` dan mengunduh driver `mongodb` secara otomatis (tidak perlu `npm install mongodb` lagi). Jalankan di folder yang sama dengan `package.json`.

Untuk memastikan driver terpasang:

```bash
node -e "console.log(require('mongodb/package.json').version)"
```

Akan menampilkan versi `6.x`. Driver ini membutuhkan **Node.js 16.20 atau lebih baru**; cek dengan `node -v`.

### Siapkan `.env`

Buat atau edit file `.env` di root project (sejajar dengan `server.js`), lalu isi:

```
PORT=3000
HOST=0.0.0.0
AUTH_USERS=admin:passwordkuat
AUTH_ADMIN_USERS=admin
AUTH_SESSION_SECRET=string-acak-minimal-32-karakter
MONGODB_URI=mongodb+srv://forum-user:PASSWORD@cluster0.xxxxx.mongodb.net/vulcaneos?retryWrites=true&w=majority&authSource=admin
```

`AUTH_USERS`, `AUTH_ADMIN_USERS`, dan `AUTH_SESSION_SECRET` dipakai untuk akun admin dan sesi; akun pengguna baru disimpan di MongoDB. Gunakan hanya satu variabel URI atau samakan nilainya. Jangan commit `.env`.

### Jalankan

```bash
npm start
```

Output yang benar:

```
Vulcaneos listening on 0.0.0.0:3000 (local: http://localhost:3000)
```

Buka `http://localhost:3000` (atau `http://127.0.0.1:3000`). Biarkan terminal tetap terbuka selama testing.

---

## 5. Verifikasi

1. Buka `http://localhost:3000/api/forum` atau `https://domain-anda/api/forum`.
   - Berhasil: `{"threads":[]}`
   - Belum aktif: 503 “Forum belum aktif…”
2. Login, buka WebGIS, klik ikon profil, lalu **Edit profil** dan simpan.
3. Buka `register.html`, buat akun, lalu masuk melalui `login.html`. Registrasi membuat akun aktif ber-role user; pengguna harus masuk sendiri setelah mendaftar. Setelah login, user diarahkan ke beranda dan ikon profil tampil. Email disimpan sebagai kontak tetapi tidak diverifikasi.
4. Di section **Forum**, user yang login dapat membuat topik, berkomentar, menyukai diskusi, dan menghapus komentarnya sendiri. Admin dapat menghapus komentar atau diskusi siapa pun. Pengunjung anonim hanya dapat membaca dan berkomentar.
5. Di Atlas, buka **Browse Collections**. Akan muncul database `vulcaneos` dengan collection:
   - `users` → akun dari `AUTH_USERS` dan registrasi mandiri (password disimpan sebagai hash scrypt)
   - `forum_threads` → diskusi beserta balasan di dalam array `replies`

Tidak ada langkah seeding manual. Akun dari `AUTH_USERS` disalin otomatis saat aplikasi pertama kali terhubung; registrasi mandiri langsung membuat akun di collection `users`.

---

## 6. Troubleshooting

| Gejala | Penyebab dan solusi |
| --- | --- |
| 503 “Database belum dikonfigurasi” | `MONGODB_URI` dan `MONGODB_URL` belum terbaca. Cek nama variabel, scope environment, lalu redeploy. |
| `Authentication failed` / `bad auth` | Username atau password salah, memakai password login Atlas/Google (bukan password Database User), user database belum dibuat, atau lupa `authSource=admin` saat menulis nama database di path URI. |
| `Could not connect to any servers` / timeout | IP belum di-allowlist. Tambahkan `0.0.0.0/0` di Network Access. |
| `querySrv ENOTFOUND` | Host cluster salah ketik atau cluster sudah dihapus. |
| `MongoServerError: not authorized` | Pastikan Database User memiliki `readWrite` pada database `vulcaneos`. |
| Driver MongoDB belum terpasang | Jalankan `npm install` di root project. |
| Server langsung berhenti saat `npm start` | `AUTH_USERS` atau `AUTH_SESSION_SECRET` (minimal 32 karakter) belum diisi. |
| Browser “this site can’t be reached” | Server tidak berjalan atau port berbeda. Lihat output terminal dan `PORT` di `.env`. |
| Forum kosong tanpa error | Normal bila belum ada diskusi. Login lalu buat diskusi pertama. |

Log error runtime di Vercel tersedia di **Project → Logs** (atau **Deployments → Functions**). Jangan pernah menempelkan connection string lengkap ke log atau laporan.

---

## 7. Keamanan

- Jangan commit `.env`; file ini sudah dikecualikan oleh `.gitignore`.
- Gunakan user database khusus dengan hak **readWrite**, bukan akun pemilik organisasi.
- `0.0.0.0/0` memudahkan setup, tetapi berarti siapa pun yang memiliki connection string bisa mengakses database. Jaga kerahasiaannya dan rotasi password bila pernah bocor.
- Ganti password user database secara berkala di **Database Access → Edit → Edit Password**, lalu perbarui `MONGODB_URI` di Vercel dan lokal.
- Endpoint forum memakai sesi yang ditandatangani dan memeriksa origin untuk permintaan POST.
