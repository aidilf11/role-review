# User Role Review

Aplikasi web untuk otomatisasi proses review & konfirmasi kesesuaian role karyawan pada aplikasi **LOS** dan **LMS**.

---

## Struktur Project

```
role-review/
├── backend/                  # Node.js + Express API
│   ├── src/
│   │   ├── db/
│   │   │   ├── config.js     # Koneksi SQL Server
│   │   │   └── schema.sql    # DDL - jalankan di SSMS dulu
│   │   ├── middleware/
│   │   │   └── auth.js       # JWT middleware (admin)
│   │   ├── routes/
│   │   │   ├── auth.js       # POST /api/auth/login
│   │   │   ├── periods.js    # CRUD periode review
│   │   │   ├── sync.js       # Sync data dari LOS/LMS
│   │   │   ├── confirm.js    # Verifikasi & submit konfirmasi (public)
│   │   │   ├── dashboard.js  # Progress, detail, export, reset
│   │   │   ├── email.js      # Blast & reminder email
│   │   │   ├── branchPic.js  # Kelola PIC cabang & reminder per cabang
│   │   │   └── pic.js        # Akses & submit konfirmasi oleh PIC (public, via token)
│   │   ├── services/
│   │   │   └── emailService.js
│   │   └── server.js         # Entry point
│   ├── .env.example          # Copy ke .env dan isi
│   └── package.json
│
└── frontend/                 # React + Vite
    ├── src/
    │   ├── pages/
    │   │   ├── employee/
    │   │   │   ├── ReviewVerify.jsx    # Input username
    │   │   │   ├── ReviewConfirm.jsx   # Form konfirmasi role
    │   │   │   └── ReviewSuccess.jsx   # Halaman sukses
    │   │   ├── admin/
    │   │   │   ├── AdminLogin.jsx
    │   │   │   ├── AdminLayout.jsx
    │   │   │   ├── AdminDashboard.jsx  # Monitor progress
    │   │   │   ├── AdminPeriods.jsx    # Kelola periode
    │   │   │   └── AdminBranchPic.jsx  # Kelola PIC cabang
    │   │   └── pic/
    │   │       └── PicReview.jsx       # Form konfirmasi untuk PIC (akses via token)
    │   ├── context/AuthContext.jsx
    │   ├── services/api.js
    │   ├── App.jsx
    │   ├── main.jsx
    │   └── index.css
    ├── index.html
    ├── vite.config.js
    └── package.json
```

---

## Setup & Cara Menjalankan

### 1. Persiapan Database

Buka **SSMS**, jalankan script DDL:

```sql
-- Jalankan file ini di SSMS:
role-review/backend/src/db/schema.sql
```

Setelah itu, buat admin user pertama (jalankan di SSMS):

```sql
USE RoleReviewDB;
-- Password: admin123 (hash SHA256)
INSERT INTO admin_user (username, password_hash, full_name)
VALUES ('admin', '240be518fabd2724ddb6f04eeb1da5967448d7e831c08c8fa822809f74c720a9', 'Administrator');
```

> Untuk generate hash password lain: `node -e "console.log(require('crypto').createHash('sha256').update('passwordanda').digest('hex'))"`

### 2. Backend

```bash
cd role-review/backend
cp .env.example .env
# Edit .env — isi koneksi SQL Server LOS, LMS, dan RoleReviewDB

npm install
npm run dev
# API berjalan di http://localhost:3001
```

### 3. Frontend

```bash
cd role-review/frontend
npm install
npm run dev
# UI berjalan di http://localhost:5173
```

---

## Alur Penggunaan

### Admin

1. Login di `/admin/login`
2. Buka menu **Periode** → buat periode baru
3. Klik **Sync** untuk tarik data dari LOS/LMS
4. Klik **Aktifkan** → status berubah ke ACTIVE, link form otomatis dibuat
5. Buka **Dashboard** → klik **Blast Email** → email dikirim ke group all employee
6. Monitor progress di dashboard, klik **Reminder** jika ada yang belum konfirmasi
7. Setelah deadline → klik **Tutup** → **Export CSV** untuk dokumentasi

