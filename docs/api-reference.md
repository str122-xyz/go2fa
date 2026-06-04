# API Reference — E-Money 2FA Backend

Base URL: `http://localhost:8080/v1`

> Semua endpoint bertanda ✅ wajib menggunakan header:  
> `Authorization: Bearer {{BACKEND_TOKEN}}`

---

## Step 1 — GET /v1/auth/me

**Deskripsi:** Mengambil data profil user yang sedang login.

**Headers:**
| Key | Value |
|-----|-------|
| Authorization | `Bearer {{BACKEND_TOKEN}}` |

**Response 200 OK:**
```json
{
  "success": true,
  "data": {
    "id": 1,
    "firebase_uid": "abc123def456",
    "email": "user@example.com",
    "name": "John Doe",
    "role": "user",
    "email_verified": true,
    "totp_enabled": false,
    "created_at": "2026-05-21T10:00:00Z"
  }
}
```

**Response Error 400:**
```json
{
  "success": false,
  "message": "Token tidak valid atau kadaluarsa"
}
```

---

## Step 2 — PUT /v1/auth/fcm-token

**Deskripsi:** Menyimpan atau memperbarui FCM device token.  
Wajib dipanggil sebelum menggunakan OTP via Firebase notification.

**Headers:**
| Key | Value |
|-----|-------|
| Content-Type | `application/json` |
| Authorization | `Bearer {{BACKEND_TOKEN}}` |

**Request Body:**
```json
{
  "fcm_token": "cFcKwzP4S8u6IJ9k3l..."
}
```

**Response 200 OK:**
```json
{
  "success": true,
  "message": "FCM token berhasil diupdate"
}
```

**Response Error 400:**
```json
{
  "success": false,
  "message": "fcm_token diperlukan"
}
```

---

## Step 3 — POST /v1/otp/send-firebase

**Deskripsi:** Mengirim kode OTP 6 digit ke device user via Firebase Cloud Messaging.  
OTP berlaku selama **5 menit**. Prasyarat: FCM token sudah didaftarkan (Step 2).

**Headers:**
| Key | Value |
|-----|-------|
| Authorization | `Bearer {{BACKEND_TOKEN}}` |

**Response 200 OK:**
```json
{
  "success": true,
  "message": "OTP berhasil dikirim via notifikasi Firebase",
  "data": {
    "otp_type": "firebase",
    "expires_in": 300
  }
}
```

---

## Step 4 — POST /v1/otp/confirm

**Deskripsi:** Verifikasi kode OTP dari Firebase notification atau Email.  
Setelah berhasil, kode langsung dihapus (tidak bisa dipakai ulang).

**Headers:**
| Key | Value |
|-----|-------|
| Content-Type | `application/json` |
| Authorization | `Bearer {{BACKEND_TOKEN}}` |

**Request Body (Firebase OTP):**
```json
{
  "code": "123456",
  "otp_type": "firebase"
}
```

**Request Body (Email OTP):**
```json
{
  "code": "123456",
  "otp_type": "email"
}
```

**Response 200 OK:**
```json
{
  "success": true,
  "message": "OTP berhasil diverifikasi"
}
```

**Response Error 400:**
```json
{
  "success": false,
  "message": "otp_type harus 'firebase' atau 'email'"
}
```

**Response Error 401:**
```json
{
  "success": false,
  "message": "Kode OTP tidak valid atau sudah kadaluarsa",
  "error_code": "INVALID_OTP"
}
```

---

## Step 5 — POST /v1/otp/send-email

**Deskripsi:** Mengirim kode OTP 6 digit ke alamat email terdaftar.  
OTP berlaku selama **5 menit**.

**Headers:**
| Key | Value |
|-----|-------|
| Authorization | `Bearer {{BACKEND_TOKEN}}` |

**Response 200 OK:**
```json
{
  "success": true,
  "message": "OTP berhasil dikirim ke email user@example.com",
  "data": {
    "otp_type": "email",
    "expires_in": 300
  }
}
```

**Response Error 500:**
```json
{
  "success": false,
  "message": "Gagal mengirim OTP via email: SMTP not configured"
}
```

---

## Step 6 — POST /v1/otp/totp/register

