# Mini Project Deploy Project Wordpress

[](https://opensource.org/licenses/MIT)

## Instalasi dan Konfigurasi

Berikut adalah langkah-langkah untuk menyiapkan dan menjalankan proyek ini di server Anda.

### 1\. Instalasi Certbot

Certbot adalah alat untuk secara otomatis mendapatkan dan memperbarui sertifikat SSL/TLS dari Let's Encrypt.

```bash
sudo apt update
sudo apt install certbot -y
```

### 2\. Penerbitan Sertifikat SSL

Gunakan Certbot untuk menerbitkan sertifikat untuk domain Anda. Metode `dns` dipilih untuk verifikasi kepemilikan domain.

```bash
sudo certbot certonly --manual --preferred-challenges dns -d domain.com
```

> **Penting:** Ganti `domain.com` dengan nama domain Anda. Ikuti instruksi di terminal untuk menambahkan **TXT record** yang diberikan oleh Certbot ke DNS manager domain Anda sebelum melanjutkan.

### 3\. Konfigurasi Nginx

Salin sertifikat yang telah diterbitkan ke direktori konfigurasi Nginx agar dapat digunakan.

**a. Buat direktori SSL:**

```bash
mkdir -p nginx/ssl
```

**b. Salin file sertifikat:**

Anda perlu menyalin `fullchain.pem` dan `privkey.pem` dari direktori Let's Encrypt. Lokasi defaultnya adalah `/etc/letsencrypt/live/domain.com/`.

```bash
sudo cp /etc/letsencrypt/live/domain.com/fullchain.pem ./nginx/ssl/
sudo cp /etc/letsencrypt/live/domain.com/privkey.pem ./nginx/ssl/
```

### 4\. Konfigurasi Environment

Buat file `.env` di direktori utama proyek untuk menyimpan variabel sensitif seperti kredensial database.

**a. Buat file `.env`:**

```bash
touch .env
```

**b. Isi file `.env` dengan konfigurasi berikut:**

```env
# MySQL
MYSQL_ROOT_PASSWORD=
MYSQL_DATABASE=
MYSQL_USER=
MYSQL_PASSWORD=

# Konfigurasi Koneksi WordPress
WORDPRESS_DB_HOST=mysql:3306
WORDPRESS_DB_NAME=${MYSQL_DATABASE}
WORDPRESS_DB_USER=${MYSQL_USER}
WORDPRESS_DB_PASSWORD=${MYSQL_PASSWORD}

# Konfigurasi phpMyAdmin
PMA_HOST=mysql
PMA_PORT=3306
PMA_ABSOLUTE_URI=https://phpmyadmin.domain.com
```

### 5\. Jalankan Aplikasi

Setelah semua konfigurasi selesai, jalankan semua layanan menggunakan Docker Compose dalam mode _detached_ (`-d`).

```bash
docker-compose up -d
```
