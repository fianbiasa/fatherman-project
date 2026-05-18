# Security & Implementation Roadmap
## Fatherman — Unified Commerce & eLearning Platform
**Versi:** 1.0 | **Tanggal:** 18 Mei 2026

---

## BAGIAN A: SECURITY ARCHITECTURE

### A.1 OWASP Top 10 Mitigations

| # | OWASP Risk | Implementasi Mitigasi |
|---|---|---|
| A01 | Broken Access Control | Role-based middleware, policy per resource, ownership check setiap akses konten |
| A02 | Cryptographic Failures | Semua data sensitif di-encrypt at-rest (AES-256), HTTPS wajib, password bcrypt cost 12 |
| A03 | Injection | Eloquent ORM (parameterized query), validasi semua input, no raw SQL dari user input |
| A04 | Insecure Design | Threat modeling sebelum dev, guest checkout tidak expose user data, webhook signature verify |
| A05 | Security Misconfiguration | `.env` tidak pernah di-commit, debug mode OFF di production, Helmet headers |
| A06 | Vulnerable Components | Composer audit + npm audit di CI/CD, dependency updates bulanan |
| A07 | Auth Failures | JWT short-lived (15 min), refresh token HttpOnly cookie, max login attempt 5x lockout 15 min |
| A08 | Data Integrity Failures | Webhook signature verification wajib, idempotency key pada payment |
| A09 | Logging Failures | Semua aksi admin/CS ter-audit, failed login di-log, anomali terdeteksi |
| A10 | SSRF | Whitelist domain untuk HTTP calls dari server, tidak accept URL dari user untuk fetch |

---

### A.2 Authentication & Authorization

#### A.2.1 JWT Strategy
```
Access Token:
- Algorithm: RS256 (asymmetric key pair)
- TTL: 15 menit
- Payload: { sub: user_id, role, email, iat, exp }
- Disimpan: Memory saja (bukan localStorage/sessionStorage)

Refresh Token:
- Format: random 64-byte hex (tidak JWT)
- TTL: 30 hari (7 hari jika "remember_me" = false)
- Disimpan client: HttpOnly Cookie, Secure, SameSite=Strict
- Disimpan server: DB (hashed SHA-256), bisa di-revoke
- Rotasi: setiap kali digunakan, token lama di-revoke
```

#### A.2.2 Password Policy
```
- Minimum 8 karakter
- Wajib: 1 huruf besar + 1 angka + 1 simbol
- Banned: password dari daftar 10.000 password umum (HaveIBeenPwned API check)
- Hash: bcrypt, cost factor 12
- History: tidak bisa pakai 5 password terakhir
- Reset link: expire 1 jam, single-use
```

#### A.2.3 Rate Limiting
```
POST /auth/login          → max 5 req/menit per IP
POST /auth/register       → max 3 req/menit per IP
POST /auth/forgot-password → max 3 req/menit per email
POST /checkout            → max 10 req/menit per IP
POST /payment/callback    → tidak rate-limited (whitelist Duitku IP)
GET  /api/* (public)      → max 60 req/menit per IP
GET  /api/* (auth)        → max 120 req/menit per user
/admin/*                  → max 200 req/menit per user
```

#### A.2.4 Brute Force Protection
```php
// Lockout setelah 5 gagal login dalam 15 menit
// Redis key: "login_attempts:{ip}:{email}"
// Setelah lockout: tampilkan captcha atau tunggu 15 menit
// Notifikasi email ke user jika ada login dari IP baru
```

---

### A.3 Data Security

#### A.3.1 Sensitive Data Handling
| Data | Perlakuan |
|---|---|
| Password | bcrypt hash, tidak pernah di-log |
| API Keys (Duitku, StarSender) | Disimpan di `.env`, encrypt di DB jika multi-tenant |
| Nomor VA/payment | Tidak di-log seluruhnya (mask sebagian) |
| Alamat pengiriman | Enkripsi JSON field di DB (jika GDPR compliance) |
| IP Address login | Di-log untuk security, di-anonymize setelah 90 hari |
| Refresh Token | Di-hash SHA-256 sebelum disimpan di DB |

#### A.3.2 File Upload Security
```
- Validasi MIME type server-side (bukan hanya extension)
- Scan virus dengan ClamAV jika budget memungkinkan
- File disimpan di private S3 bucket (bukan public URL langsung)
- Akses via signed URL dengan TTL pendek (1 jam untuk PDF/video)
- Filename di-generate UUID, bukan pakai nama asli dari user
- Max file size: 50MB untuk dokumen, 500MB untuk video
```

