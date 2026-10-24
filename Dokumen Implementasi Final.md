### Dokumen Implementasi Final

### Sistem Operasional Terintegrasi Pet Shop & Klinik Hewan

SRS & Technical Implementation Specification • Versi 1.1 • 25 Oktober 2026

Dokumen ini menjadi acuan implementasi bagi developer untuk membangun aplikasi khusus dari nol, menggunakan website responsif dan deployment cloud/VPS.

Keputusan bisnis yang sudah ditetapkan menjadi aturan dasar sistem:

* Inventori: stok bersama antara pet shop dan klinik, dengan pencatatan lokasi dan sumber pemakaian.

* Reservasi stok online: dilakukan setelah pembayaran dikonfirmasi.

* Rekam medis: hanya dapat disahkan dokter yang menangani atau dokter lain yang berwenang.

* Keuangan: kas, pengeluaran, piutang, HPP, dan laporan laba-rugi sederhana.

* Fulfillment: ambil di toko atau pengiriman yang diatur admin.

* Pemantauan rawat inap: customer melihat ringkasan harian dan riwayat perkembangan yang telah dibagikan.

* Deployment: cloud/VPS, dengan database terpusat dan dukungan multi-cabang di masa depan.

Prinsip yang paling penting: seluruh transaksi, stok, dan rekam medis harus konsisten meskipun pengguna mengaksesnya secara bersamaan dari perangkat berbeda.

### 1\. Arsitektur implementasi final

Website responsif

Owner • Dokter • Admin/Kasir • Customer

Backend API — Laravel

Auth • RBAC • Business rules • Validasi • Audit log

PostgreSQL

Transaksi dan data utama

Private object storage

Foto, dokumen, bukti bayar

POS & perangkat

Barcode, printer struk

Background jobs

Notifikasi, pengingat, ekspor

### Stack yang ditetapkan

| Komponen        | Pilihan                                             |
| --------------- | --------------------------------------------------- |
| Frontend        | Next.js + TypeScript                                |
| UI              | Tailwind CSS + komponen UI reusable                 |
| Backend         | Laravel REST API                                    |
| Database        | PostgreSQL                                          |
| Cache/queue     | Redis dan worker antrean                            |
| File            | Object storage privat                               |
| Web server      | Nginx + PHP-FPM untuk API                           |
| Deployment      | VPS/cloud dengan HTTPS                              |
| Version control | Git                                                 |
| Pengujian       | PHPUnit/Pest, API integration tests, Playwright     |
| Monitoring      | Application logs, error tracking, uptime monitoring |

Gunakan arsitektur modular monolith: satu aplikasi backend dengan batas modul yang jelas. Pembayaran, inventori, klinik, dan pesanan tetap berada dalam satu backend agar transaksi antarmodul dapat dijaga konsistensinya.

### Lingkup MVP

MVP harus mencakup:

* Login, role, permission, cabang, dan audit log.

* Pelanggan dan banyak hewan per pelanggan.

* POS, produk, harga, stok bersama, supplier, dan pembelian.

* Metode pembayaran manual dan verifikasi.

* Klinik, rekam medis, resep, dan tagihan.

* Rawat inap, grooming, dan penitipan.

* E-commerce, pembayaran manual, dan pemenuhan pesanan.

* Portal customer untuk hewan dan pemantauan perawatan.

* Kas, pengeluaran, piutang, HPP, dan laporan sederhana.

* Backup, keamanan, dan deployment produksi.

### 2\. Business rules final

Aturan berikut harus diimplementasikan pada backend, bukan hanya sebagai validasi antarmuka.

### BR-01 — Pengguna dan hak akses

* Setiap pengguna memiliki akun unik.

* Satu pengguna dapat mempunyai beberapa role atau penugasan cabang.

* Owner memiliki hak administratif penuh lintas modul.

* Dokter hanya mengakses fungsi klinis dan data pasien sesuai kewenangan.

* Admin mengakses pet shop dan fungsi administratif klinik sesuai permission.

* Customer hanya mengakses profilnya, hewan yang berhak dikelolanya, pesanan, pembayaran, dan catatan yang dibagikan.

* Setiap endpoint memeriksa autentikasi, permission, lingkup cabang, dan kepemilikan data.

* Pengubahan role atau permission oleh owner dicatat di audit log.

### BR-02 — Identitas pelanggan dan hewan

* Satu customer dapat mempunyai nol atau banyak hewan.

* Setiap hewan mempunyai identitas pasien yang unik dan terpisah.

* Hewan dapat dikaitkan dengan beberapa pemilik/pengasuh jika hubungan tersebut disetujui.

* Customer tidak otomatis berhak melihat hewan milik pengguna lain hanya karena mengetahui nomor rekam medis.

* Penggabungan data pelanggan ganda harus dilakukan melalui proses khusus dengan pencatatan audit.

### BR-03 — Stok bersama

* Pet shop dan klinik menggunakan inventori bersama pada tingkat bisnis, tetapi stok dicatat menurut cabang, lokasi penyimpanan, dan batch bila relevan.

* Stok fisik tidak boleh berkurang dua kali akibat transaksi POS dan pemakaian klinik.

* Pemakaian obat oleh klinik harus menghasilkan mutasi stok dengan referensi pasien, resep, atau tindakan terkait.

* Perpindahan antar-lokasi menghasilkan mutasi keluar dan masuk yang saling berpasangan.

* Transfer antar-cabang tidak boleh dianggap sebagai penjualan.

* Stok negatif dilarang secara default. Jika owner mengaktifkan pengecualian, tindakan harus memerlukan izin khusus dan tercatat.

* Produk yang memerlukan pelacakan batch atau kedaluwarsa harus diproses dengan batch yang sesuai.

