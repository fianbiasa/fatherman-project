# Technical Architecture Document
## Fatherman — Unified Commerce & eLearning Platform
**Versi:** 1.0 | **Tanggal:** 18 Mei 2026

---

## 1. TECH STACK

### 1.1 Backend
| Layer | Teknologi | Versi | Alasan |
|---|---|---|---|
| Framework | Laravel | 11.x | Mature ecosystem, Sanctum/JWT, Queue built-in |
| Language | PHP | 8.3+ | Typed properties, fibers, performance |
| Database | MySQL | 8.0+ | JSON column, full-text search, window functions |
| Cache | Redis | 7.x | Session, queue, rate limiting, cek ongkir cache |
| Queue | Laravel Horizon | - | Worker management, monitoring antrian |
| Search | Laravel Scout + Meilisearch | - | Full-text search produk & order |
| PDF Generator | DomPDF / Spatie Browsershot | - | Invoice & sertifikat |

### 1.2 Frontend
| Layer | Teknologi | Versi | Alasan |
|---|---|---|---|
| Framework | Vue.js | 3.x | Reaktif, Composition API, ekosistem besar |
| Build Tool | Vite | - | Fast HMR, modern bundling |
| UI Component | PrimeVue / Shadcn-Vue | - | Komponen enterprise-ready |
| State | Pinia | - | Store management ringan |
| HTTP Client | Axios | - | Interceptor untuk token refresh |
| Charts | Chart.js + Vue-Chartjs | - | Dashboard analytics |
| CSS | Tailwind CSS | 3.x | Utility-first, konsisten |

### 1.3 Infrastructure & DevOps
| Komponen | Teknologi | Alasan |
|---|---|---|
| Server | VPS (Ubuntu 22.04) / Laravel Cloud | Fleksibel, cost-effective |
| Web Server | Nginx + PHP-FPM | Performance, reverse proxy |
| SSL | Let's Encrypt (Certbot) | Gratis, auto-renew |
| Container | Docker + Docker Compose | Dev/prod parity |
| CI/CD | GitHub Actions | Automated deploy, test |
| Monitoring | Laravel Telescope (dev) + Sentry (prod) | Error tracking |
| Uptime | UptimeRobot / Better Uptime | Alert downtime |
| Backup | Automated MySQL dump ke S3 / Backblaze B2 | Disaster recovery |
| CDN | Cloudflare | DDoS protection, caching static assets |

### 1.4 Storage
| Tipe | Provider | Konfigurasi |
|---|---|---|
| File upload | S3-compatible (Cloudflare R2 / MinIO) | Murah, S3-compatible API |
| Video hosting | YouTube (unlisted) / Bunny.net | Streaming ter-manage |
| Asset statis | Cloudflare CDN | Cache global |

---

## 2. SYSTEM ARCHITECTURE

### 2.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLOUDFLARE CDN                          │
│              (DDoS Protection + Static Assets Cache)            │
└──────────────────────┬──────────────────────────────────────────┘
                       │
          ┌────────────┼────────────┐
          │            │            │
    ┌─────▼──────┐ ┌───▼────┐ ┌────▼────────────┐
    │ Public     │ │Member  │ │  Admin / CS      │
    │ Web        │ │ Area   │ │  Dashboard       │
    │ (Checkout, │ │(Vue.js)│ │  (Vue.js)        │
    │  Landing)  │ │        │ │                  │
    └─────┬──────┘ └───┬────┘ └────┬────────────┘
          │            │            │
          └─────────── ▼ ───────────┘
                  ┌────────────┐
                  │  NGINX     │
                  │  (Reverse  │
                  │   Proxy)   │
                  └─────┬──────┘
                        │
              ┌─────────▼──────────┐
              │   LARAVEL APP      │
              │   (API + Web)      │
              │                    │
              │  ┌──────────────┐  │
              │  │ Auth Service │  │
              │  │ Order Service│  │
              │  │ Course Svc   │  │
              │  │ Payment Svc  │  │
              │  │ Notif Service│  │
              │  └──────────────┘  │
              └──┬─────┬──────┬───┘
                 │     │      │
          ┌──────▼┐ ┌──▼───┐ ┌▼───────────────┐
          │ MySQL │ │Redis │ │  Queue (Horizon)│
          │  8.0  │ │  7.x │ │                │
          └───────┘ └──────┘ └────────────────┘
                                     │
              ┌──────────────────────┼───────────────────────┐
              │                      │                       │
        ┌─────▼──────┐     ┌─────────▼──────┐     ┌────────▼──────┐
        │ WA Notif   │     │  Email (SMTP)  │     │  Raja Ongkir  │
        │ (Fonnte)   │     │  (Mailgun)     │     │  / Shipper    │
        └────────────┘     └────────────────┘     └───────────────┘

              ┌──────────────────────┐
              │  Midtrans (Payment)  │
              │  Webhook → /api/     │
              │  payment/callback    │
              └──────────────────────┘