### Karyawan

1. Terima email → klik link form
2. Input **username** (sama seperti login LOS/LMS)
3. Periksa daftar role → pilih **SESUAI** atau **TIDAK SESUAI** per role
4. Isi catatan jika ada role yang tidak sesuai
5. Klik **Submit** → selesai (data dikunci, tidak bisa diubah)

### PIC Cabang

1. Terima email reminder yang berisi link khusus `/pic/<token>`
2. Buka link — halaman menampilkan daftar karyawan di cabang yang **belum konfirmasi**
3. Klik nama karyawan → muncul daftar role-nya
4. Pilih **SESUAI** / **TIDAK SESUAI** per role → klik **Simpan Konfirmasi**
5. Konfirmasi tercatat dengan flag `confirmed_as_pic = 1` (sebagai pengganti karyawan yang bersangkutan)

> Token PIC bersifat **unik per cabang** dan di-generate otomatis saat PIC didaftarkan.
> Token dapat di-regenerate dengan menghapus & mendaftarkan ulang PIC cabang.

---

## API Endpoints

| Method | Endpoint | Auth | Keterangan |
|--------|----------|------|------------|
| POST | `/api/auth/login` | — | Login admin |
| POST | `/api/confirm/verify` | — | Verifikasi username karyawan |
| POST | `/api/confirm/submit` | — | Submit konfirmasi role |
| GET | `/api/periods` | Admin | List semua periode |
| POST | `/api/periods` | Admin | Buat periode baru |
| PATCH | `/api/periods/:id/status` | Admin | Ubah status periode |
| DELETE | `/api/periods/:id` | Admin | Hapus periode (DRAFT only) |
| POST | `/api/sync/:periodId` | Admin | Sync data dari LOS/LMS |
| GET | `/api/dashboard/:periodId` | Admin | Summary progress |
| GET | `/api/dashboard/:periodId/detail` | Admin | Detail per user-role |
| GET | `/api/dashboard/:periodId/export` | Admin | Export CSV |
| POST | `/api/dashboard/:periodId/reset-user` | Admin | Reset konfirmasi 1 user |
| POST | `/api/email/blast/:periodId` | Admin | Blast/reminder email |
| GET | `/api/branch-pic` | Admin | List PIC cabang |
| POST | `/api/branch-pic` | Admin | Tambah/update PIC cabang |
| DELETE | `/api/branch-pic/:id` | Admin | Hapus PIC cabang |
| POST | `/api/branch-pic/reminder/:periodId` | Admin | Blast reminder ke semua PIC cabang |
| POST | `/api/branch-pic/reminder-one` | Admin | Reminder ke 1 PIC cabang |
| GET | `/api/pic/:token` | — | PIC akses data cabang via token |
| GET | `/api/pic/:token/user/:username` | — | Detail roles karyawan untuk dikonfirmasi PIC |
| POST | `/api/pic/:token/submit` | — | PIC submit konfirmasi atas nama karyawan |

---

## Konfigurasi Query LOS/LMS

Sesuaikan query di [`backend/src/routes/sync.js`](backend/src/routes/sync.js) dengan nama tabel aktual di database LOS dan LMS Anda:

```js
const LOS_QUERY = `
  SELECT
    e.EMPLOYEE_NAME AS emp_name,
    u.USERNAME      AS username,
    ...
  FROM USERS u
  JOIN EMPLOYEES e ON ...
  -- Sesuaikan dengan struktur tabel LOS Anda
`;
```

---

## Catatan Keamanan

- Konfirmasi karyawan dilindungi **rate limiting** (max 20 request/15 menit per IP)
- Setelah submit, data dikunci (`is_locked = 1`) — hanya admin yang bisa reset
- Sistem mencatat `ip_address` dan `confirmed_at` untuk audit trail
- Admin dilindungi JWT dengan expiry 8 jam