### BR-04 — Reservasi pesanan online

* Saat customer membuat pesanan, stok belum dialokasikan sebagai reservasi final.

* Setelah pembayaran dikonfirmasi, backend mengunci stok yang relevan dan mencoba membuat reservasi.

* Jika stok tidak mencukupi pada saat konfirmasi, pesanan masuk status `STOCK_REVIEW` dan tidak boleh diproses menjadi siap kirim.

* Pesanan yang telah dikonfirmasi pembayarannya tetapi tidak dapat dipenuhi harus masuk alur penyelesaian, misalnya pengembalian dana atau persetujuan penggantian barang.

* Setelah reservasi berhasil, stok tersedia untuk pesanan lain berkurang.

* Reservasi dilepas ketika pesanan dibatalkan secara sah sebelum pemenuhan.

* Penjualan POS, pemakaian klinik, dan reservasi online harus bersaing terhadap saldo stok yang sama secara transaksional.

Catatan desain: kebijakan ini berarti pembayaran dapat diterima sebelum stok benar-benar dialokasikan. Karena itu, sistem harus mempunyai penanganan kegagalan alokasi dan refund yang eksplisit. Jika risiko tersebut terlalu tinggi, reservasi sementara sebelum pembayaran dapat dipertimbangkan di versi berikutnya, tetapi bukan perilaku default versi ini.

### BR-05 — Pembayaran manual

* Metode pembayaran diatur owner dan dapat diaktifkan per cabang.

* Metode meliputi tunai, transfer bank, QRIS, dan e-wallet manual.

* Bukti pembayaran bukan bukti otomatis bahwa pembayaran telah berhasil.

* Konfirmasi dilakukan oleh role yang memiliki `payment.confirm`.

* Owner dapat menentukan apakah konfirmasi dilakukan kasir, supervisor, atau owner sendiri.

* Pembuat transaksi tidak otomatis boleh menyetujui tindakan yang secara khusus memerlukan pemisahan tugas.

* Pembayaran parsial diperbolehkan jika diaktifkan.

* Jumlah pembayaran yang telah dikonfirmasi tidak boleh melebihi saldo tagihan tanpa aturan kelebihan bayar yang eksplisit.

* Refund harus merujuk pada pembayaran asli dan memerlukan izin yang sesuai.

* Semua konfirmasi, penolakan, dan refund harus dapat diaudit.

### BR-06 — POS dan transaksi

* Nomor invoice unik dan tidak dapat dipakai ulang.

* Harga, diskon, pajak, dan biaya disimpan sebagai snapshot saat transaksi dikonfirmasi.

* Transaksi selesai tidak dapat diedit bebas.

* Retur, pembatalan, dan refund menggunakan dokumen pembalik yang merujuk transaksi awal.

* Request transaksi harus mendukung idempotensi untuk mencegah duplikasi saat jaringan lambat atau tombol ditekan berulang.

* Penutupan shift mencatat kas awal, penjualan tunai, pengeluaran, kas aktual, dan selisih.

### BR-07 — Rekam medis

* Rekam medis memiliki status `DRAFT`, `SIGNED`, `AMENDED`, atau `VOIDED`.

* Catatan `SIGNED` tidak dapat ditimpa atau dihapus langsung.

* Pengesahan hanya boleh dilakukan dokter yang menangani atau dokter lain yang secara eksplisit memiliki kewenangan klinis.

* Koreksi harus menyimpan alasan, pelaku, waktu, dan referensi catatan sebelumnya.

* Admin boleh mengelola pendaftaran dan jadwal, tetapi tidak boleh membuat diagnosis atau mengesahkan resep tanpa kewenangan medis yang sesuai.

* Dokter dapat melihat informasi pasien yang dibutuhkan untuk perawatan sesuai penugasan dan kebijakan akses.

* Semua tindakan klinis yang relevan menyimpan identitas petugas dan waktu.

### BR-08 — Rawat inap dan informasi customer

* Setiap episode rawat inap memiliki waktu masuk, status, penempatan, catatan observasi, dan proses pulang.

* Kandang atau ruang tidak boleh ditempati dua episode yang bertabrakan, kecuali sistem secara eksplisit mendukung kapasitas bersama.

* Tugas perawatan memiliki waktu yang dijadwalkan, petugas, dan status pelaksanaan.

* Catatan harian atau perkembangan yang dibagikan customer harus ditandai sebagai informasi eksternal.

* Customer melihat ringkasan harian dan riwayat yang telah dibagikan, bukan seluruh catatan internal.

* Dokumen yang sudah dibagikan tetap memiliki riwayat publikasi dan dapat ditarik dari tampilan jika ada alasan yang sah, tanpa menghapus jejak audit internal.

### BR-09 — Grooming dan penitipan

* Booking memerlukan hewan, layanan, jadwal, dan status yang valid.

* Kapasitas staf, kandang, dan ruang harus diperiksa saat reservasi.

* Perubahan jadwal memicu validasi ulang kapasitas.

* Tambahan layanan yang menambah biaya memerlukan persetujuan customer.

* Penitipan mencatat waktu aktual check-in dan check-out untuk menghitung biaya berdasarkan kebijakan tarif.

* Perawatan dan instruksi khusus hewan harus dapat dilihat oleh staf yang bertugas.

### BR-10 — Kas dan laporan

* Kas masuk dan keluar harus mempunyai sumber transaksi atau alasan yang tercatat.

* Penjualan dan pembayaran adalah peristiwa berbeda: omzet tidak sama dengan penerimaan kas.

* HPP dihitung berdasarkan metode penilaian persediaan yang ditetapkan, misalnya moving weighted average.

