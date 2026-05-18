# API Specification
## Fatherman — Unified Commerce & eLearning Platform
**Versi:** 1.0 | **Base URL:** `https://api.fatherman.id/v1` | **Format:** REST + JSON

---

## AUTH ENDPOINTS

### POST /auth/register
Registrasi akun baru.

**Request:**
```json
{
    "name": "Budi Santoso",
    "email": "budi@example.com",
    "phone": "081234567890",
    "password": "Str0ngP@ssword",
    "password_confirmation": "Str0ngP@ssword"
}
```
**Validasi:** password min 8 karakter, 1 huruf besar, 1 angka, 1 simbol.

**Response 201:**
```json
{
    "success": true,
    "data": {
        "user": { "id": 1, "name": "Budi Santoso", "email": "budi@example.com" },
        "access_token": "eyJ...",
        "token_type": "Bearer",
        "expires_in": 900
    }
}
```

---

### POST /auth/login
Login dengan email + password.

**Request:**
```json
{
    "email": "budi@example.com",
    "password": "Str0ngP@ssword",
    "remember_me": true
}
```

**Response 200:**
```json
{
    "success": true,
    "data": {
        "user": {
            "id": 1,
            "name": "Budi Santoso",
            "email": "budi@example.com",
            "role": "member",
            "avatar_url": null
        },
        "access_token": "eyJ...",
        "token_type": "Bearer",
        "expires_in": 900
    }
}
```
*Refresh token dikirim via `Set-Cookie: refresh_token=...; HttpOnly; Secure; SameSite=Strict`*

---

### POST /auth/google
Login/register via Google OAuth.

**Request:**
```json
{
    "id_token": "google_id_token_dari_frontend"
}
```

---

### POST /auth/refresh
Refresh access token menggunakan refresh token dari cookie.

**Headers:** Cookie: `refresh_token={token}`

**Response 200:**
```json
{
    "success": true,
    "data": {
        "access_token": "eyJ...",
        "expires_in": 900
    }
}
```

---

### POST /auth/logout
Invalidate refresh token.

**Headers:** `Authorization: Bearer {token}`

**Response 200:** `{ "success": true, "message": "Logged out successfully" }`

---

### POST /auth/forgot-password
Kirim email reset password.

**Request:** `{ "email": "budi@example.com" }`

**Response 200:** `{ "success": true, "message": "Reset link sent to your email" }`

---

### POST /auth/reset-password
**Request:**
```json
{
    "token": "reset_token_dari_email",
    "email": "budi@example.com",
    "password": "NewP@ssword123",
    "password_confirmation": "NewP@ssword123"
}
```

---

## PRODUCT ENDPOINTS

### GET /products
List produk aktif (public).

**Query Params:**
| Param | Type | Default | Deskripsi |
|---|---|---|---|
| `page` | int | 1 | Halaman |
| `per_page` | int | 20 | Item per halaman (max 100) |
| `type` | string | - | Filter: physical, digital, ecourse, bundle |
| `category` | string | - | Slug kategori |
| `search` | string | - | Full-text search |

**Response 200:**
```json
{
    "success": true,
    "data": [
        {
            "id": 1,
            "sku": "PRD-001",
            "name": "Kursus Digital Marketing 2026",
            "type": "ecourse",
            "price": 500000,
            "sale_price": 350000,
            "thumbnail_url": "https://cdn.fatherman.id/products/1.jpg",
            "short_description": "Belajar digital marketing dari nol..."
        }
    ],
    "meta": {
        "pagination": {
            "total": 45,
            "per_page": 20,
            "current_page": 1,
            "last_page": 3
        }
    }
}
```

---

### GET /products/{id}
Detail produk (public).

**Response 200:**
```json
{
    "success": true,
    "data": {
        "id": 1,
        "name": "Kursus Digital Marketing 2026",
        "type": "ecourse",
        "price": 500000,
        "sale_price": 350000,
        "description": "...",
        "weight_gram": 0,
        "media": [
            { "type": "image", "url": "https://...", "sort_order": 1 }
        ],
        "course_preview": {
            "total_lessons": 48,
            "estimated_minutes": 720,
            "instructor_name": "Pak Fatherman",
            "modules": [
                {
                    "title": "Modul 1: Pengenalan",
                    "lessons": [
                        { "title": "Apa itu Digital Marketing?", "is_free_preview": true }
                    ]
                }
            ]
        }
    }
}
```

---

## CHECKOUT ENDPOINTS

### POST /checkout/check-ownership
Cek produk apa yang sudah dimiliki oleh email/HP.

**Request:**
```json
{
    "email": "budi@example.com",
    "phone": "081234567890",
    "product_ids": [1, 2, 3, 5]
}
```