#### A.3.3 SQL Injection Prevention
```php
// SELALU gunakan Eloquent atau Query Builder
// TIDAK PERNAH:
DB::select("SELECT * FROM users WHERE email = '$email'"); // BERBAHAYA

// GUNAKAN:
User::where('email', $email)->first();
DB::select('SELECT * FROM users WHERE email = ?', [$email]);
```

---

### A.4 Payment Security

#### A.4.1 Webhook Verification (Duitku)
```php
public function verifyDuitkuSignature(array $payload): bool
{
    $merchantCode = config('services.duitku.merchant_code');
    $apiKey       = config('services.duitku.api_key');
    $amount       = $payload['amount'];
    $merchantOrderId = $payload['merchantOrderId'];

    $signatureExpected = md5($merchantCode . $amount . $merchantOrderId . $apiKey);

    return hash_equals($signatureExpected, $payload['signature']);
}
```

#### A.4.2 Idempotency
```php
// Setiap payment callback dicek payment_ref-nya di DB
// Jika sudah pernah diproses (is_processed = true), skip
// Ini mencegah double processing jika Duitku kirim webhook 2x
```

#### A.4.3 Payment Amount Validation
```php
// Selalu validasi amount dari webhook == amount di DB
// Jangan percaya amount dari frontend
// Server yang tentukan harga, bukan client
```

---

### A.5 API Security

#### A.5.1 CORS Configuration
```php
// Hanya izinkan origin yang terdaftar
'allowed_origins' => [
    'https://fatherman.id',
    'https://www.fatherman.id',
    'https://member.fatherman.id',
    'https://admin.fatherman.id',
],
// Payment callback webhook dari Duitku: bypass CORS karena server-to-server
```

#### A.5.2 Security Headers (Nginx)
```nginx
add_header X-Frame-Options "SAMEORIGIN";
add_header X-Content-Type-Options "nosniff";
add_header X-XSS-Protection "1; mode=block";
add_header Referrer-Policy "strict-origin-when-cross-origin";
add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'nonce-{NONCE}' https://passport.duitku.com; ...";
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload";
add_header Permissions-Policy "camera=(), microphone=(), geolocation=()";
```

#### A.5.3 Input Validation Rules
```php
// Checkout
'email'    => 'required_without:phone|email:rfc,dns|max:255',
'phone'    => 'required_without:email|regex:/^(08|628)[0-9]{8,12}$/',
'items'    => 'required|array|min:1|max:20',
'items.*.product_id' => 'required|integer|exists:products,id',
'items.*.quantity'   => 'required|integer|min:1|max:100',

// XSS Prevention: semua string di-sanitize dengan htmlspecialchars
// Tidak pernah output raw user content ke HTML tanpa escape
```

---

### A.6 Infrastructure Security

#### A.6.1 Server Hardening
```
- SSH key-only (disable password auth)
- SSH port non-default (ubah dari 22)
- UFW firewall: hanya allow 22, 80, 443
- Fail2ban untuk brute force SSH
- Automatic security updates enabled
- Non-root user untuk menjalankan aplikasi
```

#### A.6.2 Environment Security
```
- Semua secret di .env (tidak di kode)
- .env tidak pernah di-commit ke Git
- Di production: gunakan vault (AWS Secrets Manager / HashiCorp Vault)
- Key rotation setiap 90 hari untuk production keys
- .env.example disediakan tanpa nilai sensitif
```

#### A.6.3 Database Security
```
- DB tidak expose ke public internet (hanya localhost / VPN)
- User DB khusus dengan privilege minimal (SELECT, INSERT, UPDATE, DELETE)
- Backup daily, encrypted, disimpan di lokasi berbeda
- Connection via SSL/TLS
```

---

## BAGIAN B: IMPLEMENTATION ROADMAP

### B.1 Overview Timeline

```
Phase 1 (Minggu 1-3):  Foundation & Auth
Phase 2 (Minggu 4-7):  Core Commerce (Checkout + Payment)
Phase 3 (Minggu 8-10): Admin Dashboard + Notifikasi
Phase 4 (Minggu 11-14): eCourse Platform
Phase 5 (Minggu 15-16): Migrasi, Testing, Launch
Total Estimasi: 16 minggu (~4 bulan)
```

---

### B.2 Phase 1: Foundation & Auth (Minggu 1-3)

**Tujuan:** Setup infrastruktur dasar, auth system, product management