* Pengeluaran operasional dicatat terpisah dari pembelian barang yang masuk ke persediaan.

* Laporan laba-rugi sederhana harus menyatakan periode, cabang, basis penghitungan, serta pengecualian yang relevan.

* Perubahan saldo kas tidak boleh dilakukan melalui pengubahan angka laporan secara manual; gunakan transaksi kas yang tercatat.

### 3\. Perubahan dan finalisasi skema database

Dibandingkan rancangan awal, keputusan terbaru membutuhkan beberapa perubahan penting agar stok bersama, pembayaran manual, multi-cabang, dan publikasi catatan rawat inap aman.

### 3.1 Perubahan struktur inventori

Tambahkan atau pastikan tabel berikut tersedia:

| Tabel                    | Kolom utama                                                         | Aturan                       |
| ------------------------ | ------------------------------------------------------------------- | ---------------------------- |
| inventory_locations      | id, branch_id, name, location_type, status                          | Lokasi penyimpanan fisik     |
| inventory_lots           | id, product_id, lot_number, expiry_date                             | Batch produk jika diperlukan |
| stock_balances           | product_id, location_id, lot_id, on_hand, reserved                  | Saldo stok                   |
| inventory_movements      | id, movement_type, quantity, reference_type, reference_id           | Buku mutasi stok             |
| stock_reservations       | id, order_id, product_id, location_id, quantity, status, expires_at | Reservasi pesanan            |
| inventory_transfers      | id, source_location_id, destination_location_id, status             | Transfer lokasi              |
| inventory_transfer_items | transfer_id, product_id, lot_id, quantity                           | Item transfer                |

`stock_balances` perlu memiliki constraint unik untuk kombinasi produk, lokasi, dan batch yang relevan.

Rumus saldo:

\\text{Available} = \\text{On hand} - \\text{Reserved}

Contoh: stok fisik 20 unit dan reservasi aktif 4 unit berarti stok tersedia untuk transaksi baru adalah 16 unit.

Gunakan `on_hand` untuk jumlah fisik yang tercatat, sedangkan `reserved` untuk kuantitas yang dialokasikan tetapi belum dikeluarkan secara fisik. Pemakaian atau penyerahan barang mengurangi `on_hand` dan melepaskan reservasi terkait dalam transaksi database yang sama.

Catatan implementasi: karena stok bersama dipakai oleh POS, klinik, dan e-commerce, semua jalur pengurangan stok harus melewati satu service inventori backend. Jangan membuat logika pengurangan stok berbeda di masing-masing controller.

### 3.2 Perubahan tabel pesanan dan pembayaran

| Tabel                 | Kolom tambahan/final                                                                                            |
| --------------------- | --------------------------------------------------------------------------------------------------------------- |
| orders                | order_number, customer_id, branch_id, fulfillment_type, payment_status, fulfillment_status, stock_review_status |
| order_items           | product_id, quantity, unit_price, discount_amount, tax_amount, line_total                                       |
| payments              | invoice_id, method_id, amount, status, reference_number, confirmed_by, confirmed_at                             |
| payment_confirmations | payment_id, reviewed_by, decision, reason, created_at                                                           |
| refunds               | payment_id, amount, status, approved_by, processed_at                                                           |
| idempotency_keys      | key, user_id, operation, response_hash, expires_at                                                              |

Pisahkan status pembayaran dari status pemenuhan pesanan. Misalnya, pesanan dapat berstatus pembayaran lunas tetapi belum dialokasikan stoknya.

Gunakan constraint dan indeks untuk:

* Nomor pesanan unik.

* Nomor invoice unik sesuai lingkup bisnis.

* Nominal pembayaran positif.

* Status dan waktu konfirmasi yang konsisten.

* Satu keputusan konfirmasi per proses review.

* Idempotency key unik sesuai pengguna dan operasi.

### 3.3 Perubahan rekam medis dan rawat inap

| Tabel                       | Kolom tambahan/final                                                                   |
| --------------------------- | -------------------------------------------------------------------------------------- |
| medical_records             | author_id, doctor_id, signed_by, signed_at, version, status                            |
| medical_record_amendments   | medical_record_id, previous_version, reason, amended_by, amended_at                    |
| hospitalizations            | pet_id, branch_id, admitted_at, discharged_at, status, attending_doctor_id             |
| care_observations           | hospitalization_id, recorded_by, observed_at, clinical_notes, customer_summary         |
| shared_updates              | hospitalization_id, observation_id, published_by, published_at, visibility, revoked_at |
| care_tasks                  | hospitalization_id, task_type, scheduled_at, assigned_to, status, completed_at         |
| care_units                  | branch_id, unit_code, unit_type, capacity, status                                      |
| hospitalization_assignments | hospitalization_id, care_unit_id, started_at, ended_at                                 |

Untuk `shared_updates`, simpan hubungan dengan catatan sumber serta snapshot atau versi konten yang dibagikan. Dengan demikian, perubahan catatan internal tidak diam-diam mengubah riwayat yang pernah dilihat pelanggan.

### 3.4 Keuangan sederhana

Tambahkan tabel:

| Tabel                                   | Kegunaan                                     |
| --------------------------------------- | -------------------------------------------- |
| cash_shifts                             | Shift kasir                                  |
| cash_movements                          | Kas masuk dan keluar                         |
| operating_expenses                      | Pengeluaran bisnis                           |
| supplier_bills                          | Tagihan supplier                             |
| customer_receivables                    | Jika piutang dikelola sebagai saldo khusus   |
| cost_layers atau inventory_cost_history | Riwayat biaya untuk perhitungan HPP          |
| refunds                                 | Pengembalian dana                            |
| financial_periods                       | Opsional, untuk periode laporan yang ditutup |

