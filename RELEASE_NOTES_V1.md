## SmartCook V1 – Release Notes

### Ringkasan

SmartCook V1 adalah aplikasi pendamping dapur yang membantu kamu mencari ide masakan, menyesuaikan resep dengan kebutuhanmu, dan mengatur isi kulkas.  
Aplikasi ini menggabungkan **pencarian resep yang cerdas**, **kulkas digital**, **resep favorit**, dan **asisten chat seputar memasak**, dengan dukungan **mode offline** sehingga banyak konten tetap bisa diakses tanpa koneksi internet.

---

### Fitur Utama Aplikasi

- **Onboarding & Profil Kesehatan Pribadi**
  - Pengaturan profil: nama, rentang usia, dan jenis kelamin.
  - Pengaturan kesehatan: alergi, riwayat penyakit, gaya masak, dan peralatan dapur yang dimiliki.
  - Data ini dipakai untuk memfilter dan mempersonalisasi resep serta saran dari SmartChef Bot.

- **Autentikasi Aman (Email & Google)**
  - Login & registrasi dengan email dan password, plus dukungan login Google.
  - Alur lupa password, reset password via email, dan login via OTP bila terdeteksi aktivitas berisiko.
  - Perubahan email dan password menggunakan verifikasi OTP untuk keamanan tambahan.

- **Beranda yang Dipersonalisasi**
  - Sapaan personal beserta rekomendasi resep sesuai profil dan waktu makan (sarapan, makan siang, makan malam).
  - Seksi “Disimpan untukmu”, resep populer, dan rekomendasi khusus berdasarkan preferensi dan histori penggunaan.
  - Navigasi bawah ke: **Home, Search, SmartChef Bot, Saved, Profile**.

- **Pencarian Resep Cerdas**
  - Cari resep dengan kata kunci atau kalimat bebas, misalnya “menu makan malam simpel dan sehat”.
  - Mode pencarian biasa untuk menjelajah katalog resep yang sudah ada.
  - Menyimpan riwayat pencarian dan beberapa hasil agar bisa dilihat lagi walau sedang offline.

- **Detail Resep yang Kaya Informasi**
  - Tampilan detail resep dengan foto, waktu memasak, dan kalori.
  - Daftar bahan dengan penanda warna mana yang sudah ada di kulkas.
  - Langkah memasak yang terstruktur serta tombol langsung untuk mencari tutorial di YouTube.
  - Tombol bookmark untuk menyimpan resep ke favorit (bekerja juga secara offline dengan sinkronisasi ketika online).

- **Kulkas Digital (Manajemen Bahan)**
  - Halaman “Isi Kulkasmu” untuk melihat, menambah, mengedit, dan menghapus bahan yang dimiliki.
  - Filter menurut kategori (protein, karbo, sayur, bumbu) serta status kedaluwarsa (hari ini, besok, <3 hari, <7 hari, sudah kedaluwarsa).
  - Halaman tambah bahan dengan katalog bahan per kategori, input manual bahan baru, pengaturan jumlah dan tanggal kedaluwarsa.
  - Dari halaman resep, pengguna bisa **menambahkan semua bahan yang belum dimiliki ke kulkas** dalam sekali klik.

- **Favorit & Resep Tersimpan**
  - Menandai resep sebagai favorit untuk disimpan di tab **Saved**.
  - Daftar favorit disinkronkan dengan backend dan sekaligus disimpan lokal untuk akses offline.
  - Pull-to-refresh serta tampilan kosong yang informatif jika belum ada resep tersimpan.

- **SmartChef – Asisten Masak**
  - Fitur chat untuk bertanya seputar ide masak, tips memasak, atau penggantian bahan.
  - Rekomendasi yang muncul bisa langsung dibuka ke halaman detail resep.
  - Riwayat chat dapat dilihat kembali atau dihapus; asisten otomatis nonaktif ketika perangkat offline.

- **Profil & Pengaturan**
  - Melihat dan mengubah nama, email, serta preferensi (alergi, penyakit, gaya masak, peralatan).
  - Mengganti password (via password lama atau OTP email).
  - Logout yang membersihkan token dan kembali ke halaman login.

---

### Pengalaman Offline & UX

- **Offline-first**
  - Data penting seperti resep yang pernah dilihat, favorit, isi kulkas, dan hasil pencarian tertentu disimpan di perangkat.
  - Operasi seperti menambah bahan kulkas, menghapus bahan, atau mengubah favorit dapat diantrikan ketika offline dan akan disinkronkan otomatis saat koneksi kembali.
  - Terdapat indikator/banners ketika perangkat offline, serta snackbar ketika koneksi kembali normal.

- **Kenyamanan Penggunaan**
  - Form dengan validasi jelas dan pesan error yang ramah (misalnya untuk email dan panjang minimal password).
  - Onboarding multi-step dengan indikator langkah dan tombol yang berubah dari “Lanjut” menjadi “Save and Confirm”.
  - Desain visual modern: kartu dengan bayangan, icon tematik, warna kategori berbeda, dan layout yang dioptimalkan untuk berbagai ukuran layar Android.

---

### Catatan Rilis & Keterbatasan

- **Platform**: Aplikasi Android (APK) – fokus utama untuk pengguna berbahasa Indonesia.
- **Koneksi internet**:
  - Disarankan koneksi internet stabil untuk fitur pencarian cerdas, chat SmartChef, dan sinkronisasi data.
  - Sebagian fitur (melihat resep tersimpan, isi kulkas, beberapa hasil pencarian) tetap dapat digunakan saat offline.
- **Rilis perdana (V1)**:
  - Belum tersedia versi iOS.
  - Fitur dan rekomendasi akan terus dikembangkan di rilis berikutnya berdasarkan feedback pengguna.