| Task | PIC | Estimasi | Priority |
|---|---|---|---|
| Setup Laravel project + struktur folder | Dev | 1 hari | P0 |
| Konfigurasi database MySQL + Redis | Dev | 0.5 hari | P0 |
| Setup CI/CD GitHub Actions | Dev | 1 hari | P0 |
| Setup Docker Compose (dev environment) | Dev | 1 hari | P0 |
| Migrasi database (semua tabel) | Dev | 2 hari | P0 |
| Auth: register, login, logout, JWT | Dev | 3 hari | P0 |
| Auth: Google OAuth | Dev | 1.5 hari | P1 |
| Auth: forgot/reset password | Dev | 1 hari | P0 |
| Role-based middleware | Dev | 1 hari | P0 |
| Product CRUD (admin) | Dev | 2 hari | P0 |
| Product list & detail API (public) | Dev | 1 hari | P0 |
| Vue.js setup (admin + member frontend) | FE Dev | 2 hari | P0 |
| Halaman login/register/forgot password | FE Dev | 2 hari | P0 |
| Unit test: auth flow | Dev | 2 hari | P1 |

**Deliverable:** Auth sistem berjalan, admin bisa kelola produk

---

### B.3 Phase 2: Core Commerce (Minggu 4-7)

**Tujuan:** Checkout, payment, ownership, ongkir

| Task | PIC | Estimasi | Priority |
|---|---|---|---|
| Ownership check endpoint | Dev | 1 hari | P0 |
| Cek ongkir (Raja Ongkir API) + caching | Dev | 1.5 hari | P0 |
| Checkout endpoint (buat order) | Dev | 3 hari | P0 |
| Integrasi Duitku | Dev | 2 hari | P0 |
| Webhook payment callback + verification | Dev | 2 hari | P0 |
| Grant product access setelah bayar | Dev | 1 hari | P0 |
| Kupon/diskon system | Dev | 2 hari | P1 |
| Cancel expired orders (scheduler) | Dev | 1 hari | P0 |
| Transfer manual + konfirmasi admin | Dev | 1.5 hari | P1 |
| Halaman checkout (frontend) | FE Dev | 4 hari | P0 |
| Halaman order status / thank you | FE Dev | 1.5 hari | P0 |
| Test: checkout flow end-to-end | Dev | 2 hari | P0 |
| Test: payment sandbox Duitku | Dev | 1 hari | P0 |

**Deliverable:** User bisa checkout, bayar, dan akses digital product

---

### B.4 Phase 3: Admin Dashboard + Notifikasi (Minggu 8-10)

**Tujuan:** Admin/CS bisa kelola order, notifikasi aktif

| Task | PIC | Estimasi | Priority |
|---|---|---|---|
| Order list + filter + search API | Dev | 2 hari | P0 |
| Order detail + timeline API | Dev | 1.5 hari | P0 |
| Update status + tracking API | Dev | 1 hari | P0 |
| Follow-up log API | Dev | 1 hari | P0 |
| Integrasi StarSender (WA notifikasi) | Dev | 1.5 hari | P0 |
| Integrasi Mailgun (email notifikasi) | Dev | 1.5 hari | P0 |
| Template notifikasi (WA + email) | Dev | 1 hari | P0 |
| Queue jobs untuk semua notifikasi | Dev | 1 hari | P0 |
| Dashboard summary widget API | Dev | 1 hari | P0 |
| Chart data API | Dev | 1 hari | P1 |
| Export CSV/Excel | Dev | 1 hari | P1 |
| Admin dashboard UI (Vue.js) | FE Dev | 5 hari | P0 |
| Order detail page (UI) | FE Dev | 2 hari | P0 |
| Audit log viewing (admin) | Dev | 1 hari | P1 |

**Deliverable:** Admin/CS bisa operasional penuh, notifikasi otomatis

---

### B.5 Phase 4: eCourse Platform (Minggu 11-14)

**Tujuan:** Member area, kursus, progress tracking, admin eCourse

| Task | PIC | Estimasi | Priority |
|---|---|---|---|
| Course CRUD + modul + lesson (admin) | Dev | 3 hari | P0 |
| Enrollment setelah beli kursus | Dev | 1 hari | P0 |
| Akses lesson (dengan ownership check) | Dev | 2 hari | P0 |
| Progress tracking (watch duration, complete) | Dev | 2 hari | P0 |
| Sequential lesson lock system | Dev | 1 hari | P1 |
| Certificate generation (PDF) | Dev | 2 hari | P1 |
| Admin eCourse progress API | Dev | 2 hari | P0 |
| Analytics konten API | Dev | 1.5 hari | P1 |
| Inactive member detection + bulk notif | Dev | 1.5 hari | P1 |
| Member dashboard UI (Vue.js) | FE Dev | 3 hari | P0 |
| Course player (video + PDF + text) | FE Dev | 4 hari | P0 |
| Admin eCourse dashboard UI | FE Dev | 3 hari | P0 |
| Progress chart per kursus | FE Dev | 1.5 hari | P1 |
| Quiz system (basic) | Dev | 2 hari | P2 |

**Deliverable:** Platform eCourse fully functional

---

### B.6 Phase 5: Migrasi, Testing, Launch (Minggu 15-16)

