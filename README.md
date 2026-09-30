# Ticketing SaaS: Panduan Membaca Repo untuk Menyusun PRD

Dokumen ini adalah peta untuk memahami isi repository dan menyusun atau memperbarui Product Requirements Document (PRD). README ini tidak menggantikan spesifikasi sumber; gunakan dokumen yang dirujuk sebagai detail dan catat keputusan baru di sumber yang sesuai.

## Ringkasan Produk

Produk yang direncanakan adalah platform ticketing dan registrasi untuk organizer di Indonesia. Organizer membuat dan mengelola event; attendee membeli tiket atau mendaftar gratis; petugas gate memvalidasi QR; admin platform menangani KYC, payout, dan operasi platform.

Model MVP berfokus pada organizer sebagai pelanggan utama, payment lokal melalui Xendit, tiket QR, dan check-in lewat scanner web. Target pasar, model bisnis, metrik, dan operasi pilot dijabarkan di [model bisnis](docs/business/business-model.md).

## Cara Membaca untuk Menulis PRD

1. **Tetapkan tujuan dan batas produk.** Mulai dari [PRODUCT_SPEC.md](PRODUCT_SPEC.md), lalu baca [model bisnis](docs/business/business-model.md). Pisahkan keputusan MVP, sandbox/pilot, produksi, dan v2.
2. **Pahami aktor dan alur utama.** Baca [USER_FLOWS.md](USER_FLOWS.md), lalu spesifikasi [attendee dan guest](CUSTOMER_SPEC.md), [organizer](ORGANIZER_SPEC.md), dan [admin](ADMIN_SPEC.md). Catat kebutuhan, kondisi gagal, hak akses, serta hal yang masih berupa rekomendasi.
3. **Turunkan kebutuhan menjadi layar dan state.** Gunakan [route-and-state matrix](docs/ux/route-and-state-matrix.md), wireframe di folder `wireframe/`, lalu baca [audit wireframe](docs/product/2026-08-23-wireframe-audit.md). Satu layar dapat punya beberapa state yang berbeda.
4. **Validasi aturan dan batasan sistem.** Baca [TRD.md](TRD.md), [kontrak API](docs/api/v1-api-contract.md), [model data](docs/data/relational-data-model.md), [arsitektur](docs/architecture/cloudflare-first-architecture.md), dan [threat model](docs/security/threat-model.md). PRD sebaiknya menjelaskan perilaku dan kriteria penerimaan; detail implementasi tetap di dokumen teknis.
5. **Konfirmasi keputusan yang sudah disetujui.** Baca ADR di `docs/adr/`. ADR yang berstatus accepted adalah keputusan untuk konteks yang dinyatakan di ADR, bukan otomatis persetujuan untuk produksi.
6. **Susun prioritas dan fase.** Bandingkan [ROADMAP.md](ROADMAP.md) dengan [rencana implementasi Cloudflare](docs/plans/2026-08-23-cloudflare-first-implementation-plan.md). Tandai perbedaan fase, dependensi, dan gerbang rilis sebelum menetapkan prioritas PRD.

Saat menulis PRD, setiap kebutuhan sebaiknya memiliki aktor, pemicu, perilaku sistem, kondisi sukses/gagal, aturan bisnis, prioritas, dan kriteria penerimaan yang dapat diuji. Hubungkan kebutuhan ke flow, layar/state, dan kontrak terkait agar cakupan dapat ditelusuri.

## Peta Repository

### Spesifikasi Produk

- [PRODUCT_SPEC.md](PRODUCT_SPEC.md): visi, aktor, ruang lingkup MVP, pembayaran, entitas, non-goals, dan pertanyaan terbuka.
- [CUSTOMER_SPEC.md](CUSTOMER_SPEC.md): pengalaman attendee terdaftar dan guest, daftar layar, gap keamanan dan pengalaman, serta rekomendasi fase.
- [ORGANIZER_SPEC.md](ORGANIZER_SPEC.md): akses organizer, layar dashboard/event/KYC/refund, kebutuhan lintas layar, dan gap.
- [ADMIN_SPEC.md](ADMIN_SPEC.md): akses admin, prioritas halaman, operasi KYC/payout/audit, kontrol risiko, serta keputusan yang belum final.
- [USER_FLOWS.md](USER_FLOWS.md): alur auth, pembuatan event, checkout, check-in, operasi organizer, pembatalan event, attendee, dan admin.
- [ROADMAP.md](ROADMAP.md): fase produk dan exit criteria dari fondasi sampai persiapan launch.
- [TRD.md](TRD.md): kebutuhan teknis, rancangan entitas, endpoint, dan non-functional requirements.