```

### 2.2 Request Flow — Checkout

```
Browser ──[HTTPS]──> Cloudflare ──> Nginx ──> Laravel API
    │                                              │
    │ POST /api/checkout                           │
    │                                              ▼
    │                                    Validate Input
    │                                              │
    │                                    Check Ownership
    │                                    (email/phone/user_id)
    │                                              │
    │                                    Create Order (DB)
    │                                              │
    │                                    Request Payment to Midtrans
    │                                              │
    │ ◄── Return { order_id, payment_url, va } ────┘
    │
    │ User Bayar via Bank/QRIS
    │
Midtrans ──[Webhook]──> Nginx ──> Laravel /api/payment/callback
                                              │
                                    Verify Signature
                                              │
                                    Verify Status via Midtrans API
                                              │
                                    Update Order Status
                                              │
                              ┌───────────────┤
                              │               │
                         Grant Access    Dispatch Queue Jobs:
                         (product_       - Send WA notif
                          ownership)     - Send Email notif
                                         - Update Enrollment
```

### 2.3 Request Flow — Member Login & Akses Kursus

```
Login Request
    │
    ▼
POST /api/auth/login
    │
    ├─ Validate email+password OR Google OAuth token
    ├─ Check user status (active?)
    ├─ Generate JWT Access Token (15 min TTL)
    ├─ Generate Refresh Token (30 day, stored di DB)
    └─ Return { access_token, refresh_token, user }

Akses Lesson
    │
    ▼
GET /api/courses/{id}/lessons/{lesson_id}
    │
    ├─ Verify JWT (middleware)
    ├─ Check product_ownership (user punya akses kursus?)
    ├─ Check enrollment (sudah enroll?)
    ├─ Check is_sequential (lesson sebelumnya sudah selesai?)
    ├─ Record member_lesson_progress (status: in_progress)
    └─ Return lesson content

Mark Lesson Complete
    │
    ▼
POST /api/lessons/{id}/complete
    │
    ├─ Update member_lesson_progress (status: completed)
    ├─ Recalculate enrollment progress_percent
    └─ If progress = 100%: Generate certificate (Queue Job)
```

---

## 3. SERVICE LAYER DESIGN

### 3.1 Core Services

```php
// app/Services/

CheckoutService
├── validateCart(array $items): void
├── checkOwnership(string $email, string $phone, array $productIds): array
├── createOrder(CheckoutDTO $dto): Order
├── calculateShipping(string $origin, string $destination, int $weight): array
└── applyeCoupon(Order $order, string $code): Order

PaymentService
├── createPayment(Order $order, string $method): Payment
├── handleCallback(array $payload, string $signature): void
├── verifyPaymentStatus(string $paymentRef): PaymentStatus
├── processRefund(Payment $payment): void
└── reconcile(Carbon $date): ReconciliationReport