**Response 200:**
```json
{
    "success": true,
    "data": {
        "owned_product_ids": [1, 2],
        "user_exists": true,
        "user_prefill": {
            "name": "Budi Santoso",
            "email": "budi@example.com",
            "phone": "081234567890",
            "last_address": {
                "recipient_name": "Budi Santoso",
                "address_line1": "Jl. Mawar No. 5",
                "city": "Jakarta Selatan",
                "province": "DKI Jakarta",
                "postal_code": "12345"
            }
        }
    }
}
```

---

### POST /checkout/shipping-cost
Hitung ongkos kirim.

**Request:**
```json
{
    "destination_city_id": "152",
    "weight_gram": 500,
    "couriers": ["jne", "jnt", "sicepat"]
}
```

**Response 200:**
```json
{
    "success": true,
    "data": [
        {
            "courier": "jne",
            "service": "REG",
            "description": "Layanan Reguler",
            "cost": 18000,
            "estimated_days": "2-3"
        },
        {
            "courier": "jne",
            "service": "OKE",
            "description": "Ongkos Kirim Ekonomis",
            "cost": 14000,
            "estimated_days": "3-5"
        }
    ]
}
```

---

### POST /checkout/validate-coupon
Validasi kode kupon sebelum checkout.

**Request:**
```json
{
    "code": "HEMAT50",
    "subtotal": 500000,
    "product_ids": [1]
}
```

**Response 200:**
```json
{
    "success": true,
    "data": {
        "code": "HEMAT50",
        "type": "percentage",
        "value": 50,
        "discount_amount": 250000,
        "final_amount": 250000,
        "message": "Kode promo berhasil! Diskon 50%"
    }
}
```

---

### POST /checkout
Buat order baru (guest atau login).

**Request:**
```json
{
    "buyer_name": "Budi Santoso",
    "email": "budi@example.com",
    "phone": "081234567890",
    "items": [
        { "product_id": 1, "quantity": 1 },
        { "product_id": 4, "quantity": 2 }
    ],
    "shipping": {
        "recipient_name": "Budi Santoso",
        "phone": "081234567890",
        "address_line1": "Jl. Mawar No. 5",
        "city": "Jakarta Selatan",
        "province": "DKI Jakarta",
        "postal_code": "12345",
        "rajaongkir_city_id": "152",
        "courier": "jne",
        "service": "REG"
    },
    "coupon_code": "HEMAT50",
    "payment_method": "va_bca",
    "notes": "Tolong dikemas dengan bubble wrap"
}
```
*Field `shipping` hanya wajib jika ada produk fisik dalam cart.*

**Response 201:**
```json
{
    "success": true,
    "data": {
        "order": {
            "id": 101,
            "order_number": "FTM-2026051800001",
            "status": "pending_payment",
            "total_amount": 250000
        },
        "payment": {
            "method": "va_bca",
            "va_number": "1234567890123456",
            "expires_at": "2026-05-18T15:00:00Z",
            "amount": 250000
        }
    }
}
```

---

### GET /checkout/order-status/{order_number}
Cek status order (untuk polling halaman thank you).

**Response 200:**
```json
{
    "success": true,
    "data": {
        "order_number": "FTM-2026051800001",
        "status": "paid",
        "products": [
            { "name": "Kursus Digital Marketing", "type": "ecourse", "access_url": "/member/courses/1" }
        ]
    }
}
```

---

## PAYMENT ENDPOINTS

### POST /payment/callback
Webhook dari Duitku (internal, tidak dipanggil frontend).

**Headers:** Duitku sends raw POST with signature
**Proses:** Verify signature → verify via Duitku API → update order

---

### GET /payment/{order_number}/status
Re-check status pembayaran (jika user kembali ke halaman bayar).

---

## MEMBER ENDPOINTS

### GET /member/dashboard
Dashboard member — kursus yang dimiliki + progress.

**Auth:** Required (member)

**Response 200:**
```json
{
    "success": true,
    "data": {
        "user": { "name": "Budi", "avatar_url": null },
        "enrollments": [
            {
                "course_id": 1,
                "course_title": "Digital Marketing 2026",
                "thumbnail_url": "...",
                "progress_percent": 45.5,
                "last_lesson": { "id": 12, "title": "Google Ads Basics" },
                "last_accessed_at": "2026-05-17T10:30:00Z"
            }
        ],
        "notifications": [
            { "id": 1, "message": "Modul baru telah ditambahkan!", "is_read": false }
        ]
    }
}
```

---

### GET /member/courses/{course_id}
Detail kursus beserta struktur modul dan lesson.

