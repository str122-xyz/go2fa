# Week 10 — Implementation 2FA on Mobile Apps

> **Mata Kuliah:** KB1154 / Aplikasi Mobile Lanjutan  
> **Dosen:** I Ketut Gunawan, S.KOM, M.T.I  
> **Pertemuan:** 10 | **Bobot:** 3 SKS  
> **Prodi:** Teknik Informatika — Institut Teknologi & Bisnis Bina Sarana Global

---

## Deskripsi

Repository ini merupakan blueprint pembelajaran implementasi **Two-Factor Authentication (2FA)** pada aplikasi mobile. Terdiri dari dua fase:

| Fase | Deskripsi | Status |
|------|-----------|--------|
| **Fase 1** | Memahami Flow Backend + Testing via Postman | ✅ Completed |
| **Fase 2** | Implementasi ke Flutter Mobile App | 🔄 In Progress |

---

## Arsitektur Sistem

```
Flutter App
    │
    │ HTTP Request (JWT Bearer Token)
    ▼
Backend API (Go + Gin)  ←──→  Firebase Auth
    │
    ├──→ MySQL (users, accounts, transactions)
    ├──→ Redis (OTP cache, expiry 5 menit)
    └──→ SMTP Gmail (Email OTP)
```

**Tech Stack Backend:**
- **Language:** Go 1.21
- **Framework:** Gin
- **Database:** MySQL + Redis
- **Auth:** Firebase Authentication
- **Notification:** Firebase Cloud Messaging (FCM)
- **Email:** SMTP Gmail (App Password)

---

## Struktur Project

```
week10-2fa-backend/
├── be-emoney/                    # Source code backend (Go)
│   ├── main.go                   # Entry point
│   ├── config/
│   │   └── config.go             # Konfigurasi environment
│   ├── database/
│   │   ├── mysql.go              # Koneksi MySQL
│   │   ├── redis.go              # Koneksi Redis
│   │   └── firebase.go           # Firebase SDK init
│   ├── handlers/
│   │   ├── auth.go               # Handler autentikasi
│   │   ├── otp.go                # Handler OTP (Firebase, Email, TOTP)
│   │   ├── payment.go            # Handler transfer & topup
│   │   └── health.go             # Health check
│   ├── middleware/
│   │   └── jwt.go                # JWT middleware
│   ├── models/
│   │   ├── user.go               # Model user
│   │   ├── otp.go                # Model OTP
│   │   └── account.go            # Model akun & transaksi
│   ├── routes/
│   │   └── routes.go             # Routing API
│   ├── services/
│   │   ├── otp.go                # Business logic OTP
│   │   ├── email.go              # Service kirim email
│   │   └── jwt.go                # Service JWT
│   ├── postman/
│   │   └── emoney-2fa.postman_collection.json  # Koleksi Postman siap pakai
│   ├── go.mod
│   ├── go.sum
│   └── .env                      # Tidak di-commit (lihat .env.example)
├── docs/
│   ├── setup.md                  # Panduan setup lengkap
│   ├── api-reference.md          # Dokumentasi endpoint API
│   └── testing-results/          # Screenshot hasil testing Postman
├── .env.example                  # Template environment variable
├── .gitignore
└── README.md
```

---

## Prasyarat

Pastikan sudah terinstall sebelum menjalankan project:

| Tools | Keterangan |
|-------|-----------|
| [Go 1.21+](https://golang.org/dl/) | Runtime bahasa Go |
| [MySQL 8+](https://dev.mysql.com/downloads/) | Database utama |
| [Redis](https://redis.io/) | Cache untuk OTP (bisa via Docker) |
| [Postman](https://www.postman.com/downloads/) | Testing API |
| [Docker](https://www.docker.com/) *(opsional)* | Untuk menjalankan Redis dengan mudah |
| Firebase Project | Service account + FCM enabled |
| Gmail Account | Dengan 2-Step Verification aktif |

---

## Cara Menjalankan

### 1. Clone Repository
```bash
git clone https://github.com/str122-xyz/go2fa.git
cd go2fa
```

### 2. Setup Database MySQL
```sql
CREATE DATABASE emoney CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'useremoney'@'%' IDENTIFIED BY 'Password#123';
GRANT ALL PRIVILEGES ON emoney.* TO 'useremoney'@'%';
FLUSH PRIVILEGES;
```

### 3. Jalankan Redis
```bash
# Menggunakan Docker (recommended)
docker run --name redis-server -p 6379:6379 -d redis

# Atau native Windows (Memurai)
redis-server.exe
```

### 4. Setup File `.env`
```bash
cd be-emoney
cp ../.env.example .env
# Edit .env sesuai konfigurasi lu
```

### 5. Letakkan Firebase Service Account
```bash
# Download dari Firebase Console → Project Settings → Service Accounts
# Simpan sebagai:
be-emoney/firebase_service_account.json
```

### 6. Jalankan Backend
```bash
cd be-emoney
go run main.go
```

Server berjalan di `http://localhost:8080`

---

## API Endpoints

| Method | Endpoint | Deskripsi | Auth |
|--------|----------|-----------|------|
| GET | `/v1/health` | Health check | ❌ |
| POST | `/v1/auth/verify-token` | Verifikasi Firebase token | ❌ |
| GET | `/v1/auth/me` | Data profil user login | ✅ JWT |
| PUT | `/v1/auth/fcm-token` | Update FCM device token | ✅ JWT |
| POST | `/v1/otp/send-firebase` | Kirim OTP via FCM notification | ✅ JWT |
| POST | `/v1/otp/send-email` | Kirim OTP via Email | ✅ JWT |
| POST | `/v1/otp/confirm` | Konfirmasi kode OTP | ✅ JWT |
| POST | `/v1/otp/totp/register` | Register Google Authenticator | ✅ JWT |
| POST | `/v1/otp/totp/verify` | Verifikasi kode TOTP | ✅ JWT |
| GET | `/v1/account` | Info saldo akun | ✅ JWT |
| GET | `/v1/account/transactions` | Riwayat transaksi | ✅ JWT |
| POST | `/v1/payment/topup` | Top up saldo | ✅ JWT |
| POST | `/v1/payment/transfer` | Transfer dengan OTP | ✅ JWT |

> Dokumentasi lengkap → [`docs/api-reference.md`](docs/api-reference.md)

---

## Testing dengan Postman

1. Buka Postman → **Import** → pilih file `be-emoney/postman/emoney-2fa.postman_collection.json`
2. Buat Environment baru bernama `Firebase Auth Dev` dengan variabel:

| Variable | Keterangan |
|----------|-----------|
| `FIREBASE_API_KEY` | Web API Key dari Firebase Console |
| `BACKEND_BASE_URL` | `http://localhost:8080/v1` |
| `USER_EMAIL` | Email untuk testing |
| `USER_PASSWORD` | Password untuk testing |

3. Ikuti urutan testing di [`docs/setup.md`](docs/setup.md)

---

## Dokumentasi Tambahan

- [Panduan Setup Lengkap](docs/setup.md)
- [API Reference](docs/api-reference.md)
- [Konsep 2FA & Perbandingan Metode](docs/setup.md#konsep-2fa)
- [Screenshot Testing Endpoint via Postman](docs/testing-results)

---

## Progress Pembelajaran

- [x] Fase 1: Setup environment & jalankan backend
- [x] Fase 1: Testing endpoint via Postman
- [ ] Fase 2: Integrasi Flutter — Firebase Auth
- [ ] Fase 2: Implementasi OTP di Flutter
- [ ] Fase 2: Transfer dengan verifikasi 2FA