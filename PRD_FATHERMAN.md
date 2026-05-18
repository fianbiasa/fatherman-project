# Product Requirements Document (PRD)
## Fatherman — Platform Unified Commerce & eLearning
**Versi:** 1.0  
**Tanggal:** 18 Mei 2026  
**Status:** Draft Final

---

## 1. OVERVIEW & TUJUAN

### 1.1 Latar Belakang
Fatherman saat ini mengoperasikan sistem yang terfragmentasi:
- Checkout produk fisik & digital di sistem terpisah
- Admin/CS mengelola order secara manual
- Notifikasi masih dilakukan manual
- Member eCourse harus login di 2 sistem berbeda (WordPress + aMember)
- Tidak ada visibility progres member bagi admin

### 1.2 Tujuan Produk
Membangun **satu platform terintegrasi** yang menangani:
1. Unified Checkout untuk produk fisik & digital
2. Dashboard Order Management untuk Admin & CS
3. Integrasi multi-channel: WA, Email, Ongkir, Virtual Account
4. Unified Member Area untuk eCourse
5. Admin Dashboard Progres eCourse

### 1.3 Target Pengguna
| Role | Deskripsi |
|---|---|
| **Guest** | Calon pembeli tanpa akun, checkout via email/HP |
| **Member** | Pembeli yang sudah punya akun, akses eCourse |
| **CS (Customer Service)** | Follow up order, komunikasi dengan buyer |
| **Admin** | Full access, manage produk, user, laporan |
| **Super Admin** | Konfigurasi sistem, integrasi, setting global |

---

## 2. FITUR 1 — UNIFIED CHECKOUT PAGE

### 2.1 Deskripsi
Halaman checkout tunggal yang mampu menangani pembelian produk fisik (perlu ongkir) dan digital (tanpa ongkir), dengan kemampuan identifikasi buyer tanpa paksa login.

### 2.2 User Stories

**US-01:** Sebagai guest, saya ingin bisa checkout hanya dengan mengisi email/HP tanpa membuat akun, agar prosesnya cepat.

**US-02:** Sebagai buyer, saya ingin sistem otomatis mendeteksi produk apa saja yang sudah saya miliki berdasarkan email/HP, agar tidak perlu beli ulang.

**US-03:** Sebagai buyer produk digital, saya tidak ingin mengisi alamat pengiriman karena produk saya tidak perlu dikirim.

**US-04:** Sebagai buyer produk fisik, saya ingin bisa cek ongkos kirim realtime sebelum bayar.

**US-05:** Sebagai returning buyer, saya ingin data saya (nama, email, alamat) terisi otomatis saat checkout ulang.

**US-06:** Sebagai buyer, saya ingin memilih metode pembayaran: transfer bank, virtual account, e-wallet, atau QRIS.

### 2.3 Acceptance Criteria

#### AC-01: Guest Checkout
- [ ] Form checkout dapat disubmit tanpa login
- [ ] Email atau nomor HP wajib diisi (minimal salah satu)
- [ ] Sistem membuat "temporary identity" berbasis email/HP
- [ ] Setelah bayar, buyer mendapat opsi untuk membuat akun

#### AC-02: Ownership Detection
- [ ] Saat email/HP diinput, sistem query kepemilikan produk
- [ ] Produk yang sudah dimiliki ditampilkan badge "Sudah Dimiliki"
- [ ] Tidak bisa beli produk yang sudah dimiliki (kecuali produk repeat-purchase)
- [ ] Kepemilikan dicek berdasarkan: email, HP, atau user_id jika login

#### AC-03: Produk Fisik vs Digital
- [ ] Field alamat hanya muncul jika cart berisi produk fisik
- [ ] Cek ongkir otomatis via API Raja Ongkir / Shipper
- [ ] Produk digital: setelah bayar, akses langsung tersedia
- [ ] Produk fisik: status order berubah ke "menunggu pengiriman"

#### AC-04: Pembayaran
- [ ] Minimal tersedia: Virtual Account (BCA, BRI, BNI, Mandiri), QRIS, Transfer Manual
- [ ] Setelah payment gateway konfirmasi, status order otomatis update
- [ ] Jika pembayaran gagal/expire, order dibatalkan otomatis setelah X jam

#### AC-05: Returning Buyer
- [ ] Pre-fill form jika email/HP pernah digunakan sebelumnya
- [ ] Data yang di-pre-fill: nama, email, HP, alamat terakhir
- [ ] User dapat mengubah data sebelum submit

### 2.4 Flow Diagram