Jangan menyimpan saldo piutang atau stok sebagai angka bebas yang dapat diubah tanpa transaksi sumber. Jika saldo disimpan untuk performa, nilainya harus dapat direkonsiliasi dengan transaksi yang mendasarinya.

### 3.5 Standar database

* Gunakan primary key yang konsisten; UUID atau bigint dapat dipilih, tetapi jangan mencampur strategi tanpa alasan.

* Tambahkan `created_at`, `updated_at`, dan kolom status yang sesuai.

* Gunakan foreign key untuk relasi penting.

* Gunakan unique constraints untuk kode produk, nomor dokumen, dan identitas relevan.

* Gunakan transaksi database serta locking ketika mengubah saldo stok atau pembayaran.

* Simpan nominal uang menggunakan `NUMERIC` dengan presisi yang ditentukan.

* Simpan waktu dalam UTC dan konversikan saat ditampilkan sesuai zona waktu cabang.

* Buat indeks pada `branch_id`, `customer_id`, `pet_id`, `status`, `created_at`, dan foreign key yang sering digunakan.

* Gunakan migration versioning; perubahan struktur database harus dapat ditinjau dan diuji.

### 4\. Kontrak API inti

Gunakan REST API dengan prefix `/api/v1`. Endpoint berikut merupakan kontrak yang disarankan, bukan klaim bahwa API sudah dibangun.

Semua endpoint yang memerlukan login wajib memvalidasi identitas, permission, dan lingkup data. API publik seperti katalog tetap perlu rate limiting dan validasi input.

### 4.1 Autentikasi, pengguna, dan pengaturan

| Method | Endpoint                              | Fungsi                       | Permission               |
| ------ | ------------------------------------- | ---------------------------- | ------------------------ |
| POST   | /api/v1/auth/login                    | Login                        | Publik, dibatasi rate    |
| POST   | /api/v1/auth/logout                   | Logout                       | Authenticated            |
| GET    | /api/v1/auth/me                       | Profil dan permission aktif  | Authenticated            |
| GET    | /api/v1/users                         | Daftar pengguna              | users.view               |
| POST   | /api/v1/users                         | Membuat pengguna             | users.create             |
| PATCH  | /api/v1/users/{id}                    | Memperbarui pengguna         | users.update             |
| GET    | /api/v1/roles                         | Daftar role                  | roles.view               |
| PUT    | /api/v1/roles/{id}/permissions        | Mengatur permission          | users.permissions.manage |
| GET    | /api/v1/branches                      | Daftar cabang yang diizinkan | branches.view            |
| GET    | /api/v1/settings/payment-methods      | Metode pembayaran            | settings.payment.view    |
| POST   | /api/v1/settings/payment-methods      | Membuat metode pembayaran    | settings.payment.manage  |
| PATCH  | /api/v1/settings/payment-methods/{id} | Mengubah konfigurasi metode  | settings.payment.manage  |

Konfigurasi rekening dan gambar QRIS tidak boleh ikut terkirim ke endpoint publik yang tidak memerlukannya. Data yang dipublikasikan harus dibatasi pada informasi pembayaran yang memang perlu diketahui customer.

### 4.2 Pelanggan, hewan, dan klinik

| Method | Endpoint                                     | Fungsi                                                |
| ------ | -------------------------------------------- | ----------------------------------------------------- |
| GET    | /api/v1/customers                            | Pencarian pelanggan                                   |
| POST   | /api/v1/customers                            | Registrasi pelanggan                                  |
| GET    | /api/v1/customers/{id}/pets                  | Hewan milik pelanggan                                 |
| POST   | /api/v1/customers/{id}/pets                  | Menambahkan hewan                                     |
| GET    | /api/v1/pets/{id}                            | Detail pasien hewan                                   |
| POST   | /api/v1/appointments                         | Membuat appointment                                   |
| PATCH  | /api/v1/appointments/{id}/status             | Check-in, selesai, batal, dan transisi yang diizinkan |
| POST   | /api/v1/medical-records                      | Membuat draft rekam medis                             |
| GET    | /api/v1/pets/{id}/medical-records            | Riwayat medis sesuai izin                             |
| POST   | /api/v1/medical-records/{id}/sign            | Mengesahkan catatan medis                             |
| POST   | /api/v1/medical-records/{id}/amendments      | Mengajukan/membuat amendemen                          |
| POST   | /api/v1/prescriptions                        | Membuat resep                                         |
| POST   | /api/v1/prescriptions/{id}/dispense          | Memproses penyerahan obat                             |
| POST   | /api/v1/hospitalizations                     | Penerimaan rawat inap                                 |
| POST   | /api/v1/hospitalizations/{id}/observations   | Menambahkan observasi                                 |
| POST   | /api/v1/hospitalizations/{id}/tasks          | Menjadwalkan tugas perawatan                          |
| POST   | /api/v1/hospitalizations/{id}/shared-updates | Menerbitkan catatan untuk customer                    |
| POST   | /api/v1/hospitalizations/{id}/discharge      | Memproses pasien pulang                               |

Endpoint `sign`, `amendments`, `dispense`, dan `discharge` harus memiliki validasi domain tersendiri. Tidak cukup sekadar memperbarui kolom status.

### 4.3 POS, inventori, dan pembelian

