# 🚀 Panduan Anti-Lemot: Melindungi Server Web dari Serangan Bot dengan Nginx

Sistem dengan lalu lintas publik seperti OPAC (Online Public Access Catalog) atau portal akademik sering menjadi target empuk mesin otomatis (*bot scraper*). Mereka menyedot data terus-menerus tanpa henti. Jika dibiarkan, bot ini akan membuat *server* kehabisan napas, dan dampaknya: **pengguna asli (mahasiswa atau staf) akan mendapati website yang *loading* berputar-putar hingga *error*.**

Mari kita bahas cara paling efisien untuk memblokir "tamu tak diundang" ini tanpa membebani *server* Anda.

---

## 📖 Memahami Masalah Lewat Analogi Perpustakaan

Bayangkan *server* aplikasi Anda (misalnya SLiMS) adalah sebuah **Gedung Perpustakaan**.

* **Pengunjung Asli (Manusia):** Datang ke meja layanan, meminta dicarikan 1-2 buku, lalu duduk membaca.
* **Pustakawan (PHP & Database):** Staf yang bertugas berlari ke rak buku (Database) untuk mencarikan data yang diminta pengunjung.
* **Bot Scraper:** Ratusan "robot" yang datang bersamaan ke meja layanan, masing-masing meminta 1.000 buku dalam waktu 1 detik.

**❌ Pendekatan yang Salah (Proteksi di Level Aplikasi)**
Banyak yang mencoba menangkal bot dengan memasang *plugin* keamanan di dalam aplikasinya. Ini sama saja membiarkan robot-robot itu masuk ke dalam gedung, berhadapan dengan pustakawan, baru kemudian pustakawan tersebut mengecek KTP mereka satu per satu untuk mengusirnya. Hasilnya? Pustakawan tetap kelelahan (*CPU/RAM melonjak*), dan pengunjung asli tetap harus antre panjang (*High TTFB*).

**✅ Pendekatan yang Benar (Proteksi Nginx Rate Limiting)**
Kita menempatkan **Satpam di Gerbang Depan (Nginx)**. Sebelum masuk gedung, satpam sudah menghitung: *"Satu orang maksimal hanya boleh minta 3 buku per detik. Lebih dari itu, langsung suruh pulang!"* Dengan begini, pustakawan di dalam tetap santai dan bisa melayani mahasiswa dengan cepat.

---

## 🛠️ Cara Memasang "Satpam Nginx" (Langkah demi Langkah)

Pengaturannya sangat sederhana. Kita hanya perlu menambahkan dua blok kode pendek ke dalam konfigurasi Nginx Anda.

### Langkah 1: Buat Aturan Mainnya
Buka file konfigurasi utama Nginx (biasanya di `/etc/nginx/nginx.conf`). Kita akan membuat "buku catatan" untuk si Satpam.

```nginx
http {
    # Tambahkan baris ini di dalam blok http
    limit_req_zone $binary_remote_addr zone=satpam_nginx:10m rate=3r/s;
}
```
**Maksud kode di atas:**
* `zone=satpam_nginx:10m` : Menyediakan ruang memori 10MB untuk mencatat plat nomor (IP Address) pengunjung. Cukup untuk mengingat ratusan ribu IP sekaligus.
* `rate=3r/s` : Kecepatan maksimal adalah **3 halaman per detik** untuk setiap alamat IP.

### Langkah 2: Terapkan Aturan di Area Sensitif (PHP)
Selanjutnya, buka file konfigurasi website Anda (misal: `/etc/nginx/sites-available/web_perpustakaan.conf`). Cari bagian yang mengeksekusi PHP, lalu panggil aturan yang sudah kita buat tadi.

```nginx
location ~ \.php$ {
    # Panggil satpam di sini!
    limit_req zone=satpam_nginx burst=10 nodelay;

    fastcgi_pass unix:/var/run/php/php-fpm.sock;
    include fastcgi.conf;
}
```
**Maksud kode di atas:**
* `burst=10` : Memberikan "toleransi" jika pengunjung manusia sesekali membuka halaman yang cukup berat (misal: memuat fitur *Advanced Search* atau membuka beberapa *tab* sekaligus).
* `nodelay` : Memastikan pengunjung asli yang tidak melanggar aturan tetap dilayani secepat kilat tanpa ditunda.

### Langkah 3: Simpan dan Aktifkan
Periksa apakah ada yang salah ketik, lalu aktifkan aturannya:
```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## ⚖️ Untung vs Rugi

**✅ Keuntungan Langsung:**
1.  **Website Kembali Responsif:** Beban CPU turun drastis, aplikasi terasa ringan kembali bagi pengguna asli.
2.  **Hemat Resource Server:** Anda tidak perlu terburu-buru *upgrade* RAM atau CPU *server* hanya karena serangan bot.

**⚠️ Risiko yang Perlu Diantisipasi:**
1.  **Penggunaan WiFi Publik / Kampus:** Mahasiswa yang menggunakan WiFi kampus biasanya terdeteksi memiliki **satu IP Publik yang sama**. Jika aturan `rate=3r/s` terlalu ketat, mahasiswa asli bisa ikut terblokir (muncul *Error 429 Too Many Requests*).
    * *Solusi:* Jika pengguna Anda kebanyakan mengakses dari jaringan institusi, longgarkan aturannya (misal: ubah menjadi `rate=15r/s` dan `burst=30`).
2.  **Robot Pencari (Google):** *Indexing* oleh Googlebot mungkin sedikit melambat, namun ini wajar dan jauh lebih baik daripada membiarkan *server* mati (*downtime*).

***

Apakah Anda ingin saya buatkan juga contoh *script* (misalnya menggunakan Python) untuk menganalisis *log* Nginx secara otomatis agar kita tahu IP mana saja yang paling sering terkena blokir oleh aturan ini?