### Dokumen Pendukung

- [Business model](docs/business/business-model.md): segmen awal, proposisi nilai, komisi, alur dana, metrik, pilot, dan risiko bisnis.
- [Wireframe audit](docs/product/2026-08-23-wireframe-audit.md): state yang bertabrakan, gap alur, dan kriteria sebelum UI dianggap siap diimplementasikan.
- [Route and state matrix](docs/ux/route-and-state-matrix.md): route utama dan state wajib untuk buyer, organizer, scanner, dan admin.
- [API contract v1](docs/api/v1-api-contract.md): rute API, autentikasi, idempotency, dan aturan respons server.
- [Relational data model](docs/data/relational-data-model.md): tabel inti, invariants, status machine, indeks, dan batas tanggung jawab data.
- [Cloudflare-first architecture](docs/architecture/cloudflare-first-architecture.md): komponen runtime dan keputusan arsitektur.
- [Threat model](docs/security/threat-model.md): aset, ancaman, kontrol, dan pengujian keamanan yang diperlukan sebelum pilot.
- [Implementation plan](docs/plans/2026-08-23-cloudflare-first-implementation-plan.md): milestone implementasi dan verification criteria; Milestone 0 adalah gerbang sebelum scaffold aplikasi.

### Architecture Decision Records (ADR)

- [ADR 0001: Xendit](docs/adr/0001-xendit-payment-gateway.md): Xendit sebagai gateway v1 dan webhook sebagai sumber status pembayaran.
- [ADR 0002: KYC dan payout](docs/adr/0002-organizer-kyc-and-payout-policy.md): KYC manual wajib sebelum publish/payout; payout manual mingguan setelah event.
- [ADR 0003: aturan komersial sandbox](docs/adr/0003-sandbox-commercial-policy.md): nilai sandbox untuk komisi, batas plan, pajak, refund, dan payout; bukan persetujuan komersial/produksi.
- [ADR 0004: transactional email](docs/adr/0004-transactional-email.md): adapter email dengan Resend dan pengiriman lewat Queue setelah transaksi tersimpan.

### Wireframe

File `.dc.html` adalah artefak desain/prototipe layar. Gunakan bersama flow dan route/state matrix; jangan menganggap setiap isi sebagai aturan final atau layar production-ready.

- `AccountSettings.dc.html`: pengaturan akun attendee.
- `AdminDashboard.dc.html`: ringkasan admin platform.
- `AdminOrganizerList.dc.html`: daftar dan operasi organizer oleh admin.
- `AttendeeDashboard.dc.html`: dashboard/tiket attendee.
- `AttendeeList.dc.html`: daftar peserta untuk organizer.
- `AuditLog.dc.html`: catatan aktivitas admin.
- `CancelEvent.dc.html`: konfirmasi pembatalan event dan refund.
- `Cart.dc.html`: cart dan data checkout.
- `CheckInPinEntry.dc.html`: akses scanner dengan PIN event.
- `CheckInScanner.dc.html`: scanner QR untuk gate.
- `EventForm.dc.html`: pembuatan/pengubahan event.
- `ForgotPassword.dc.html`: permintaan reset password.
- `Homepage.dc.html`: browse dan pencarian event.
- `Invoice.dc.html`: invoice/receipt.
- `KYCQueue.dc.html`: antrean review KYC admin.
- `Login.dc.html`: login.
- `Main.dc.html`: artboard utama/komposisi wireframe.
- `OrderConfirmation.dc.html`: konfirmasi order dan tiket.
- `OrderDetail.dc.html`: detail order dan tindakan refund.
- `OrganizerDetail.dc.html`: detail organizer untuk admin.
- `OrganizerSettings.dc.html`: profil, KYC, payout, dan pengaturan organizer.
- `PaymentFailed.dc.html`: state pembayaran gagal.
- `PaymentMethod.dc.html`: pemilihan metode pembayaran.
- `PaymentPending.dc.html`: state pembayaran tertunda.
- `PayoutQueue.dc.html`: antrean payout.
- `PlanUpgrade.dc.html`: perbandingan/upgrade plan.
- `PromoCodeForm.dc.html`: pembuatan promo code.
- `PublicEventPage.dc.html`: detail event publik.
- `Report.dc.html`: laporan event.
- `ResetPassword.dc.html`: penggantian password melalui tautan reset.
- `SalesSummary.dc.html`: ringkasan penjualan.
- `Signup.dc.html`: pendaftaran akun.
- `TicketTypeForm.dc.html`: pembuatan/penyuntingan jenis tiket.
- `canvas.json`: posisi dan ukuran artboard pada canvas.
- `ticketing-saas-wireframes.html`: export board wireframe gabungan.

