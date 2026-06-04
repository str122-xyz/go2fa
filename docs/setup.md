# Panduan Setup Lengkap — Week 10 2FA Backend

## Daftar Isi
1. [Konsep 2FA](#konsep-2fa)
2. [Setup Gmail SMTP](#1-setup-gmail-smtp)
3. [Setup MySQL](#2-setup-mysql)
4. [Setup Redis](#3-setup-redis)
5. [Setup Firebase](#4-setup-firebase)
6. [Setup File .env](#5-setup-file-env)
7. [Menjalankan Backend](#6-menjalankan-backend)
8. [Setup Postman](#7-setup-postman)

---

## Konsep 2FA

Two-Factor Authentication (2FA) mengharuskan verifikasi identitas menggunakan **dua faktor dari kategori berbeda**.

| Faktor | Nama | Contoh |
|--------|------|--------|
| Kategori A | Something You Know | Password, PIN |
| Kategori B | Something You Have | HP (OTP), Authenticator App |
| Kategori C | Something You Are | Sidik jari, Face ID |
| Kategori D | Somewhere You Are | GPS Lokasi |
| Kategori E | Something You Do | Pola mengetik |

> ⚠️ **Password + PIN BUKAN 2FA!** Keduanya sama-sama "Something You Know". 2FA harus dari dua KATEGORI berbeda.

### Perbandingan Metode 2FA

| Metode | Level Keamanan | Keterangan |
|--------|---------------|-----------|
| SMS OTP | ⭐⭐ Rendah | Rentan SIM Swap |
| Email OTP | ⭐⭐ Rendah-Sedang | Gratis, mudah diimplementasi |
| TOTP (Authenticator) | ⭐⭐⭐⭐ Tinggi | Offline, sangat aman |
| Push Auth (Firebase) | ⭐⭐⭐ Sedang-Tinggi | UX bagus |
| Hardware Key (FIDO2) | ⭐⭐⭐⭐⭐ Sangat Tinggi | Anti-phishing |

---

## 1. Setup Gmail SMTP

### Langkah-langkah

1. Buka [myaccount.google.com](https://myaccount.google.com)
2. Masuk ke **Keamanan** → **Verifikasi 2 Langkah** (wajib aktif)
3. Scroll ke bawah → klik **Sandi Aplikasi**
4. Ketik nama aplikasi (misal: `E-Money Backend`) → klik **Buat**
5. Salin **16 karakter** kode yang muncul — ini `SMTP_PASSWORD` kamu
6. Simpan segera, Google tidak akan menampilkannya lagi

### Konfigurasi SMTP
```
SMTP Server : smtp.gmail.com
Port        : 587 (TLS) atau 465 (SSL)
Username    : emailkamu@gmail.com
Password    : 16 digit App Password (BUKAN password Gmail utama)
```

### Test via cURL
```bash
curl --url "smtp://smtp.gmail.com:587" \
  --ssl-reqd \
  --mail-from "dariemail@gmail.com" \
  --mail-rcpt "tujuan@gmail.com" \
  --user "emailkamu@gmail.com:apppassword16digit" \
  -T <(echo -e "From: dariemail@gmail.com\nTo: tujuan@gmail.com\nSubject: Test\n\nHello dari SMTP!")
```

---

## 2. Setup MySQL

### Buat Database & User
```sql
-- Login ke MySQL
mysql -u root -p

-- Buat database
CREATE DATABASE emoney CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- Buat user khusus (jangan pakai root untuk aplikasi)
CREATE USER 'useremoney'@'%' IDENTIFIED BY 'Password#123';

-- Berikan akses
GRANT ALL PRIVILEGES ON emoney.* TO 'useremoney'@'%';
FLUSH PRIVILEGES;

-- Verifikasi
SHOW DATABASES;
USE emoney;
SELECT USER();
```

> ℹ️ Tabel akan dibuat otomatis saat backend pertama kali dijalankan (auto-migrate).

---

## 3. Setup Redis

### Opsi A — Docker (Recommended)
```bash
# Pull image
docker pull redis

# Jalankan container
docker run --name redis-server -p 6379:6379 -d redis

# Test koneksi
docker exec -it redis-server redis-cli ping
# Output: PONG

# Dengan persist data
docker run --name redis-server \
  -p 6379:6379 \
  -v redis-data:/data \
  -d redis redis-server --appendonly yes
```

### Opsi B — Windows Native (Memurai)
```
Download: https://www.memurai.com/get-memurai
atau: https://github.com/tporadowski/redis/releases

# Jalankan
redis-server.exe

# Test di terminal lain
redis-cli.exe
> ping
PONG
```

---

## 4. Setup Firebase

### 4.1 Buat Firebase Project
1. Buka [console.firebase.google.com](https://console.firebase.google.com)
2. Klik **Add Project** → beri nama (misal: `emoney-2fa`)
3. Enable Google Analytics (opsional) → **Create Project**

### 4.2 Enable Authentication
1. Di sidebar kiri → **Authentication** → **Get Started**
2. Tab **Sign-in method** → aktifkan **Email/Password**

### 4.3 Download Service Account Key
1. **Project Settings** (ikon gear) → tab **Service accounts**
2. Klik **Generate new private key** → **Generate Key**
3. Simpan file JSON yang terdownload sebagai:
   ```
   be-emoney/firebase_service_account.json
   ```
   > ⚠️ File ini ada di `.gitignore` — jangan pernah di-commit!

### 4.4 Dapatkan Web API Key
1. **Project Settings** → tab **General**
2. Scroll ke bawah → bagian **Your apps** → **Web App**
3. Salin nilai **apiKey** → ini untuk variabel `FIREBASE_API_KEY` di Postman

### 4.5 Setup FCM (untuk OTP via Notification)
1. **Project Settings** → tab **Cloud Messaging**
2. Pastikan **Firebase Cloud Messaging API** sudah aktif

---

## 5. Setup File .env

```bash
cd be-emoney

# Salin template
cp ../.env.example .env
```

Edit file `.env`:
```env
PORT=8080

DB_HOST=localhost
DB_PORT=3306
DB_USER=useremoney
DB_PASSWORD=Password#123
DB_NAME=emoney

REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=

JWT_SECRET=ganti-dengan-string-acak-yang-panjang

FIREBASE_CREDENTIALS_PATH=firebase_service_account.json

SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=emailkamu@gmail.com
SMTP_PASSWORD=xxxx-xxxx-xxxx-xxxx
SMTP_FROM=emailkamu@gmail.com
SMTP_FROM_NAME=E-Money App

OTP_EXPIRY_MINUTES=5
```

---

## 6. Menjalankan Backend

```bash
cd be-emoney

# Install dependencies
go mod tidy

# Jalankan server
go run main.go
```

### Output yang diharapkan:
```
[GIN-debug] GET    /v1/health
[GIN-debug] POST   /v1/auth/verify-token
[GIN-debug] GET    /v1/auth/me
[GIN-debug] PUT    /v1/auth/fcm-token
[GIN-debug] POST   /v1/otp/send-firebase
[GIN-debug] POST   /v1/otp/send-email
[GIN-debug] POST   /v1/otp/confirm
[GIN-debug] POST   /v1/otp/totp/register
[GIN-debug] POST   /v1/otp/totp/verify
[GIN-debug] GET    /v1/account
[GIN-debug] GET    /v1/account/transactions
[GIN-debug] POST   /v1/payment/topup
[GIN-debug] POST   /v1/payment/transfer
Server running on port 8080
```

---

## 7. Setup Postman

### Import Collection
1. Buka Postman → klik **Import**
2. Pilih file: `be-emoney/postman/emoney-2fa.postman_collection.json`

### Buat Environment
1. **Environments** → **New** → beri nama `Firebase Auth Dev`
2. Tambahkan variabel berikut:

| Variable | Initial Value | Keterangan |
|----------|--------------|-----------|
| `FIREBASE_API_KEY` | `AIzaSyB_xxx...` | Web API Key dari Firebase Console |
| `FIREBASE_ID_TOKEN` | *(kosong)* | Diisi otomatis via Test script |
| `BACKEND_BASE_URL` | `http://localhost:8080/v1` | Base URL backend |
| `BACKEND_TOKEN` | *(kosong)* | JWT dari backend |
| `USER_EMAIL` | `test@example.com` | Email testing |
| `USER_PASSWORD` | `Test@12345` | Password testing |

3. Klik **Save** lalu aktifkan environment ini (pojok kanan atas Postman)

### Urutan Testing (8 Steps)
Ikuti urutan ini sesuai modul — lihat [`api-reference.md`](api-reference.md) untuk detail setiap step.

| Step | Endpoint | Deskripsi |
|------|----------|-----------|
| 1 | `GET /v1/auth/me` | Cek profil user login |
| 2 | `PUT /v1/auth/fcm-token` | Update FCM token |
| 3 | `POST /v1/otp/send-firebase` | Kirim OTP via Firebase notification |
| 4 | `POST /v1/otp/confirm` | Konfirmasi OTP |
| 5 | `POST /v1/otp/send-email` | Kirim OTP via Email |
| 6 | `POST /v1/otp/totp/register` | Register Google Authenticator |
| 7 | `POST /v1/otp/totp/verify` | Verifikasi kode TOTP |
| 8 | `POST /v1/payment/transfer` | Transfer dengan OTP |