| Method | Endpoint                                 | Fungsi                                           |
| ------ | ---------------------------------------- | ------------------------------------------------ |
| GET    | /api/v1/products                         | Pencarian produk                                 |
| POST   | /api/v1/products                         | Membuat produk                                   |
| PATCH  | /api/v1/products/{id}                    | Memperbarui produk                               |
| GET    | /api/v1/inventory/balances               | Saldo stok menurut lokasi                        |
| GET    | /api/v1/inventory/movements              | Riwayat mutasi                                   |
| POST   | /api/v1/inventory/adjustments            | Penyesuaian stok berizin                         |
| POST   | /api/v1/inventory/transfers              | Membuat transfer                                 |
| POST   | /api/v1/inventory/transfers/{id}/receive | Menerima transfer                                |
| POST   | /api/v1/purchase-orders                  | Membuat pesanan pembelian                        |
| POST   | /api/v1/goods-receipts                   | Mencatat penerimaan barang                       |
| POST   | /api/v1/cash-shifts/open                 | Membuka shift                                    |
| POST   | /api/v1/cash-shifts/{id}/close           | Menutup shift                                    |
| POST   | /api/v1/sales                            | Membuat transaksi POS                            |
| POST   | /api/v1/sales/{id}/complete              | Menyelesaikan transaksi sesuai status pembayaran |
| POST   | /api/v1/sales/{id}/returns               | Mengajukan atau memproses retur                  |

Semua operasi yang membuat transaksi keuangan atau mengubah stok harus menggunakan idempotency key ketika sesuai. Operasi mutasi harus menghasilkan referensi dokumen yang dapat ditelusuri.

### 4.4 Pembayaran dan pesanan online

| Method | Endpoint                                | Fungsi                                   |
| ------ | --------------------------------------- | ---------------------------------------- |
| POST   | /api/v1/orders                          | Membuat pesanan online                   |
| GET    | /api/v1/orders/{id}                     | Detail pesanan sesuai kepemilikan        |
| POST   | /api/v1/orders/{id}/payment-submissions | Mengirim informasi atau bukti pembayaran |
| POST   | /api/v1/payments/{id}/confirm           | Mengonfirmasi pembayaran                 |
| POST   | /api/v1/payments/{id}/reject            | Menolak pembayaran dengan alasan         |
| POST   | /api/v1/payments/{id}/refunds           | Memulai refund                           |
| POST   | /api/v1/refunds/{id}/approve            | Menyetujui refund                        |
| POST   | /api/v1/orders/{id}/fulfill             | Memulai proses pemenuhan                 |
| POST   | /api/v1/orders/{id}/dispatch            | Mencatat pengiriman                      |
| POST   | /api/v1/orders/{id}/complete            | Menandai pesanan selesai                 |
| GET    | /api/v1/invoices/{id}                   | Detail invoice yang diizinkan            |

Konfirmasi pembayaran harus dilakukan dalam satu alur transaksi backend yang aman:

* Kunci pembayaran dan periksa status terkini.

* Pastikan pembayaran belum pernah dikonfirmasi.

* Simpan keputusan verifikasi dan identitas reviewer.

* Perbarui status pembayaran dan invoice.

* Jika pembayaran pesanan online telah lunas, alokasikan stok secara atomik.

* Jika stok tidak mencukupi, tandai `STOCK_REVIEW` dan buat tugas penyelesaian.

* Commit perubahan bersama audit log dan event yang diperlukan.

Jika stok gagal dialokasikan, jangan menampilkan pesanan sebagai siap dipenuhi.

### 4.5 Laporan dan portal customer

| Method | Endpoint                           | Fungsi                                 |
| ------ | ---------------------------------- | -------------------------------------- |
| GET    | /api/v1/reports/sales              | Laporan penjualan                      |
| GET    | /api/v1/reports/inventory          | Laporan stok                           |
| GET    | /api/v1/reports/clinic             | Laporan klinik                         |
| GET    | /api/v1/reports/cash-flow          | Arus kas                               |
| GET    | /api/v1/reports/profit-loss        | Laba-rugi sederhana                    |
| GET    | /api/v1/portal/me/pets             | Hewan customer                         |
| GET    | /api/v1/portal/me/orders           | Pesanan customer                       |
| GET    | /api/v1/portal/me/invoices         | Tagihan customer                       |
| GET    | /api/v1/portal/me/hospitalizations | Episode rawat inap yang boleh dilihat  |
| GET    | /api/v1/portal/me/shared-updates   | Catatan rawat inap yang dipublikasikan |

Semua endpoint daftar sebaiknya mendukung pagination, sorting terbatas, dan filter yang diizinkan. API tidak boleh menerima nama kolom bebas untuk sorting atau filter tanpa validasi.

### Format respons dan error

Gunakan format konsisten, misalnya:

```json
{
  "data": {
    "id": "order-id",
    "status": "PENDING_PAYMENT"
  },
  "meta": {
    "request_id": "request-id"
  }
}
```

Format error:

```json
{
  "error": {
    "code": "INSUFFICIENT_STOCK",
    "message": "Stok tidak mencukupi.",
    "request_id": "request-id"
  }
}
```

Gunakan status HTTP yang tepat, seperti `200`, `201`, `204`, `400`, `401`, `403`, `404`, `409`, `422`, dan `500`. Pesan error tidak boleh membocorkan stack trace, SQL, kredensial, atau informasi sensitif.

### 5\. Acceptance criteria final

Setiap fitur harus mempunyai kriteria yang dapat diuji secara objektif. Tabel berikut menjadi dasar persetujuan hasil pekerjaan developer.

### 5.1 Autentikasi dan otorisasi

| ID         | Kriteria penerimaan                                                                                  |
| ---------- | ---------------------------------------------------------------------------------------------------- |
| AC-AUTH-01 | Pengguna yang belum login tidak dapat mengakses endpoint privat.                                     |
| AC-AUTH-02 | Admin yang mencoba mengakses endpoint pengaturan owner menerima 403 Forbidden.                       |
| AC-AUTH-03 | Customer tidak dapat melihat pesanan atau rekam medis customer lain dengan mengganti ID di URL.      |
| AC-AUTH-04 | Pengguna yang dinonaktifkan tidak dapat membuat sesi baru; sesi yang masih aktif dapat dicabut.      |
| AC-AUTH-05 | Perubahan role dan permission langsung berlaku sesuai kebijakan sesi serta tercatat dalam audit log. |

