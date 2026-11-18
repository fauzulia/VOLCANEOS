# Deploy VULCANEOS ke Vercel

Proyek ini sudah memakai struktur yang dapat dijalankan di Vercel: halaman dan berkas di root menjadi aset statis, sedangkan endpoint di `api/` menjadi Vercel Functions. `vercel.json` melindungi `/webgis.html` melalui endpoint sesi; hanya akun pada `AUTH_ADMIN_USERS` yang dapat membukanya. Akun biasa tetap dapat memakai peta publik. Berkas rahasia `.env` tidak dimasukkan ke Git; gunakan Vercel Environment Variables.

## 1. Siapkan repository

1. Buat repository GitHub untuk proyek ini.
2. Dari folder proyek, commit dan push seluruh file website, termasuk folder `api/`, `lib/`, `assets/`, dan `JSON/`.
3. Jangan commit `.env`, kredensial, atau token. `.gitignore` sudah mengecualikan `.env`.

## 2. Import ke Vercel

1. Di Vercel, pilih **Add New → Project**, lalu import repository GitHub.
2. Atur **Framework Preset** ke **Other** dan **Root Directory** ke folder proyek ini.
3. Biarkan **Build Command** dan **Output Directory** kosong. Tidak ada proses build frontend yang diperlukan.
4. Sebelum deploy, buka **Environment Variables** dan tambahkan variabel di bawah. Pilih environment **Production** dan **Preview** sesuai kebutuhan.

## 3. Environment Variables

Wajib untuk login admin:

| Nama | Contoh format | Catatan |
| --- | --- | --- |
| `AUTH_USERS` | `id-admin:password-kuat` | Satu atau lebih akun dipisahkan koma: `id1:password1,id2:password2`. Hindari koma pada ID/kata sandi. |
| `AUTH_ADMIN_USERS` | `id-admin` | Daftar ID akun admin, dipisahkan koma; setiap ID harus sama dengan ID pada `AUTH_USERS`. Hanya akun ini yang dapat membuka `/webgis.html` dan mengekspor laporan; akun lain diarahkan ke peta publik. Wajib di production. |
| `AUTH_SESSION_SECRET` | string acak minimal 32 karakter | Buat nilai rahasia baru untuk production; jangan gunakan contoh dari `.env.example`. |

Opsional:

Pada server lokal non-production, jika `AUTH_ADMIN_USERS` tidak diisi, akun pertama pada `AUTH_USERS` otomatis menjadi admin agar pengembangan lokal tetap praktis. Tetapkan variabel ini secara eksplisit di production.

| Nama | Fungsi |
| --- | --- |
| `AUTH_PROFILES` | JSON untuk nama dan instansi profil per ID, misalnya `{"id-admin":{"name":"Nama Admin","institution":"Tim VULCANEOS"}}`. |
| `SESSION_TTL_SECONDS` | Lama sesi normal; default 28800 detik. |
| `REMEMBER_TTL_SECONDS` | Lama sesi jika pengguna memilih ingat perangkat; default 2592000 detik. |
| `MONGODB_URI` atau `MONGODB_URL` | Connection string MongoDB Atlas untuk profil, forum, dan registrasi. Jika keduanya diisi, `MONGODB_URI` diprioritaskan. |
| `CESIUM_ION_TOKEN` | Token Cesium ion untuk terrain/imagery yang memerlukannya. Gunakan token terbatas untuk domain situs. |
| `ADMIN_EMAIL` | Hanya diperlukan untuk endpoint permohonan akses FormSubmit lama; formulir registrasi saat ini membuat akun langsung di MongoDB. |

Registrasi menggunakan endpoint `/api/register`; akun langsung aktif sebagai `user` dan pengguna diarahkan ke halaman login untuk masuk sendiri. Setelah login, user diarahkan ke beranda dan melihat ikon profil; admin tetap diarahkan ke `/webgis.html`. Email disimpan sebagai kontak tetapi tidak diverifikasi lewat email. Akun baru tidak mendapat hak admin. Vercel tidak membaca file `.env` lokal.

## 3a. Database MongoDB untuk profil dan forum

Profil pengguna, registrasi, dan forum diskusi di beranda memakai MongoDB Atlas. Tanpa `MONGODB_URI` maupun `MONGODB_URL`, registrasi/profil/forum tidak aktif.

1. Masuk [cloud.mongodb.com](https://cloud.mongodb.com) → **Create Project** → **Build a Database** → pilih **M0 Free** dengan provider **AWS** dan region **Singapore**.
2. **Database Access** → tambahkan user database khusus aplikasi dengan role `readWrite` hanya pada database `vulcaneos`.
3. **Network Access** → izinkan egress Vercel. Jika tidak tersedia IP statis, `0.0.0.0/0` dapat dipakai dengan risiko jaringan yang lebih luas; gunakan kredensial kuat dan batasi role database.
4. **Connect** → **Drivers** → **Node.js**, salin connection string, lalu tambahkan nama database dan `authSource=admin`, misalnya `.../vulcaneos?retryWrites=true&w=majority&authSource=admin`. `authSource=admin` wajib karena user Atlas dibuat di database `admin`.
5. Di Vercel, buka **Project → Settings → Environment Variables**, tambahkan `MONGODB_URI` atau `MONGODB_URL` untuk Production dan Preview, lalu **Redeploy**.

Collection `users` dan `forum_threads` dibuat otomatis saat endpoint database pertama kali dipanggil, termasuk index unik username dan email; tidak ada langkah migrasi manual. Kata sandi disimpan sebagai hash scrypt, bukan teks biasa. Balasan forum disimpan sebagai array di dalam dokumen diskusi.

Akun pada `AUTH_USERS` disalin otomatis ke collection `users` saat pertama kali terhubung; ID pada `AUTH_ADMIN_USERS` mendapat peran admin. Pengguna dapat mendaftar langsung lewat `register.html`, lalu masuk dari `login.html`. Akun yang mendaftar mandiri selalu ber-role `user`; hanya admin yang dikonfigurasi yang dapat membuka `/webgis.html`. Nama, instansi, dan bio dapat diubah pengguna lewat menu profil di WebGIS.

Di forum, user yang login dapat membuat topik, berkomentar, menyukai diskusi, serta menghapus komentar miliknya sendiri. Admin dapat menghapus komentar maupun diskusi siapa pun. Pengunjung anonim hanya dapat membaca, memberi komentar, dan tidak dapat membuat topik atau memberi suka.

Untuk pengembangan lokal, isi `MONGODB_URI` atau `MONGODB_URL` di `.env` lalu jalankan `npm install` (sekali) dan `npm start`. Perubahan kata sandi pada `AUTH_USERS` akan disinkronkan ke database saat server berikutnya terhubung ulang.

## 4. Deploy dan domain

Setelah variabel tersimpan, tekan **Deploy**. Vercel akan memberi URL preview/production. Setelah deploy pertama, hubungkan domain khusus melalui **Project → Settings → Domains** jika diperlukan. Setiap push ke branch production akan memicu deployment baru; branch lain dapat membuat Preview Deployment.

Halaman publik tersedia di `/` (Tentang), `/peta.html`, `/latar-belakang.html`, `/tujuan.html`, `/game.html`, dan `/misi.html`. Dashboard admin berada di `/webgis.html` dan meminta login. Dataset pada `JSON/` adalah berkas statis yang dimuat langsung oleh peta.

Jika deployment menampilkan error, periksa **Project → Deployments → Build Logs** dan **Functions Logs**. Error API autentikasi biasanya berarti `AUTH_USERS` atau `AUTH_SESSION_SECRET` belum terpasang pada environment deployment yang sedang dibuka.