```
[Landing/Sales Page]
        |
        v
[Klik Beli / Add to Cart]
        |
        v
[Checkout Page]
        |
   +---------+
   |  Masuk  |          Tidak
   |  email/ |---------> [Isi Form Manual]
   |    HP   |
   +---------+
        | Ya
        v
[Auto-detect Identitas] ---> [Cek kepemilikan produk]
        |
        v
[Tampilkan Cart] --> [Produk fisik? -> Form alamat + cek ongkir]
        |
        v
[Pilih Metode Bayar]
        |
        v
[Generate Order + Payment Link/VA]
        |
        v
[Callback/Webhook Payment Gateway]
        |
   +----+----+
   |Sukses   |Gagal
   v         v
[Aktivasi]  [Order Cancelled]
[Akses]
```

### 2.5 Non-Functional Requirements
- Halaman checkout load < 2 detik
- Cek ongkir response < 3 detik
- Payment callback diproses < 5 detik
- Form data tidak boleh tersimpan di browser cache (sensitif)

---

## 3. FITUR 2 — DASHBOARD ADMIN & CS ORDER MANAGEMENT

### 3.1 Deskripsi
Dashboard terpusat untuk Admin dan CS mengelola semua order: melihat status, follow up buyer, update pengiriman, dan export laporan.

### 3.2 User Stories

**US-07:** Sebagai Admin, saya ingin melihat semua order dalam satu tabel dengan filter status, tanggal, dan jenis produk.

**US-08:** Sebagai CS, saya ingin bisa klik satu tombol untuk kirim pesan WA follow-up ke buyer.

**US-09:** Sebagai Admin, saya ingin bisa update nomor resi pengiriman langsung dari dashboard.

**US-10:** Sebagai CS, saya ingin bisa tambah catatan internal per order untuk koordinasi tim.

**US-11:** Sebagai Admin, saya ingin export data order ke Excel/CSV untuk laporan bulanan.

**US-12:** Sebagai Admin, saya ingin lihat summary: total revenue hari ini, order pending, order sukses.

### 3.3 Acceptance Criteria

#### AC-06: Tabel Order
- [ ] Kolom: No. Order, Nama, Produk, Total, Status, Tanggal, Aksi
- [ ] Filter: status (pending/paid/shipped/done/cancelled), tanggal range, jenis produk
- [ ] Search: nama buyer, email, no. order
- [ ] Pagination: 25/50/100 per halaman
- [ ] Bulk action: tandai selesai, batalkan order

#### AC-07: Detail Order
- [ ] Data buyer lengkap (nama, email, HP, alamat)
- [ ] Daftar produk yang dibeli + harga
- [ ] Timeline status order
- [ ] Tombol: konfirmasi bayar manual, update resi, batalkan
- [ ] Form catatan internal CS (tidak terlihat buyer)
- [ ] History follow-up log

#### AC-08: Follow-up WA
- [ ] Template WA yang bisa dikustomisasi per status
- [ ] Klik tombol "Kirim WA" → buka link WA dengan pesan ter-pre-fill
- [ ] Atau integrasi WA API (Fonnte/WA Business API) untuk kirim otomatis
- [ ] Log waktu terakhir follow-up

#### AC-09: Dashboard Summary
- [ ] Widget: Revenue hari ini / bulan ini
- [ ] Widget: Order baru (24 jam terakhir)
- [ ] Widget: Order perlu tindakan (pending payment, perlu konfirmasi)
- [ ] Widget: Produk terlaris
- [ ] Grafik: tren order harian (7/30 hari)

#### AC-10: Role-based Access
- [ ] CS: hanya bisa lihat order, tambah catatan, kirim WA, tidak bisa hapus
- [ ] Admin: semua akses termasuk konfigurasi produk & user
- [ ] Setiap aksi tercatat dalam audit log (siapa, apa, kapan)

### 3.4 Status Order Flow
```
PENDING_PAYMENT
      |
      v
   PAID ---------> PROCESSING
      |                  |
      |                  v
      |            SHIPPED (input resi)
      |                  |
      |                  v
      +-----------> COMPLETED
      |
      v
  CANCELLED (expired/manual)
```

---

## 4. FITUR 3 — INTEGRASI MULTI-CHANNEL

### 4.1 WhatsApp Notification

#### 4.1.1 Trigger Otomatis
| Event | Penerima | Template |
|---|---|---|
| Order dibuat | Buyer | Konfirmasi order + instruksi bayar |
| Pembayaran diterima | Buyer | Invoice + detail produk/akses |
| Order dikirim | Buyer | Nomor resi + tracking link |
| Order selesai | Buyer | Terima kasih + request review |
| Order pending > 2 jam | Buyer | Reminder pembayaran |
| Order baru masuk | Admin/CS | Notifikasi internal |