### 5.2 POS dan inventori

| ID        | Kriteria penerimaan                                                                                          |
| --------- | ------------------------------------------------------------------------------------------------------------ |
| AC-POS-01 | Kasir dapat mencari produk, scan barcode, mengubah kuantitas, menghitung total, dan menyelesaikan transaksi. |
| AC-POS-02 | Harga dan diskon tersimpan sebagai snapshot dan tidak berubah ketika harga master diubah kemudian.           |
| AC-INV-01 | Penjualan, penerimaan pembelian, retur, pemakaian obat, dan transfer memperbarui stok sesuai aturan.         |
| AC-INV-02 | Dua transaksi bersamaan tidak dapat menggunakan unit stok yang sama melebihi jumlah tersedia.                |
| AC-INV-03 | Pengulangan request dengan idempotency key yang sama tidak menciptakan transaksi atau mutasi stok ganda.     |
| AC-INV-04 | Riwayat mutasi menunjukkan dokumen sumber, pelaku, kuantitas, lokasi, dan waktu.                             |
| AC-INV-05 | Sistem menolak stok negatif kecuali pengecualian diaktifkan secara eksplisit dan diotorisasi.                |

### 5.3 Pembayaran dan e-commerce

| ID        | Kriteria penerimaan                                                                                              |
| --------- | ---------------------------------------------------------------------------------------------------------------- |
| AC-PAY-01 | Owner dapat menambah, mengubah, menonaktifkan, dan mengatur cakupan cabang metode pembayaran.                    |
| AC-PAY-02 | Customer dapat mengirim bukti pembayaran, tetapi tidak dapat menandai pembayaran sendiri sebagai lunas.          |
| AC-PAY-03 | Hanya role yang diizinkan yang dapat mengonfirmasi pembayaran.                                                   |
| AC-PAY-04 | Konfirmasi pembayaran tidak dapat diproses dua kali akibat request bersamaan.                                    |
| AC-ORD-01 | Pesanan belum memperoleh reservasi final sebelum pembayaran dikonfirmasi.                                        |
| AC-ORD-02 | Jika stok tidak mencukupi setelah pembayaran dikonfirmasi, pesanan menjadi STOCK_REVIEW dan tidak dapat dikirim. |
| AC-ORD-03 | Pembatalan yang sah melepaskan reservasi dan mencatat mutasi yang diperlukan.                                    |
| AC-ORD-04 | Customer dapat melihat status pesanan, metode fulfillment, dan invoice miliknya.                                 |
| AC-ORD-05 | Pesanan yang dikirim memiliki pencatatan petugas, waktu, dan informasi pengiriman yang tersedia.                 |

### 5.4 Klinik dan rawat inap

| ID        | Kriteria penerimaan                                                                                               |
| --------- | ----------------------------------------------------------------------------------------------------------------- |
| AC-MED-01 | Dokter dapat membuat draft rekam medis untuk pasien yang berwenang ditanganinya.                                  |
| AC-MED-02 | Rekam medis yang telah disahkan tidak dapat ditimpa secara langsung.                                              |
| AC-MED-03 | Amendemen memiliki alasan, pelaku, waktu, dan versi sebelumnya.                                                   |
| AC-MED-04 | Resep, penyerahan obat, mutasi stok, dan tagihan dapat ditelusuri tanpa dianggap sebagai satu kejadian yang sama. |
| AC-HOS-01 | Pasien rawat inap mempunyai riwayat observasi dengan pencatat dan waktu.                                          |
| AC-HOS-02 | Kandang tidak dapat dialokasikan untuk dua pasien yang saling bertabrakan dalam waktu.                            |
| AC-HOS-03 | Customer hanya dapat melihat ringkasan dan riwayat yang telah dipublikasikan untuk hewannya.                      |
| AC-HOS-04 | Catatan yang pernah dibagikan memiliki riwayat publikasi dan pencatat.                                            |
| AC-HOS-05 | Proses pulang tidak dapat diselesaikan sebelum validasi discharge yang diwajibkan.                                |

### 5.5 Keuangan, laporan, dan infrastruktur

| ID        | Kriteria penerimaan                                                                                 |
| --------- | --------------------------------------------------------------------------------------------------- |
| AC-FIN-01 | Kasir dapat membuka dan menutup shift dengan perhitungan kas aktual serta selisih.                  |
| AC-FIN-02 | Laporan membedakan omzet, pembayaran, pengeluaran, HPP, piutang, dan laba kotor.                    |
| AC-FIN-03 | Perubahan transaksi setelah selesai masuk melalui proses koreksi, retur, atau refund yang tercatat. |
| AC-SEC-01 | Pengujian akses lintas cabang tidak menemukan kebocoran data.                                       |
| AC-SEC-02 | File privat tidak dapat diunduh tanpa otorisasi yang valid.                                         |
| AC-OPS-01 | Backup database dan file berhasil dipulihkan pada lingkungan pengujian.                             |
| AC-OPS-02 | Website berfungsi pada ukuran layar desktop, tablet, dan HP.                                        |
| AC-OPS-03 | Error kritis tercatat dan dapat ditelusuri melalui request ID tanpa mencatat rahasia pengguna.      |

Kriteria ini perlu dilengkapi dengan hasil pengujian dan bukti penerimaan dari perwakilan owner, admin, dokter, dan customer.