| Task | PIC | Estimasi | Priority |
|---|---|---|---|
| Script migrasi data aMember | Dev | 2 hari | P0 |
| Test migrasi di staging environment | Dev | 1 hari | P0 |
| User acceptance testing (UAT) | Tim | 3 hari | P0 |
| Bug fixing dari UAT | Dev | 2 hari | P0 |
| Security audit (OWASP checklist) | Dev | 1 hari | P0 |
| Load testing (minimal 100 concurrent user) | Dev | 1 hari | P1 |
| Setup monitoring (Sentry + UptimeRobot) | Dev | 0.5 hari | P0 |
| Backup & disaster recovery test | Dev | 0.5 hari | P0 |
| Dokumentasi teknis final | Dev | 1 hari | P1 |
| Training Admin & CS | PM/Dev | 1 hari | P0 |
| Soft launch (limited users) | Tim | - | P0 |
| Email blast ke member aMember lama | Tim | 0.5 hari | P0 |
| Full launch | Tim | - | P0 |

---

### B.7 Resource Requirements

| Role | Jumlah | Fokus |
|---|---|---|
| Backend Developer | 1-2 orang | Laravel API, integrasi, DB |
| Frontend Developer | 1 orang | Vue.js, admin + member UI |
| Project Manager | 1 orang (part-time) | Koordinasi, UAT |
| QA Tester | 1 orang (part-time, mulai Phase 3) | Testing, UAT |
| Designer (UI/UX) | 1 orang (part-time, awal) | Wireframe, component library |

---

### B.8 Infrastructure Cost Estimation

| Item | Provider | Estimasi Biaya/Bulan |
|---|---|---|
| VPS (4 vCPU, 8GB RAM) | DigitalOcean / Vultr | $40-60 / ~Rp650rb |
| Database backup storage | Backblaze B2 | $5 / ~Rp80rb |
| Email service | Mailgun | $35 / ~Rp560rb (50k email) |
| WA Notifikasi | StarSender | Rp100rb-300rb |
| CDN + DDoS | Cloudflare Free/Pro | $0-20 |
| Payment Gateway | Duitku | 0.7-2% per transaksi |
| Ongkir API | Raja Ongkir Pro | Rp150rb/bulan |
| SSL | Let's Encrypt | Gratis |
| Domain | Namecheap / Cloudflare | $15/tahun |
| Video CDN (jika self-host) | Bunny.net | $0.01/GB/bulan |
| **Total Estimasi** | | **~Rp1.5jt - 2.5jt/bulan** |

---

### B.9 Testing Strategy

#### Unit Tests (PHPUnit)
```
- Auth service: register, login, token refresh, logout
- CheckoutService: ownership check, price calculation, coupon apply
- PaymentService: webhook verification, amount validation
- CourseService: progress calculation, certificate trigger
- NotificationService: template rendering
```

#### Feature Tests (Laravel HTTP Tests)
```
- Full checkout flow (guest + member)
- Payment callback processing
- Role-based access control
- Lesson access with ownership check
```

#### End-to-End Tests (Playwright / Cypress)
```
- Guest checkout → pembayaran → akses digital
- Member login → akses kursus → tandai selesai → sertifikat
- Admin login → kelola order → update status → kirim WA
```

#### Load Testing (k6 / Artillery)
```
Target: 100 concurrent users
Skenario:
  - 50% browse produk
  - 30% checkout process
  - 20% akses member area
Pass criteria: P95 response time < 500ms, error rate < 1%
```

---

### B.10 Definition of Done

Sebuah fitur dianggap DONE jika:
- [ ] Kode sudah di-review (minimal 1 reviewer)
- [ ] Unit test coverage > 80% untuk service layer
- [ ] Feature test untuk happy path dan error cases
- [ ] API dokumentasi diupdate
- [ ] Security checklist dijalankan
- [ ] Performance: endpoint critical < 300ms (P95)
- [ ] Sudah di-deploy ke staging dan ditest
- [ ] UAT sign-off dari stakeholder

---

### B.11 Risk Register

| Risiko | Probabilitas | Dampak | Mitigasi |
|---|---|---|---|
| Integrasi Duitku bermasalah | Sedang | Tinggi | Test sandbox menyeluruh, backup metode manual |
| Migrasi data aMember korup | Rendah | Tinggi | Backup full sebelum migrasi, dry-run di staging |
| WA API kena blokir | Sedang | Sedang | Fallback ke email, dokumentasi backup provider |
| Raja Ongkir down | Sedang | Rendah | Cache agresif, fallback flat rate |
| Scope creep | Tinggi | Sedang | Strict change management, document semua perubahan |
| Developer keluar di tengah project | Rendah | Tinggi | Dokumentasi kode, handover process |