NotificationService
├── sendWhatsApp(string $phone, string $template, array $vars): void
├── sendEmail(string $email, string $template, array $vars): void
├── getTemplate(string $key, string $channel): NotificationTemplate
└── queueNotification(NotificationJob $job): void

CourseService
├── enrollUser(User $user, Course $course): Enrollment
├── markLessonComplete(User $user, Lesson $lesson): void
├── calculateProgress(User $user, Course $course): float
├── generateCertificate(User $user, Course $course): Certificate
└── getInactiveMember(int $dayThreshold): Collection

OrderService
├── updateStatus(Order $order, string $status, User $actor): void
├── addFollowupLog(Order $order, User $actor, array $data): void
├── updateTracking(Order $order, string $resi): void
├── cancelExpiredOrders(): void
└── generateReport(Carbon $from, Carbon $to): Report
```

### 3.2 Queue Jobs

```php
// app/Jobs/

SendWANotificationJob         // async - kirim WA via Fonnte
SendEmailNotificationJob      // async - kirim email via Mailgun
GrantProductAccessJob         // after payment - buat product_ownership
EnrollCourseJob               // enroll user ke course setelah beli
GenerateCertificateJob        // generate PDF sertifikat
GenerateInvoiceJob            // generate PDF invoice
CancelExpiredOrdersJob        // scheduled - tiap 15 menit
SyncPaymentStatusJob          // scheduled - reconciliation tiap malam
NotifyInactiveMembersJob      // scheduled - tiap hari cek inactive
```

### 3.3 Scheduled Tasks (Cron via Laravel Scheduler)

```php
// app/Console/Kernel.php

$schedule->job(new CancelExpiredOrdersJob)->everyFifteenMinutes();
$schedule->job(new SyncPaymentStatusJob)->dailyAt('02:00');
$schedule->job(new NotifyInactiveMembersJob)->dailyAt('09:00');
$schedule->command('backup:run')->dailyAt('03:00');
$schedule->command('telescope:prune')->daily();
$schedule->command('audit:prune', ['--days=730'])->monthly();
```

---

## 4. API DESIGN PRINCIPLES

### 4.1 Base URL Structure
```
https://api.fatherman.id/v1/
```

### 4.2 Authentication Headers
```
Authorization: Bearer {access_token}
X-Refresh-Token: {refresh_token}          // untuk auto-refresh
X-Device-ID: {device_fingerprint}         // tracking device
```

### 4.3 Standard Response Format
```json
// Success
{
    "success": true,
    "data": { ... },
    "meta": {
        "pagination": { ... }              // jika list
    },
    "message": "Order created successfully"
}