**Auth:** Required, must own the course

**Response 200:**
```json
{
    "success": true,
    "data": {
        "id": 1,
        "title": "Digital Marketing 2026",
        "progress_percent": 45.5,
        "certificate": null,
        "modules": [
            {
                "id": 1,
                "title": "Modul 1: Pengenalan",
                "lessons": [
                    {
                        "id": 1,
                        "title": "Apa itu Digital Marketing?",
                        "content_type": "video",
                        "duration_minutes": 15,
                        "is_completed": true,
                        "is_locked": false
                    },
                    {
                        "id": 2,
                        "title": "Strategi Konten",
                        "content_type": "video",
                        "duration_minutes": 25,
                        "is_completed": false,
                        "is_locked": false
                    }
                ]
            }
        ]
    }
}
```

---

### GET /member/lessons/{lesson_id}
Akses konten lesson.

**Auth:** Required, must own course, lesson must not be locked

**Response 200:**
```json
{
    "success": true,
    "data": {
        "id": 2,
        "title": "Strategi Konten",
        "content_type": "video",
        "video_url": "https://www.youtube.com/embed/abcdefg",
        "video_provider": "youtube",
        "duration_minutes": 25,
        "progress": {
            "status": "in_progress",
            "watch_duration_seconds": 320
        },
        "next_lesson": { "id": 3, "title": "SEO Dasar" },
        "prev_lesson": { "id": 1, "title": "Apa itu Digital Marketing?" }
    }
}
```

---

### POST /member/lessons/{lesson_id}/progress
Update progress menonton.

**Request:**
```json
{
    "watch_duration_seconds": 450,
    "is_completed": false
}
```

---

### POST /member/lessons/{lesson_id}/complete
Tandai lesson sebagai selesai.

**Response 200:**
```json
{
    "success": true,
    "data": {
        "lesson_id": 2,
        "course_progress": 52.1,
        "next_lesson_unlocked": true,
        "certificate_earned": false
    }
}
```

---

### GET /member/certificates
Daftar sertifikat yang dimiliki.

---

### GET /member/certificates/{id}/download
Download PDF sertifikat.

---

### PUT /member/profile
Update profil member.

**Request:**
```json
{
    "name": "Budi Santoso",
    "phone": "081234567890",
    "current_password": "OldPass123",
    "new_password": "NewP@ss456",
    "new_password_confirmation": "NewP@ss456"
}
```

---

## ADMIN ORDER ENDPOINTS

**Base:** `/admin/orders` | **Auth:** Admin atau CS role

### GET /admin/orders
List semua order dengan filter.

**Query Params:**
| Param | Type | Deskripsi |
|---|---|---|
| `status` | string | Filter by status |
| `date_from` | date | Filter tanggal mulai |
| `date_to` | date | Filter tanggal akhir |
| `type` | string | physical, digital, ecourse |
| `search` | string | Nama, email, nomor order |
| `page` | int | Halaman |
| `per_page` | int | Default 25 |

**Response 200:**
```json
{
    "success": true,
    "data": [
        {
            "id": 101,
            "order_number": "FTM-2026051800001",
            "buyer_name": "Budi Santoso",
            "buyer_email": "budi@example.com",
            "buyer_phone": "081234567890",
            "total_amount": 250000,
            "status": "pending_payment",
            "created_at": "2026-05-18T10:00:00Z",
            "last_followup_at": null
        }
    ],
    "meta": { "pagination": { ... }, "summary": {
        "total_revenue_today": 5400000,
        "orders_pending": 12,
        "orders_paid_today": 8
    }}
}
```

---

### GET /admin/orders/{id}
Detail order lengkap.

**Response 200:**
```json
{
    "success": true,
    "data": {
        "id": 101,
        "order_number": "FTM-2026051800001",
        "buyer": { "name": "Budi", "email": "...", "phone": "..." },
        "items": [
            { "product_name": "Kursus DM", "quantity": 1, "price": 250000 }
        ],
        "shipping": { "address": "...", "courier": "JNE REG", "resi": null },
        "payment": { "method": "va_bca", "va_number": "...", "status": "pending" },
        "status_logs": [
            { "from": null, "to": "pending_payment", "at": "...", "by": "system" }
        ],
        "followup_logs": [],
        "notes_internal": ""
    }
}
```

---

### PUT /admin/orders/{id}/status
Update status order.

**Auth:** Admin only (CS tidak bisa ubah status)

**Request:** `{ "status": "shipped", "note": "Sudah dikemas" }`

---

### PUT /admin/orders/{id}/tracking
Update nomor resi pengiriman.

**Request:** `{ "courier_name": "JNE", "tracking_number": "JNE123456789" }`