#### 4.1.2 Provider WA
- **Primary:** Fonnte (lebih mudah setup, WABA)
- **Fallback:** Twilio WhatsApp / WA Business API langsung
- Template pesan menggunakan variabel dinamis: `{{nama}}`, `{{no_order}}`, `{{total}}`, `{{link_bayar}}`

### 4.2 Email Notification

#### 4.2.1 Trigger
| Event | Template Email |
|---|---|
| Order dibuat | Order Confirmation dengan detail lengkap |
| Pembayaran sukses | Invoice PDF + instruksi akses |
| Order dikirim | Shipping notification |
| Welcome member baru | Onboarding email + link akses |
| Reset password | Link reset (expire 1 jam) |

#### 4.2.2 Provider Email
- **SMTP:** Mailgun / Postmark (deliverability tinggi, ada tracking)
- HTML template menggunakan MJML atau Blade component
- Unsubscribe link wajib ada di setiap email marketing

### 4.3 Cek Ongkir

#### 4.3.1 Integrasi API
- **Primary:** RajaOngkir (Pro Plan — support semua kurir)
- **Kurir yang didukung:** JNE, JNT, SiCepat, Anteraja, Pos Indonesia, Wahana, Tiki
- **Flow:** Input kota asal (otomatis dari setting toko) + kota tujuan + berat → tampilkan opsi kurir + harga

#### 4.3.2 Fitur
- Caching hasil cek ongkir (TTL 1 jam per kombinasi asal-tujuan-berat)
- Fallback manual jika API down: admin set flat rate per zona
- Estimasi hari pengiriman ditampilkan

### 4.4 Payment Gateway — Virtual Account

#### 4.4.1 Provider
- **Primary:** Midtrans (Snap) — support VA, QRIS, e-wallet, kartu kredit
- **Alternatif:** Xendit (jika volume tinggi, fee lebih hemat)

#### 4.4.2 Metode yang Didukung
| Metode | Provider |
|---|---|
| Virtual Account BCA | Midtrans |
| Virtual Account BRI | Midtrans |
| Virtual Account BNI | Midtrans |
| Virtual Account Mandiri | Midtrans |
| QRIS | Midtrans |
| GoPay | Midtrans |
| OVO | Midtrans |
| ShopeePay | Midtrans |
| Transfer Manual | Custom (konfirmasi admin) |

#### 4.4.3 Webhook & Reconciliation
- Webhook Midtrans diterima di endpoint `/api/payment/callback`
- Signature verification wajib dilakukan sebelum proses
- Double-check status ke Midtrans API (jangan hanya percaya webhook)
- Reconciliation harian: bandingkan data order vs data Midtrans

---

## 5. FITUR 4 — UNIFIED MEMBER AREA (eCourse)

### 5.1 Deskripsi
Menggantikan sistem login ganda (WordPress + aMember) dengan satu sistem auth terpusat. Member cukup login sekali untuk mengakses semua konten eCourse dan melakukan pembelian baru.

### 5.2 User Stories

**US-13:** Sebagai member, saya ingin login hanya di satu tempat dan langsung bisa akses semua kursus saya.

**US-14:** Sebagai member, saya ingin bisa beli kursus baru langsung dari member area tanpa pergi ke halaman lain.

**US-15:** Sebagai member, saya ingin melihat daftar kursus yang saya miliki, progres belajar, dan sertifikat.

**US-16:** Sebagai calon member, saya ingin bisa daftar akun dengan email atau Google OAuth.

**US-17:** Sebagai member, saya ingin reset password mudah via email.

### 5.3 Acceptance Criteria

#### AC-11: Unified Auth
- [ ] Login via email + password
- [ ] Login via Google OAuth 2.0
- [ ] Token berbasis JWT (access token 15 menit + refresh token 30 hari)
- [ ] "Remember me" memperpanjang refresh token
- [ ] Session invalidation saat ganti password

#### AC-12: Member Dashboard
- [ ] Daftar kursus yang dimiliki (card dengan thumbnail, title, progres %)
- [ ] Continue belajar: langsung ke materi terakhir yang diakses
- [ ] Notifikasi: materi baru, pengumuman, promo
- [ ] Profile settings: nama, foto, HP, ganti password

#### AC-13: Akses Konten eCourse
- [ ] Materi tersusun: Modul → Sub-modul → Lesson
- [ ] Jenis konten: Video (YouTube embed / hosted), PDF, Teks, Quiz
- [ ] Progress tracking: lesson ditandai selesai setelah dibuka/ditonton
- [ ] Lock system: lesson berikutnya unlock setelah lesson sebelumnya selesai (opsional per kursus)
- [ ] Sertifikat otomatis di-generate saat progress = 100%