## Hal yang Perlu Diputuskan atau Direkonsiliasi di PRD

Dokumen repo memuat beberapa konflik atau rekomendasi yang belum menjadi keputusan seragam. Jangan menyatukannya diam-diam:

- **Komisi saat refund:** [PRODUCT_SPEC.md](PRODUCT_SPEC.md) menyebut komisi tidak dibalik sebagai asumsi yang menunggu persetujuan; [model bisnis](docs/business/business-model.md) dan [ADR 0003](docs/adr/0003-sandbox-commercial-policy.md) menyebut komisi dibalik untuk refund penuh.
- **Akses order guest:** [CUSTOMER_SPEC.md](CUSTOMER_SPEC.md) merekomendasikan klaim otomatis guest order saat akun memakai email yang sama; [TRD.md](TRD.md) membatasi guest pada tautan email. Pilih aturan dan batas keamanan yang jelas.
- **Endpoint cancel:** [API contract](docs/api/v1-api-contract.md) menggunakan `POST /organizer/events/:id/cancel`, sedangkan [TRD.md](TRD.md) menuliskan `POST /events/:id/cancel`.
- **Admin 2FA:** [ADMIN_SPEC.md](ADMIN_SPEC.md) masih menyebutnya rekomendasi/keputusan terbuka; [ADR 0002](docs/adr/0002-organizer-kyc-and-payout-policy.md), arsitektur, dan threat model mensyaratkannya untuk aksi sensitif.
- **Batasan plan:** nilai 3 event aktif dan 500 tiket adalah default sandbox di [ADR 0003](docs/adr/0003-sandbox-commercial-policy.md); jangan menyatakannya sebagai kebijakan komersial produksi tanpa persetujuan.
- **Status “Built”:** CUSTOMER_SPEC, ORGANIZER_SPEC, dan ADMIN_SPEC memakai label status seperti “Built”, tetapi repository ini hanya berisi Markdown, HTML wireframe, dan data canvas; tidak ada source aplikasi atau konfigurasi build. Pastikan status tersebut berarti rancangan/dokumen, atau perbarui berdasarkan implementasi yang benar.
- **Kesiapan wireframe:** audit wireframe menyatakan board adalah peta cakupan, bukan UX spec siap implementasi. Pastikan state utama dan state error/loading/empty/unauthorized/destructive ditentukan sebelum menganggap layar selesai.
- **Keputusan sebelum implementasi:** implementation plan menetapkan Milestone 0 sebagai gerbang, termasuk spike password hashing yang aman dan keputusan komersial/teknis terkait.

## Batas Repo Saat Ini

Repository ini adalah paket spesifikasi, referensi arsitektur, ADR, dan wireframe. Belum ada source aplikasi, dependency manifest, test runner, atau perintah build. Gunakan [implementation plan](docs/plans/2026-08-23-cloudflare-first-implementation-plan.md) untuk menentukan pekerjaan implementasi setelah PRD dan gerbang keputusan disepakati.