---

### POST /admin/orders/{id}/followup
Tambah log follow-up.

**Request:**
```json
{
    "channel": "whatsapp",
    "content": "Sudah dihubungi, customer konfirmasi masih ingin lanjutkan order",
    "send_notification": true
}
```

---

### GET /admin/orders/export
Export data order ke CSV/Excel.

**Query Params:** `format=csv|xlsx`, `date_from`, `date_to`, `status`

**Response:** File download

---

## ADMIN ECOURSE ENDPOINTS

### GET /admin/courses/{course_id}/members
List member yang enroll beserta progress.

**Query Params:** `progress_min`, `progress_max`, `last_active_days`, `search`

**Response 200:**
```json
{
    "success": true,
    "data": [
        {
            "user_id": 5,
            "name": "Budi Santoso",
            "email": "budi@example.com",
            "progress_percent": 45.5,
            "last_lesson_title": "Google Ads Basics",
            "last_active_at": "2026-05-17T10:30:00Z",
            "enrolled_at": "2026-04-01T08:00:00Z",
            "is_completed": false,
            "has_certificate": false
        }
    ]
}
```

---

### GET /admin/courses/{course_id}/members/{user_id}/progress
Detail progress per member.

**Response 200:**
```json
{
    "success": true,
    "data": {
        "user": { "name": "Budi", "email": "..." },
        "course": { "title": "Digital Marketing 2026" },
        "overall_progress": 45.5,
        "total_watch_time_minutes": 187,
        "modules": [
            {
                "title": "Modul 1",
                "lessons": [
                    {
                        "title": "Pengenalan",
                        "status": "completed",
                        "completed_at": "2026-04-02T10:00:00Z"
                    }
                ]
            }
        ]
    }
}
```

---

### POST /admin/courses/{course_id}/notify-inactive
Kirim notifikasi ke member inactive.

**Request:**
```json
{
    "days_inactive": 7,
    "channel": "whatsapp",
    "message_override": null
}
```

**Response 200:**
```json
{
    "success": true,
    "data": {
        "members_notified": 14,
        "jobs_queued": 14
    }
}
```

---

### GET /admin/courses/{course_id}/analytics
Analytics konten kursus.

**Response 200:**
```json
{
    "success": true,
    "data": {
        "total_enrolled": 150,
        "avg_progress": 42.3,
        "completion_rate": 18.5,
        "lessons": [
            {
                "lesson_id": 1,
                "title": "Pengenalan",
                "completion_rate": 92.0,
                "avg_watch_seconds": 780
            },
            {
                "lesson_id": 8,
                "title": "Advanced Facebook Ads",
                "completion_rate": 21.3,
                "avg_watch_seconds": 220
            }
        ]
    }
}
```

---

## ADMIN DASHBOARD ENDPOINTS

### GET /admin/dashboard/summary
Summary untuk widget dashboard.

**Response 200:**
```json
{
    "success": true,
    "data": {
        "revenue": {
            "today": 5400000,
            "this_month": 87600000,
            "last_month": 74200000,
            "growth_percent": 18.1
        },
        "orders": {
            "new_today": 23,
            "pending_payment": 12,
            "need_action": 5
        },
        "members": {
            "total_active": 847,
            "new_this_month": 68,
            "inactive_7days": 34
        },
        "top_products": [
            { "name": "Kursus DM 2026", "sold": 45, "revenue": 15750000 }
        ]
    }
}
```

---

### GET /admin/dashboard/chart/orders
Data untuk grafik tren order.

**Query Params:** `period=7d|30d|3m|1y`

**Response 200:**
```json
{
    "success": true,
    "data": {
        "labels": ["2026-05-12", "2026-05-13", "..."],
        "datasets": [
            { "label": "Orders", "data": [12, 19, 8, 25, ...] },
            { "label": "Revenue", "data": [2400000, 3800000, ...] }
        ]
    }
}
```

---

## NOTIFICATION TEMPLATE ENDPOINTS

### GET /admin/notification-templates
List semua template notifikasi.

### PUT /admin/notification-templates/{id}
Update isi template.

**Request:**
```json
{
    "body_template": "Halo {{nama}}, pesanan #{{no_order}} sebesar Rp{{total}} telah dikonfirmasi. Silakan bayar ke VA BCA: {{va_number}} sebelum {{expired_at}}.",
    "is_active": true
}
```

---

## SHIPPING CITY ENDPOINT

### GET /shipping/cities
Daftar kota/kabupaten untuk dropdown alamat.

**Query Params:** `search=jakarta`, `province_id=6`

**Response:** List kota dengan `id`, `name`, `province`, `type`