#### AC-14: Pembelian dari Member Area
- [ ] Katalog produk tersedia di dalam member area
- [ ] Data buyer ter-pre-fill dari profile member
- [ ] Setelah beli, kursus baru langsung muncul di dashboard tanpa perlu refresh/re-login

#### AC-15: Migrasi dari aMember
- [ ] Script migrasi data member dari aMember ke sistem baru
- [ ] Mapping: user, produk yang dimiliki, password (bcrypt rehash)
- [ ] Masa transisi: sistem lama tetap jalan 30 hari sambil migrasi
- [ ] Email blast ke member lama tentang cara login baru

---

## 6. FITUR 5 — ADMIN DASHBOARD PROGRES eCOURSE

### 6.1 Deskripsi
Dashboard khusus untuk admin memantau aktivitas belajar setiap member: modul apa yang sudah diakses, berapa lama, dan siapa yang butuh follow-up.

### 6.2 User Stories

**US-18:** Sebagai Admin, saya ingin melihat daftar semua member per kursus beserta progress % mereka.

**US-19:** Sebagai Admin, saya ingin tahu member mana yang sudah lama tidak login (inactive).

**US-20:** Sebagai Admin, saya ingin bisa kirim notifikasi pengingat ke member yang belum progress.

**US-21:** Sebagai Admin, saya ingin melihat lesson mana yang paling banyak/sedikit diakses.

**US-22:** Sebagai Admin, saya ingin export data progres ke Excel untuk laporan.

### 6.3 Acceptance Criteria

#### AC-16: Tabel Progres Member
- [ ] Kolom: Nama, Email, Kursus, Progres %, Lesson Terakhir, Terakhir Aktif, Sertifikat
- [ ] Filter per kursus, status progres (0-25%, 26-75%, 76-99%, 100%)
- [ ] Sort: progres tertinggi/terendah, terbaru aktif
- [ ] Search: nama, email

#### AC-17: Detail Progres Per Member
- [ ] Daftar semua lesson dengan status: belum, sedang, selesai
- [ ] Waktu akses per lesson
- [ ] Total waktu belajar (akumulasi)
- [ ] Tombol: kirim WA/email pengingat

#### AC-18: Analytics Konten
- [ ] Heatmap lesson: warna berdasarkan % completion rate
- [ ] Lesson dengan completion rate terendah → indikasi konten bermasalah
- [ ] Average time to complete per kursus

#### AC-19: Inactive Member Alert
- [ ] Tandai member yang tidak login > 7 hari (konfigurasi)
- [ ] Bulk action: kirim email/WA pengingat ke member inactive
- [ ] Segmentasi: baru beli (< 7 hari), aktif (< 30 hari), dormant (> 30 hari)

---

## 7. DEFINISI PRODUK & STATUS

### 7.1 Jenis Produk
| Tipe | Deskripsi | Ongkir | Akses Digital |
|---|---|---|---|
| `PHYSICAL` | Produk fisik, perlu kirim | Ya | Tidak |
| `DIGITAL` | File download / akses URL | Tidak | Ya |
| `ECOURSE` | Akses kursus di platform | Tidak | Ya (member area) |
| `BUNDLE` | Kombinasi fisik + digital/ecourse | Conditional | Ya |

### 7.2 Status Order
| Status | Deskripsi |
|---|---|
| `PENDING_PAYMENT` | Order dibuat, belum bayar |
| `PAID` | Pembayaran dikonfirmasi |
| `PROCESSING` | Sedang diproses/dikemas |
| `SHIPPED` | Dikirim, ada nomor resi |
| `COMPLETED` | Diterima buyer / akses aktif |
| `CANCELLED` | Dibatalkan (expired / manual) |
| `REFUNDED` | Refund diproses |

### 7.3 Status Member / Akses Kursus
| Status | Deskripsi |
|---|---|
| `ACTIVE` | Akses penuh, tidak ada batas waktu |
| `TRIAL` | Akses terbatas (beberapa lesson gratis) |
| `EXPIRED` | Akses habis (jika ada masa aktif) |
| `SUSPENDED` | Diblokir admin |

---

## 8. OUT OF SCOPE (v1.0)
- Affiliate / referral system
- Multi-bahasa (i18n)
- Mobile app native (iOS/Android)
- Live streaming / webinar
- Forum / komunitas member
- Multi-seller / marketplace
- Langganan (subscription) recurring