### 6\. Rencana pengujian

Pengujian harus dijalankan sejak awal pengembangan, bukan hanya menjelang peluncuran.

### 01

Unit

Unit test

Uji fungsi harga, diskon, perhitungan tagihan, validasi status, HPP, saldo stok, dan permission. Jalankan pada setiap perubahan kode yang relevan.

### 02

Integrasi

Integration test

Uji hubungan antara POS dan stok, resep dan pemakaian obat, invoice dan pembayaran, serta pesanan online dan reservasi. Gunakan database pengujian yang terisolasi.

### 03

API

API & authorization test

Uji setiap role, perubahan ID, akses lintas cabang, request berulang, pembayaran ganda, dan input tidak valid.

### 04

E2E

End-to-end test

Simulasikan alur nyata dari login, pendaftaran hewan, transaksi POS, pembayaran, resep, rawat inap, checkout, hingga laporan.

### 05

UAT

User Acceptance Testing

Owner, dokter, admin, dan customer perwakilan memverifikasi skenario operasional menggunakan data simulasi yang realistis.

### 6.1 Skenario uji wajib

Skenario A — Konflik stok

* Saldo awal 5 unit.

* POS mencoba menjual 4 unit.

* Pada waktu hampir bersamaan, pesanan online yang pembayarannya dikonfirmasi mencoba mengalokasikan 3 unit.

* Hasil yang benar: total alokasi tidak boleh melebihi 5 unit. Salah satu operasi dapat berhasil terlebih dahulu, sedangkan operasi lainnya ditolak atau masuk proses peninjauan stok.

Skenario B — Pembayaran ganda

* Dua request konfirmasi dikirim untuk pembayaran yang sama.

* Hasil yang benar: hanya satu konfirmasi yang mengubah status dan hanya satu alokasi stok yang dilakukan.

Skenario C — Stok habis setelah pembayaran

* Customer membayar pesanan.

* Pada saat konfirmasi, stok tidak lagi mencukupi.

* Hasil yang benar: status menjadi `STOCK_REVIEW`, pesanan tidak dapat dikirim, dan sistem menyediakan tindakan penyelesaian yang tercatat.

Skenario D — Perlindungan rekam medis

* Dokter A mengesahkan catatan.

* Admin mencoba mengubah diagnosis.

* Dokter yang tidak memiliki penugasan atau izin mencoba melakukan amendemen.

* Hasil yang benar: tindakan tanpa kewenangan ditolak, sedangkan amendemen yang sah mempertahankan catatan lama.

Skenario E — Isolasi data customer

* Customer A mencoba membaca invoice, rekam medis, dan status rawat inap milik Customer B.

* Hasil yang benar: akses ditolak tanpa membocorkan apakah dokumen tersebut ada.

Skenario F — Pembatalan dan refund

* Pesanan dibayar, stok dialokasikan, kemudian pesanan dibatalkan secara sah.

* Hasil yang benar: sistem mencatat pembatalan, melepaskan atau mengembalikan stok sesuai kondisi fisik, dan menjalankan refund melalui proses yang dapat diaudit.

Skenario G — Pemulihan backup

* Pulihkan database dan file ke lingkungan staging.

* Periksa keterkaitan invoice, pembayaran, stok, rekam medis, dan lampiran.

* Hasil yang benar: seluruh data penting konsisten dan aplikasi dapat digunakan kembali.

### 6.2 Target pengujian

Target awal yang disarankan:

* Unit test untuk aturan bisnis kritis, dengan sasaran cakupan tinggi pada modul pembayaran, stok, dan otorisasi.

* Seluruh acceptance criteria kritis harus lulus.

* Tidak ada kerentanan kritis atau tinggi yang belum ditangani saat peluncuran.

* Uji beban berdasarkan perkiraan pengguna serentak dan transaksi puncak yang ditetapkan owner.

* Uji backup dan restore secara nyata, bukan sekadar memastikan file backup dibuat.

* Uji lintas browser dan ukuran layar yang relevan.

Persentase code coverage bukan satu-satunya ukuran kualitas. Pengujian harus memprioritaskan kegagalan yang dapat menyebabkan kehilangan uang, ketidaksesuaian stok, kebocoran data, atau kesalahan rekam medis.

### 7\. Deployment cloud/VPS dan operasional produksi

### 7.1 Struktur environment

Pisahkan lingkungan berikut:

| Environment | Tujuan                    | Data                     |
| ----------- | ------------------------- | ------------------------ |
| Development | Pengembangan fitur        | Data dummy               |
| Staging     | UAT dan simulasi produksi | Data uji yang disamarkan |
| Production  | Operasional bisnis        | Data aktual              |

Jangan menggunakan database produksi untuk eksperimen atau pengujian developer.

### 7.2 Rancangan deployment

Pengguna melalui HTTPS

Nginx / Reverse Proxy

TLS • Routing • Rate limiting

Next.js

Frontend

Laravel

Backend API

PostgreSQL

Storage & Redis

Background worker memproses antrean, pengingat, dan tugas yang tidak harus berjalan dalam request pengguna.

### 7.3 Standar operasional

* Domain dengan HTTPS dan sertifikat yang diperbarui otomatis.

* Database tidak terbuka langsung ke internet publik.

* Kredensial disimpan pada secret manager atau konfigurasi environment yang aman.

* Backup database dan file terjadwal dengan salinan di lokasi terpisah.

* Monitoring penggunaan CPU, RAM, disk, database, antrean, dan error aplikasi.

* Log aplikasi tanpa password, token, atau informasi sensitif yang tidak diperlukan.

* Proses deployment menggunakan migrasi database yang ditinjau dan rencana rollback.