**Deskripsi:** Mendaftarkan TOTP untuk user. Mengembalikan secret key dan QR code (base64 PNG) untuk di-scan dengan Google Authenticator.

> Setelah register, user harus melakukan **verify** satu kali untuk mengaktifkan TOTP.

**Headers:**
| Key | Value |
|-----|-------|
| Authorization | `Bearer {{BACKEND_TOKEN}}` |

**Response 200 OK:**
```json
{
  "success": true,
  "message": "TOTP berhasil didaftarkan. Scan QR code dengan Google Authenticator.",
  "data": {
    "secret": "JBSWY3DPEHPK3PXP",
    "qr_code": "data:image/png;base64,iVBORw0KGgoAAAANS...",
    "issuer": "E-Money App",
    "account": "user@example.com",
    "algorithm": "SHA1",
    "digits": 6,
    "period": 30
  }
}
```

**Cara setup di Google Authenticator:**
1. Buka aplikasi Google Authenticator
2. Tap **+** → **Scan a QR Code**
3. Tampilkan nilai `qr_code` (base64 PNG) dan scan
4. Atau tap **Enter a setup key** → masukkan nilai `secret` secara manual
5. Gunakan kode 6 digit yang muncul untuk endpoint verify (Step 7)

---

## Step 7 — POST /v1/otp/totp/verify

**Deskripsi:** Memverifikasi kode TOTP dari Google Authenticator.  
Jika ini verifikasi **pertama kali** setelah register, TOTP akan diaktifkan (`totp_enabled: true`).

**Headers:**
| Key | Value |
|-----|-------|
| Content-Type | `application/json` |
| Authorization | `Bearer {{BACKEND_TOKEN}}` |

**Request Body:**
```json
{
  "code": "123456"
}
```

**Response 200 OK:**
```json
{
  "success": true,
  "message": "TOTP berhasil diverifikasi",
  "data": {
    "totp_enabled": true
  }
}
```

**Response Error 400:**
```json
{
  "success": false,
  "message": "TOTP belum didaftarkan"
}
```

**Response Error 401:**
```json
{
  "success": false,
  "message": "Kode TOTP tidak valid",
  "error_code": "INVALID_TOTP"
}
```

---

## Step 8 — POST /v1/payment/transfer

**Deskripsi:** Melakukan transfer/pembayaran dengan **verifikasi OTP wajib**.  
Mendukung 3 jenis OTP: Firebase, Email, atau TOTP Google Authenticator.

**Headers:**
| Key | Value |
|-----|-------|
| Content-Type | `application/json` |
| Authorization | `Bearer {{BACKEND_TOKEN}}` |

**Request Body (OTP Firebase):**
```json
{
  "amount": 50000,
  "description": "Bayar makan siang",
  "otp_code": "123456",
  "otp_type": "firebase"
}
```

**Request Body (OTP Email):**
```json
{
  "amount": 50000,
  "description": "Bayar makan siang",
  "otp_code": "123456",
  "otp_type": "email"
}
```

**Request Body (TOTP):**
```json
{
  "amount": 75000,
  "description": "Beli pulsa",
  "otp_code": "123456",
  "otp_type": "totp"
}
```

**Response 200 OK:**
```json
{
  "success": true,
  "message": "Transfer berhasil",
  "data": {
    "transaction_id": 3,
    "amount": 75000.00,
    "description": "Beli pulsa",
    "balance_before": 100000.00,
    "balance_after": 25000.00,
    "created_at": "2026-05-21T11:00:00Z"
  }
}
```

**Response Error 400:**
```json
{
  "success": false,
  "message": "Semua field diperlukan: amount, otp_code, otp_type"
}
```

**Response Error 401:**
```json
{
  "success": false,
  "message": "OTP tidak valid atau sudah kadaluarsa",
  "error_code": "INVALID_OTP"
}
```

---

## Endpoint Tambahan

### GET /v1/health
Health check server. Tidak memerlukan auth.

### POST /v1/auth/verify-token
Verifikasi Firebase ID Token dan dapatkan JWT Backend.

**Request Body:**
```json
{
  "firebase_token": "eyJhbGci..."
}
```

### GET /v1/account
Mengambil informasi saldo akun.

### GET /v1/account/transactions
Mengambil riwayat transaksi.

### POST /v1/payment/topup
Top up saldo akun.