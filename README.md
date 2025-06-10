# Deploy WordPress dengan Docker, Nginx, dan SSL (Let's Encrypt)

Ini adalah sebuah boilerplate untuk melakukan deployment aplikasi WordPress secara mudah menggunakan Docker Compose. Proyek ini sudah mencakup Nginx sebagai _reverse proxy_, database MySQL, phpMyAdmin, dan konfigurasi SSL otomatis menggunakan Certbot (Let's Encrypt).

## Fitur Utama

- **WordPress**: Sistem manajemen konten (CMS) yang siap digunakan.
- **Nginx**: Berfungsi sebagai _reverse proxy_ untuk WordPress dan menangani terminasi SSL.
- **MySQL**: Database untuk instalasi WordPress.
- **phpMyAdmin**: Alat bantu untuk mengelola database MySQL melalui antarmuka web.
- **Certbot**: Untuk membuat dan memperbarui sertifikat SSL dari Let's Encrypt secara otomatis.

## Struktur Direktori

```
.
├── certbot/
│   ├── conf/
│   └── www/
├── nginx/
│   ├── nginx.conf          # Digunakan untuk inisialisasi SSL
│   └── nginx-ssl.conf      # Konfigurasi final dengan SSL
├── .env                    # File konfigurasi (dibuat manual)
├── docker-compose.yml      # Definisi semua service docker
└── README.md
```

## Prasyarat

Sebelum memulai, pastikan Anda memiliki:

1.  **Docker** dan **Docker Compose** terinstal di server Anda.
2.  Sebuah **nama domain** yang sudah terdaftar (contoh: `domain.com`).
3.  Server (VPS) dengan **IP publik** yang dapat diakses dari internet.
4.  **DNS record (tipe A)** untuk domain Anda yang sudah diarahkan (`pointing`) ke IP publik server.

## Langkah-langkah Instalasi dan Konfigurasi

Berikut adalah panduan lengkap untuk menjalankan proyek ini dari awal hingga production.

### Langkah 1: Kloning Repositori

```bash
git clone https://github.com/Rullabcde/mini-project-jetor.git
```

### Langkah 2: Konfigurasi Environment

File `.env` digunakan untuk menyimpan semua variabel sensitif seperti password database dan konfigurasi lainnya.

**a. Buat file `.env`:**

```bash
touch .env
```

**b. Isi file `.env` dengan konfigurasi berikut dan sesuaikan.**

```env
# MySQL
MYSQL_ROOT_PASSWORD=your_root_password
MYSQL_DATABASE=wordpress_db
MYSQL_USER=wordpress_user
MYSQL_PASSWORD=your_password

# WordPress
WORDPRESS_DB_HOST=mysql:3306
WORDPRESS_DB_NAME=wordpress_db
WORDPRESS_DB_USER=wordpress_user
WORDPRESS_DB_PASSWORD=your_password

# phpMyAdmin
PMA_HOST=mysql
PMA_PORT=3306
PMA_ABSOLUTE_URI=https://domain.com/phpmyadmin
```

### Langkah 3: Generate Sertifikat SSL (Let's Encrypt)

Proses ini memerlukan Nginx untuk berjalan sementara di port 80 untuk validasi domain oleh Certbot.

**a. Persiapan Direktori & Konfigurasi Nginx Awal**

Buat direktori yang dibutuhkan oleh Certbot dan Nginx.

```bash
mkdir -p ./certbot/conf ./certbot/www ./nginx
```

Salin konfigurasi Nginx awal (hanya HTTP) dari file template yang tersedia di repositori.

```bash
# Salin konten dari `nginx/nginx.conf` yang ada di repositori ini
cp nginx/nginx.conf nginx/nginx.conf
```

**b. Jalankan Service untuk Validasi Domain**

Jalankan hanya service yang diperlukan untuk proses validasi Certbot.

```bash
# Jalankan service tanpa SSL (hanya Nginx, MySQL, WordPress, phpMyAdmin)
docker-compose up -d mysql wordpress phpmyadmin nginx
```

**c. Buat Sertifikat SSL**

Jalankan service `certbot` untuk meminta sertifikat SSL dari Let's Encrypt.

> **Penting:**
>
> - Pastikan Anda sudah mengubah `email@gmail.com` dan `domain.com` di dalam file `docker-compose.yml` pada service `certbot`.
> - Pastikan domain Anda sudah mengarah ke IP server ini.
> - Pastikan port 80 pada server Anda dapat diakses dari internet.

```bash
# Jalankan service certbot untuk membuat sertifikat
docker-compose run --rm certbot
```

**d. Verifikasi Sertifikat**

Cek apakah sertifikat berhasil dibuat di direktori yang sesuai.

```bash
ls -la ./certbot/conf/live/domain.com/
```

Anda seharusnya melihat file `fullchain.pem` dan `privkey.pem`.

### Langkah 4: Konfigurasi Nginx dengan SSL dan Jalankan Aplikasi

Setelah sertifikat berhasil didapatkan, ganti konfigurasi Nginx untuk menggunakan SSL.

**a. Update Konfigurasi Nginx untuk SSL**

Ganti file `nginx.conf` dengan konfigurasi yang sudah menyertakan SSL.

```bash
# Timpa file konfigurasi Nginx dengan versi SSL
cp nginx/nginx-ssl.conf nginx/nginx.conf
```

**b. Muat Ulang Nginx**

Terapkan konfigurasi baru pada Nginx tanpa perlu me-restart kontainer.

```bash
docker-compose exec nginx nginx -s reload
```

### Langkah 5: Konfigurasi Auto-Renewal Sertifikat

Untuk penggunaan jangka panjang (production), Anda perlu mengaktifkan perpanjangan otomatis sertifikat SSL.

**a. Modifikasi `docker-compose.yml`**

Edit service `certbot` di file `docker-compose.yml` menjadi seperti ini:

```yaml
certbot:
  image: certbot/certbot:latest
  container_name: certbot
  volumes:
    - ./certbot/conf:/etc/letsencrypt
    - ./certbot/www:/var/www/certbot
  # command: certonly --webroot -w /var/www/certbot --email email@gmail.com -d domain.com --agree-tos --no-eff-email # Baris ini dinonaktifkan setelah pembuatan sertifikat pertama
  # Aktifkan entrypoint untuk auto-renewal setiap 12 jam
  entrypoint: /bin/sh -c "trap exit TERM; while :; do certbot renew; sleep 12h & wait $$!; done;"
  networks:
    - wp_network
```

**b. Terapkan Perubahan dan Jalankan Semua Services**

```bash
# Jalankan kembali semua service dengan konfigurasi terbaru
docker-compose up -d
```

**c. Uji Coba Proses Renewal (Opsional)**

Anda bisa melakukan simulasi perpanjangan untuk memastikan konfigurasinya benar.

```bash
docker-compose run --rm certbot renew --dry-run
```

## Verifikasi Setup

Gunakan perintah berikut untuk memastikan semua konfigurasi berjalan dengan baik.

```bash
# Tes redirect dari HTTP ke HTTPS
curl -I http://domain.com

# Tes koneksi HTTPS
curl -I https://domain.com

# Tes sertifikat SSL secara detail
openssl s_client -connect domain.com:443 -servername domain.com

# Cek tanggal kedaluwarsa sertifikat
docker run --rm -v $(pwd)/certbot/conf:/etc/letsencrypt certbot/certbot certificates
```

## Akses Aplikasi

- **WordPress**: `https://domain.com`
- **phpMyAdmin**: `https://domain.com/phpmyadmin`

## Troubleshooting

#### Gagal Membuat Sertifikat SSL

1.  **DNS Propagation**: Pastikan domain sudah benar-benar mengarah ke IP server Anda. Cek dengan `nslookup domain.com`.
2.  **Firewall**: Pastikan firewall di server Anda (misalnya `ufw`) mengizinkan trafik masuk pada port 80 dan 443.
3.  **Port 80 Terpakai**: Pastikan tidak ada aplikasi lain (seperti Apache) yang sedang berjalan dan menggunakan port 80.
4.  **Rate Limit Let's Encrypt**: Jika Anda terlalu sering mencoba, Let's Encrypt mungkin akan memblokir permintaan Anda untuk sementara waktu.

#### Nginx Gagal Berjalan

1.  **Cek Sintaks Konfigurasi**: Jalankan perintah ini untuk memvalidasi file `nginx.conf`.
    ```bash
    docker run --rm -v $(pwd)/nginx/nginx.conf:/etc/nginx/nginx.conf nginx nginx -t
    ```
2.  **Cek Log Nginx**: Lihat log untuk pesan error yang lebih spesifik.
    ```bash
    docker-compose logs nginx
    ```