* Worker antrean, scheduler, dan tugas backup diawasi agar kegagalannya terdeteksi.

* Pembaruan dependensi dan patch keamanan dilakukan berkala.

* Sediakan prosedur ketika VPS atau internet bermasalah, termasuk cara staf mencatat transaksi sementara dan merekonsiliasinya setelah sistem pulih.

### 7.4 Target pemulihan

Tetapkan target awal berikut sebagai usulan operasional:

| Parameter       | Target awal                                                                        |
| --------------- | ---------------------------------------------------------------------------------- |
| RPO             | Maksimal kehilangan data 15 menit                                                  |
| RTO             | Layanan kritis pulih dalam 4 jam                                                   |
| Backup database | Berkala, dengan target interval maksimal 15 menit untuk backup/log yang diperlukan |
| Backup file     | Harian dan setelah perubahan penting sesuai kebutuhan                              |
| Uji restore     | Minimal bulanan dan sebelum perubahan infrastruktur besar                          |
| Monitoring      | Notifikasi untuk error kritis, disk hampir penuh, dan kegagalan backup             |

Target tersebut bukan jaminan otomatis. RPO 15 menit memerlukan strategi backup dan pemulihan point-in-time yang sesuai, sedangkan RTO 4 jam memerlukan prosedur, infrastruktur, dan personel yang siap menjalankannya.

### 7.5 Strategi saat internet terputus

Versi awal sebaiknya menggunakan model online-first dengan transaksi yang diproses server. POS tidak boleh mengklaim transaksi offline berhasil jika belum ada mekanisme sinkronisasi yang teruji.

Jika operasi tanpa internet menjadi kebutuhan penting, implementasikan mode offline secara khusus pada tahap lanjutan, dengan:

* Penyimpanan transaksi lokal yang aman.

* ID transaksi unik dan idempotensi.

* Sinkronisasi saat koneksi kembali.

* Penanganan konflik stok dan harga.

* Rekonsiliasi pembayaran dan audit transaksi offline.

### 8\. Struktur pekerjaan developer

Agar pengerjaan dapat dipantau, backlog awal dapat dibagi menjadi kelompok berikut.

| Epic    | Deliverable utama                         |
| ------- | ----------------------------------------- |
| EPIC-01 | Auth, role, permission, cabang, audit     |
| EPIC-02 | Customer, hewan, profil dan kepemilikan   |
| EPIC-03 | Master produk, harga, supplier, inventori |
| EPIC-04 | POS, shift, pembayaran, retur             |
| EPIC-05 | Appointment, rekam medis, resep, tindakan |
| EPIC-06 | Rawat inap, observasi, tugas perawatan    |
| EPIC-07 | Grooming, penitipan, kapasitas            |
| EPIC-08 | Invoice, kas, pengeluaran, laporan        |
| EPIC-09 | E-commerce, reservasi stok, fulfillment   |
| EPIC-10 | Portal customer dan informasi rawat inap  |
| EPIC-11 | Keamanan, pengujian, backup, deployment   |

Setiap tiket pekerjaan harus memiliki:

* Tujuan dan deskripsi.

* Permission yang diperlukan.

* Perubahan database/API.

* Business rules yang terkait.

* Acceptance criteria.

* Unit test atau integration test yang diwajibkan.

* Status review dan persetujuan.

Definition of Done untuk sebuah fitur berarti kode selesai, migrasi tersedia, pengujian lulus, hak akses diperiksa, dokumentasi diperbarui, dan fitur telah diuji di staging.

### 9\. Checklist sebelum go-live

### Checklist peluncuran

0/12

Seluruh role dan permission sudah diuji.

POS, pembayaran manual, dan penutupan shift lulus UAT.

Stok bersama konsisten antara POS, klinik, dan e-commerce.

Rekam medis yang disahkan tidak bisa ditimpa langsung.

Alur rawat inap dan publikasi informasi customer telah diuji.

Grooming, penitipan, dan kapasitas booking telah diuji.

Retur, refund, piutang, dan laporan keuangan telah diverifikasi.

Backup dan pemulihan database serta file berhasil diuji.

Domain, HTTPS, monitoring, dan alert produksi aktif.

Data awal produk, harga, stok, pelanggan, dan akun telah diverifikasi.

Owner, dokter, dan admin sudah mendapatkan pelatihan.

Prosedur gangguan internet dan pemulihan layanan sudah tersedia.

Salin checklist

### 10\. Kesimpulan dan keputusan implementasi

Dokumen ini menetapkan fondasi implementasi dengan fokus pada tiga risiko terbesar bisnis Anda:

* Stok dan transaksi: stok bersama harus konsisten saat POS, klinik, dan pesanan online beroperasi bersamaan.

* Keamanan dan rekam medis: izin diperiksa di backend, catatan medis yang disahkan dilindungi, dan informasi rawat inap yang dibagikan customer dapat diaudit.

* Keuangan: pembayaran manual, konfirmasi, retur, refund, kas, dan laporan harus saling terhubung tanpa menyamakan omzet dengan kas masuk.

Urutan implementasi yang direkomendasikan adalah menyelesaikan fondasi data dan permission, lalu inventori serta POS, kemudian klinik dan rawat inap, setelah itu e-commerce dan portal customer, dan terakhir penyempurnaan laporan serta pengujian produksi.

Status dokumen: siap dijadikan acuan teknis untuk memulai desain UI, migration database, implementasi API, dan backlog sprint. Dokumen ini belum berarti aplikasi telah dibangun atau diuji; developer tetap perlu memvalidasi detail skema, mengimplementasikan aturan bisnis, dan memenuhi seluruh acceptance criteria sebelum sistem digunakan dalam operasional nyata.