// Error
{
    "success": false,
    "error": {
        "code": "VALIDATION_ERROR",
        "message": "The email field is required.",
        "details": {
            "email": ["The email field is required."]
        }
    }
}
```

### 4.4 Error Codes
| Code | HTTP Status | Deskripsi |
|---|---|---|
| `VALIDATION_ERROR` | 422 | Input tidak valid |
| `UNAUTHORIZED` | 401 | Token tidak ada / expired |
| `FORBIDDEN` | 403 | Tidak punya akses |
| `NOT_FOUND` | 404 | Resource tidak ditemukan |
| `ALREADY_OWNED` | 409 | Produk sudah dimiliki |
| `PAYMENT_FAILED` | 422 | Pembayaran gagal |
| `RATE_LIMITED` | 429 | Terlalu banyak request |
| `SERVER_ERROR` | 500 | Internal error |

---

## 5. FRONTEND ARCHITECTURE

### 5.1 Vue.js Project Structure
```
src/
├── api/                     # Axios API calls, per-domain
│   ├── auth.js
│   ├── checkout.js
│   ├── orders.js
│   ├── courses.js
│   └── ...
├── components/
│   ├── ui/                  # Base UI: Button, Input, Modal, etc.
│   ├── checkout/            # Checkout-specific components
│   ├── course/              # eCourse player components
│   ├── dashboard/           # Admin/CS dashboard widgets
│   └── member/              # Member area components
├── layouts/
│   ├── PublicLayout.vue     # Checkout & landing
│   ├── MemberLayout.vue     # Member area
│   └── AdminLayout.vue      # Admin & CS dashboard
├── pages/                   # Route pages (Vue Router)
├── stores/                  # Pinia stores
│   ├── auth.js
│   ├── cart.js
│   └── notification.js
├── composables/             # Reusable Composition API logic
│   ├── useAuth.js
│   ├── useCheckout.js
│   └── usePagination.js
└── utils/                   # Helpers, formatters, validators
```

### 5.2 Token Management (Frontend)
```javascript
// Axios interceptor — auto refresh token
axios.interceptors.response.use(
    response => response,
    async error => {
        if (error.response?.status === 401 && !error.config._retry) {
            error.config._retry = true;
            const newToken = await refreshAccessToken();
            error.config.headers['Authorization'] = `Bearer ${newToken}`;
            return axios(error.config);
        }
        return Promise.reject(error);
    }
);
// Access token: disimpan di memory (tidak di localStorage!)
// Refresh token: disimpan di httpOnly cookie
```

---

## 6. INTEGRATION ARCHITECTURE

### 6.1 Midtrans Integration

```
Laravel App
    │
    ├── Create Transaction → Midtrans API (server-to-server)
    │   POST https://app.sandbox.midtrans.com/snap/v1/transactions
    │   Headers: Authorization: Basic {base64(SERVER_KEY:)}
    │
    ├── Redirect user ke Snap payment page / embed Snap.js
    │
    └── Webhook Handler: POST /api/v1/payment/callback
        ├── Verify: SHA512(order_id + status_code + gross_amount + SERVER_KEY)
        ├── Verify status via GET https://api.sandbox.midtrans.com/v2/{order_id}/status
        └── Process: update DB, grant access, send notif
```

### 6.2 Raja Ongkir Integration

```
Frontend
    │
    └── POST /api/v1/shipping/cost
        Body: { origin_city_id, destination_city_id, weight, courier }
        │
        ├── Check Redis cache (TTL: 3600 sec)
        │   Key: "ongkir:{origin}:{destination}:{weight}:{courier}"
        │
        └── Cache miss → GET https://api.rajaongkir.com/starter/cost
            Headers: key: {RAJAONGKIR_API_KEY}
            └── Store to Redis → return to frontend
```

### 6.3 Fonnte WA Integration

```
Queue Job: SendWANotificationJob
    │
    └── POST https://api.fonnte.com/send
        Headers: Authorization: {FONNTE_TOKEN}
        Body: {
            target: "628123456789",
            message: "Halo {{nama}}, order Anda #{{no_order}} telah dikonfirmasi...",
            countryCode: "62"
        }
        │
        └── Log result ke notification_logs
            ├── success: mark status = 'sent'
            └── failed: mark status = 'failed', log error, retry max 3x
```

---

## 7. MIGRATION PLAN (aMember → Sistem Baru)

### 7.1 Data yang Dimigrasikan
1. **Users**: email, nama, password hash
2. **Product Ownership**: mapping user → produk yang sudah dibeli
3. **Order History**: data historis order
4. **Enrollment**: progress kursus (jika ada tracking)

### 7.2 Script Migrasi (Laravel Command)

```php
// artisan migrate:amember

Step 1: Export dari aMember → CSV/SQL
Step 2: Transform & validate data
Step 3: Import users (re-hash password atau force reset)
Step 4: Import product ownership
Step 5: Send email ke semua member: instruksi login baru
Step 6: Monitor login success rate (target: 80% dalam 7 hari)
Step 7: Shutdown aMember setelah 30 hari
```

### 7.3 Parallel Running Period
- Hari 0-7: Sistem baru live, aMember masih aktif
- Hari 7-14: Email blast ke member yang belum login ke sistem baru
- Hari 14-21: aMember mode read-only (tidak bisa beli baru)
- Hari 30: Shutdown aMember
