# Spesifikasi Lengkap Sistem Operasional Pet Shop dan Klinik Hewan

Versi 3.0 · 25 Oktober 2026
Target deployment: Next.js di Vercel dengan Supabase sebagai backend terkelola.

Dokumen ini adalah spesifikasi tunggal dan final. Setiap keputusan yang sebelumnya terbuka sudah dikunci di Bagian 1, dan keputusan tersebut berlaku di seluruh dokumen. Dokumen ini hanya menjelaskan apa yang harus dibangun, aturan bisnisnya, dan kriteria penerimaannya. Dokumen ini tidak memuat urutan pengerjaan, fase, atau jadwal.

---

## 0. Prinsip Dasar

1. Satu aplikasi web responsif untuk komputer, tablet, dan ponsel, dengan satu basis data untuk seluruh cabang.
2. Stok pet shop dan klinik berada dalam satu inventori bersama per cabang. Stok dapat dipindahkan antarlokasi dan antarcabang melalui transfer.
3. Pembayaran dilakukan secara manual. Tidak ada payment gateway.
4. Setiap hak akses diperiksa di server dan di basis data. Menyembunyikan tombol di antarmuka bukan kontrol keamanan.
5. Setiap perubahan stok, kas, pembayaran, dan data klinis tercatat dalam jejak audit yang tidak dapat diubah.
6. Operasi yang mengubah stok, kas, atau status pembayaran dijalankan sebagai satu transaksi basis data, dilindungi idempotensi, dan tidak bergantung pada urutan request dari browser.
7. Aplikasi hanya memakai bahasa Indonesia.

---

## 1. Keputusan yang Dikunci

Bagian ini menutup seluruh keputusan terbuka. Tidak ada keputusan yang boleh ditetapkan ulang oleh developer tanpa perubahan dokumen ini.

| No | Topik | Keputusan |
|----|-------|-----------|
| K1 | Wali hewan | Satu hewan dapat memiliki lebih dari satu wali. Tepat satu wali berstatus utama. Wali utama menerima tagihan dan menyetujui biaya tambahan. Wali sekunder dapat melihat riwayat, membuat janji temu, dan menerima update, tetapi tidak menerima tagihan. |
| K2 | Akun pelanggan | Satu pelanggan dapat memiliki satu akun login. Satu akun login dapat ditautkan ke banyak hewan melalui tabel `pet_guardians`. Pelanggan tanpa akun (walk-in) dapat dibuat oleh staf dan ditautkan ke akun nanti. |
| K3 | Dokter dan remunerasi | Sistem tidak menghitung gaji atau komisi. Sistem menyediakan laporan layanan per dokter (jumlah kunjungan, tindakan, dan nilai layanan). |
| K4 | Harga dan katalog antarcabang | Master produk bersifat global. Harga jual, status jual POS, status jual online, dan stok minimum ditetapkan per cabang. Tidak ada harga fallback: setiap produk aktif di suatu cabang wajib memiliki harga di cabang tersebut. |
| K5 | Batch dan kedaluwarsa | Produk berkategori obat dan produk bertanda `track_lot = true` wajib memiliki nomor batch dan tanggal kedaluwarsa sejak penerimaan pertama. Pengambilan stok memakai FEFO (first expired, first out). |
| K6 | Pembayaran parsial dan piutang | POS dan pesanan online wajib lunas. Invoice layanan (klinik, rawat inap, grooming, penitipan) mendukung pembayaran parsial dan menghasilkan piutang sampai lunas. |
| K7 | Kasir dan metode pembayaran per cabang | Setiap cabang memiliki kasir dan shift sendiri. Metode pembayaran didefinisikan secara global, lalu diaktifkan per cabang. Rekening transfer dan instruksi QRIS manual disimpan per cabang. |
| K8 | Format nomor dokumen | Format nomor dokumen adalah `{KODE_CABANG}-{JENIS}-{TAHUN}-{NOMOR6}`. Nomor berurutan per cabang, per jenis, dan per tahun. Jenis dokumen: `POS`, `ONL`, `SVC`, `RFD`, `RTN`, `PO`, `TRF`. Contoh: `UTM-POS-2026-000145`. |
| K9 | Beban kapasitas | Target desain: 150 pengguna terautentikasi per hari, puncak 40 sesi bersamaan lintas cabang, dan maksimal 15 transaksi tulis bersamaan. Sistem dirancang dan diuji pada beban 2× target tersebut. |
| K10 | Wilayah dan zona waktu | Basis data Supabase berada di region `ap-southeast-1` (Singapura). Fungsi Vercel berjalan di region `sin1`. Zona waktu default cabang adalah `Asia/Jakarta`. Cabang dapat memakai `Asia/Makassar` atau `Asia/Jayapura`. Waktu disimpan UTC, ditampilkan dalam zona cabang. |
| K11 | Mata uang dan pajak | Mata uang hanya IDR. Uang disimpan dalam `NUMERIC(14,2)`, tetapi seluruh perhitungan transaksi dibulatkan ke rupiah penuh dengan pembulatan half-up. Sistem tidak menghitung PPN; harga yang diinput dianggap sudah final. Invoice mencantumkan teks "Harga sudah termasuk pajak sesuai ketentuan yang berlaku". |
| K12 | Biaya per unit | Biaya satuan persediaan (HPP) disimpan dalam `NUMERIC(14,4)`, bukan `NUMERIC(14,2)`, untuk menghindari pembulatan berlebih pada pembagian rata-rata tertimbang. |
| K13 | Peran sistem | Hanya empat peran sistem: `OWNER`, `DOCTOR`, `ADMIN`, `CUSTOMER`. Peran kustom tidak tersedia. Pemberian izin tambahan per pengguna dilakukan melalui `user_permission_grants`. |
| K14 | Dokter ganda | Pemilik dapat sekaligus berstatus dokter jika memiliki profil dokter dengan nomor STR. Tanpa profil dokter, pemilik hanya dapat membaca rekam medis dan tidak dapat menandatanganinya. |
| K15 | Rekam medis | Setiap kunjungan memiliki paling banyak satu rekam medis. Rekam medis yang sudah ditandatangani tidak dapat diubah. Koreksi hanya melalui amendemen oleh dokter penanda tangan asli. |
| K16 | Obat resep | Produk bertanda `is_prescription` hanya dapat dijual melalui pengeluaran klinik dengan resep bertanda tangan. Produk tersebut diblokir dari POS dan toko online. |
| K17 | Diskon | Diskon POS sampai 10% dapat diberikan oleh peran dengan izin `pos.discount.apply`. Diskon di atas 10% wajib otorisasi ulang pemilik dengan verifikasi kata sandi. Diskon invoice layanan hanya dapat diberikan oleh pemilik. |
| K18 | Persetujuan pengeluaran | Pengeluaran operasional di atas Rp 1.000.000 wajib disetujui pengguna lain yang memiliki `finance.expense.approve`. |
| K19 | Persetujuan refund | Setiap refund wajib disetujui pengguna lain yang memiliki `refund.approve`. Secara default hanya pemilik. Peminta refund tidak boleh menyetujui refund sendiri. |
| K20 | Penyesuaian stok | Penyesuaian stok tidak memerlukan persetujuan tambahan, tetapi wajib memakai kode alasan dan dicatat. Laporan penyesuaian bulanan dikirim ke pemilik. |
| K21 | Kedaluwarsa | Stok yang sudah lewat tanggal kedaluwarsa tidak dapat dijual maupun dikeluarkan untuk pasien. Stok yang kedaluwarsa dalam 90 hari ditandai di dashboard. |
| K22 | Autentikasi dan MFA | Login memakai email dan kata sandi. Akun staf dibuat melalui undangan dari pemilik. Pendaftaran publik hanya untuk pelanggan. MFA TOTP wajib untuk pengguna yang memiliki salah satu izin: `payment.confirm`, `refund.approve`, `settings.manage`, `finance.*`, `medical_record.sign`, `users.manage`. |
| K23 | Sesi | Sesi staf berakhir setelah 8 jam tanpa aktivitas. Sesi pelanggan berakhir setelah 30 hari tanpa aktivitas. Refresh token dirotasi. |
| K24 | Notifikasi | Saluran notifikasi adalah dalam aplikasi dan email. WhatsApp tidak termasuk dalam versi ini. Isi notifikasi tidak memuat data medis. |
| K25 | Pemilik pertama | Akun pemilik pertama dibuat melalui skrip setup yang hanya dapat dijalankan sekali di setiap lingkungan dan dicatat di audit. Setelah itu pemilik mengundang pengguna lain. |
| K26 | Undangan staf | Undangan berlaku 72 jam dan hanya dapat dipakai sekali. |
| K27 | Target pemulihan | RPO 15 menit. RTO 4 jam. Basis data memakai PITR. |
| K28 | Retensi data | Rekam medis dan data transaksi disimpan minimal 10 tahun sejak kunjungan terakhir atau transaksi terakhir. Dokumen yang dihapus dari bucket private tetap tersimpan di arsip terenkripsi selama masa retensi. Penghapusan data pribadi pelanggan dilakukan melalui anonimisasi, bukan penghapusan baris. |
| K29 | Pelacakan layanan | Layanan tanpa stok fisik (konsultasi, grooming, penitipan) tidak memiliki inventori. Layanan dikelola melalui `service_catalog`. |
| K30 | Komunikasi lintas cabang | Pesanan online ditetapkan ke satu cabang fulfillment. Stok diperiksa dan dialokasikan di cabang tersebut. |

---

## 2. Arsitektur Teknis

### 2.1 Stack

| Lapisan | Pilihan |
|---------|---------|
| Runtime | Node.js 22 LTS |
| Framework | Next.js App Router, TypeScript strict |
| UI | Tailwind CSS, komponen aksesibel (Radix UI) |
| Validasi | Zod di server dan klien untuk bentuk input yang sama |
| Form | React Hook Form |
| Basis data | Supabase PostgreSQL 15 |
| Autentikasi | Supabase Auth dengan email dan TOTP MFA |
| Otorisasi | RBAC dan Row Level Security (RLS) |
| Logika transaksi | Fungsi PostgreSQL (RPC) dengan `SECURITY DEFINER`, `search_path` tetap |
| Penyimpanan berkas | Supabase Storage dengan bucket privat |
| API | Next.js Route Handlers di `/api/v1` |
| Email transaksional | Resend |
| Pemantauan error | Sentry, dengan pembersihan data pribadi sebelum kirim |
| Pengujian | Vitest (unit), pgTAP (basis data dan RLS), Playwright (E2E) |
| Version control | GitHub |
| Hosting | Vercel |

### 2.2 Lingkungan

| Lingkungan | Branch | Basis data | URL |
|------------|--------|-----------|-----|
| Development | Lokal | Supabase CLI lokal | `localhost:3000` |
| Preview | Pull request | Proyek Supabase staging | Preview URL Vercel |
| Staging | `main` (otomatis) | Proyek Supabase staging | `staging.{domain}` |
| Production | `main` (setelah persetujuan) | Proyek Supabase production | `app.{domain}` |

Data staging bersifat sintetis. Data nyata tidak boleh disalin ke staging atau preview.

### 2.3 Variabel lingkungan

```dotenv
# Dapat diekspos ke browser
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY=

# Hanya server
SUPABASE_SECRET_KEY=
SUPABASE_DB_POOLER_URL=
RESEND_API_KEY=
SENTRY_DSN=
APP_CANONICAL_URL=
INVOICE_SIGNING_SECRET=
```

Aturan:
- Variabel dengan prefix `NEXT_PUBLIC_` tidak boleh berisi rahasia.
- `SUPABASE_SECRET_KEY` hanya dapat diimpor dari modul yang berada di folder `src/lib/server`. Linter harus menolak impor tersebut dari komponen klien.
- Environment variables dipisah untuk Development, Preview, Staging, dan Production di Vercel.
- Rahasia tidak boleh masuk ke repository, log, tangkapan layar, atau dokumen.
- Kunci yang pernah terekspos wajib dirotasi dalam 24 jam.

### 2.4 Aturan runtime

- Operasi yang mengubah data hanya boleh dijalankan oleh Route Handler atau Server Action. Komponen klien tidak mengakses tabel bisnis secara langsung untuk penulisan.
- Pembacaan data dari klien memakai klien Supabase dengan sesi pengguna sehingga RLS berlaku.
- Tidak ada pekerjaan latar di memori fungsi Vercel. Tugas terjadwal dijalankan melalui `pg_cron` di basis data atau Supabase Edge Function terjadwal, dengan pencatatan hasil di tabel `job_runs`.
- Batas waktu fungsi Vercel ditetapkan 10 detik untuk Route Handler. Operasi yang lebih lama dipecah atau dijalankan sebagai job.

---

## 3. Cakupan Modul

| Kode | Modul | Isi |
|------|-------|-----|
| MOD-01 | Identitas dan akses | Login, MFA, undangan, peran, izin, sesi, audit |
| MOD-02 | Cabang | Profil cabang, zona waktu, lokasi stok, kasir, rekening pembayaran |
| MOD-03 | Pelanggan dan hewan | Pelanggan, wali hewan, hewan, dokumen hewan |
| MOD-04 | Produk | SKU, barcode, kategori, satuan, harga per cabang, status jual |
| MOD-05 | Inventori | Lokasi, batch, saldo, mutasi, penyesuaian, transfer, HPP |
| MOD-06 | Pembelian | Supplier, pesanan pembelian, penerimaan, tagihan supplier |
| MOD-07 | POS | Shift kasir, penjualan, pembayaran terpisah, retur, invoice |
| MOD-08 | Toko online | Katalog, keranjang, pesanan, pembayaran manual, fulfillment |
| MOD-09 | Klinik | Jadwal dokter, janji temu, kunjungan, rekam medis, resep, pemakaian obat |
| MOD-10 | Rawat inap | Unit rawat, penempatan, observasi, tugas, update pelanggan |
| MOD-11 | Grooming dan penitipan | Katalog layanan, kapasitas, booking, check-in/out, layanan tambahan |
| MOD-12 | Tagihan layanan | Invoice layanan, pembayaran parsial, piutang |
| MOD-13 | Keuangan | Buku kas, pengeluaran, refund, tagihan supplier, HPP, periode |
| MOD-14 | Portal pelanggan | Hewan saya, janji temu, pesanan, invoice, rekam medis ringkas, update rawat inap |
| MOD-15 | Notifikasi | Dalam aplikasi dan email |
| MOD-16 | Laporan | Penjualan, stok, klinik, layanan, kas, laba rugi sederhana, piutang |
| MOD-17 | Pengaturan dan operasional | Metode pembayaran, kode alasan, template invoice, pengaturan cabang, job terjadwal |

Di luar cakupan: payment gateway, integrasi akuntansi penuh, perhitungan pajak, marketplace eksternal, aplikasi native, mode offline, payroll, AI diagnosis, integrasi alat laboratorium, WhatsApp, dan multi-mata uang.

---

## 4. Identitas, Peran, dan Otorisasi

### 4.1 Peran sistem dan izin default

Izin dinyatakan sebagai kode tetap dalam tabel `permissions`. Pemetaan peran ke izin ditetapkan dalam migration dan hanya dapat diubah melalui migration.

| Izin | OWNER | DOCTOR | ADMIN | CUSTOMER |
|------|:-----:|:------:|:-----:|:--------:|
| `branch.manage` | ✔ | | | |
| `users.manage` | ✔ | | | |
| `settings.manage` | ✔ | | | |
| `product.manage` | ✔ | | ✔ | |
| `price.manage` | ✔ | | | |
| `inventory.read` | ✔ | ✔ (klinik) | ✔ | |
| `inventory.read_cost` | ✔ | | | |
| `inventory.adjust` | ✔ | | (via grant) | |
| `inventory.transfer` | ✔ | | ✔ (minta) | |
| `inventory.transfer.approve` | ✔ | | | |
| `purchase.manage` | ✔ | | ✔ | |
| `pos.sale` | ✔ | | ✔ | |
| `pos.discount.apply` | ✔ | | ✔ | |
| `pos.discount.override` | ✔ | | | |
| `pos.return.request` | ✔ | | ✔ | |
| `cash_shift.manage` | ✔ | | ✔ | |
| `payment.review` | ✔ | | ✔ | |
| `payment.confirm` | ✔ | | ✔ | |
| `refund.request` | ✔ | | ✔ | |
| `refund.approve` | ✔ | | | |
| `refund.complete` | ✔ | | | |
| `customer.manage` | ✔ | | ✔ | |
| `pet.manage` | ✔ | | ✔ | |
| `appointment.manage` | ✔ | ✔ (miliknya) | ✔ | |
| `schedule.manage` | ✔ | | ✔ | |
| `grooming.manage` | ✔ | | ✔ | |
| `visit.manage` | ✔ | ✔ | | |
| `medical_record.read` | ✔ | ✔ (cabang) | | ✔ (miliknya) |
| `medical_record.write` | | ✔ (tugas) | | |
| `medical_record.sign` | | ✔ (tugas) | | |
| `medical_record.amend` | | ✔ (penanda asli) | | |
| `prescription.write` | | ✔ | | |
| `clinic.medication.use` | | ✔ | | |
| `hospitalization.manage` | ✔ | ✔ | ✔ | |
| `hospitalization.observe` | | ✔ | | |
| `hospitalization.publish_update` | | ✔ | | |
| `service_invoice.manage` | ✔ | | ✔ | |
| `service_invoice.discount` | ✔ | | | |
| `service_invoice.void` | ✔ | | | |
| `finance.cash.read` | ✔ | | | |
| `finance.expense.create` | ✔ | | ✔ | |
| `finance.expense.approve` | ✔ | | | |
| `finance.report.read` | ✔ | | | |
| `finance.period.close` | ✔ | | | |
| `report.sales.read` | ✔ | | ✔ | |
| `audit.read` | ✔ | | (terbatas) | |

Catatan:
- `(via grant)` berarti izin tersebut hanya diberikan kepada pengguna tertentu melalui `user_permission_grants` oleh pemilik. Setiap grant mencatat alasan, pemberi, dan tanggal kedaluwarsa opsional.
- `(miliknya)` untuk dokter berarti janji temu yang dibuat atau ditugaskan kepadanya.
- `(cabang)` berarti rekam medis dalam cabang yang ditugaskan kepada dokter.
- `(tugas)` berarti kunjungan yang ditugaskan kepada dokter tersebut.
- `(terbatas)` untuk admin pada `audit.read` berarti hanya entri dari cabang yang ditugaskan dan tidak mencakup entitas medis.
- Admin tidak memiliki `refund.approve`, `inventory.adjust`, atau akses data klinis.
- Pemilik memiliki `medical_record.read` untuk seluruh cabang. Pemilik tidak memiliki izin tanda tangan kecuali memiliki profil dokter.

### 4.2 Penugasan cabang

- `OWNER` memiliki akses ke seluruh cabang secara default.
- `DOCTOR`, `ADMIN` ditugaskan ke satu atau lebih cabang melalui `user_branch_access`.
- Setiap pemeriksaan akses cabang di server dan RLS memeriksa `user_branch_access` atau kepemilikan global.

### 4.3 Pemisahan tugas

- Peminta refund tidak boleh menyetujui refund yang sama.
- Pengaju pengeluaran di atas batas tidak boleh menyetujui pengeluarannya sendiri.
- Penerima transfer tidak boleh menjadi pengirim transfer yang sama dalam satu dokumen. Penerimaan transfer harus oleh pengguna cabang tujuan.
- Dokter yang menandatangani rekam medis tidak boleh menyetujui amendemen atas rekam medisnya sendiri jika tidak ada dokter lain yang menandatangani; amendemen dilakukan oleh dokter penanda asli saja.

### 4.4 Akun dan sesi

- Login memakai email dan kata sandi minimal 10 karakter.
- Setelah 5 kegagalan login dalam 15 menit untuk satu akun, akun dikunci sementara 15 menit.
- Reset kata sandi dikirim ke email terdaftar dan tautannya berlaku 60 menit.
- Pengguna nonaktif tidak dapat login dan semua sesinya dicabut saat dinonaktifkan.
- Pemilik dapat menonaktifkan pengguna dan melihat daftar sesi aktif.

### 4.5 Undangan staf

1. Pemilik membuat undangan dengan email, peran, dan cabang.
2. Sistem mengirim tautan undangan yang berlaku 72 jam.
3. Penerima membuat kata sandi dan wajib mengaktifkan MFA jika izinnya termasuk daftar K22.
4. Undangan yang sudah dipakai atau kedaluwarsa tidak dapat digunakan kembali.

---

## 5. Pelanggan, Hewan, dan Produk

### 5.1 Pelanggan dan wali

- Pelanggan dibuat oleh staf (walk-in) atau mendaftar sendiri (portal).
- Data minimal: nama, nomor telepon, email (opsional untuk walk-in), alamat.
- Pelanggan yang mendaftar sendiri wajib memverifikasi email.
- Jika staf membuat pelanggan walk-in dengan email yang sudah terdaftar pada akun lain, sistem menolak dan meminta staf menautkan melalui verifikasi.
- Pelanggan tidak dihapus secara fisik. Nonaktifasi dan anonimisasi sesuai K28.

### 5.2 Hewan

- Setiap hewan memiliki identitas, spesies, ras, jenis kelamin, status steril, tanggal lahir atau estimasi usia, berat terakhir, warna, nomor microchip (opsional), dan status meninggal.
- Spesies: `DOG`, `CAT`, `BIRD`, `RABBIT`, `REPTILE`, `OTHER`.
- Jenis kelamin: `MALE`, `FEMALE`, `UNKNOWN`.
- Hewan dengan status meninggal tidak dapat menerima kunjungan, booking, atau rawat inap baru, tetapi riwayatnya tetap dapat dibaca.

### 5.3 Wali hewan

- `pet_guardians` menghubungkan `pets` dan `customers` dengan kolom `is_primary`.
- Tepat satu wali utama untuk setiap hewan aktif. Indeks unik parsial menegakkan aturan ini.
- Mengganti wali utama dilakukan oleh admin atau pemilik dan dicatat di audit.
- Tagihan dan persetujuan biaya tambahan hanya dapat dilakukan oleh wali utama.
- Wali sekunder dapat melihat riwayat dan update, membuat janji temu, dan mengajukan booking grooming atau penitipan.

### 5.4 Produk

- `products` berisi SKU unik global, nama, kategori, tipe satuan (`COUNT` atau `WEIGHT`), satuan dasar, `track_lot`, `is_prescription`, dan status aktif.
- Kategori: `RETAIL`, `MEDICINE`, `CLINIC_SUPPLY`, `FOOD`, `ACCESSORY`, `OTHER`.
- `MEDICINE` otomatis memiliki `track_lot = true` dan `is_prescription` sesuai data produk.
- Satu produk dapat memiliki banyak barcode. Barcode unik global.
- Harga per cabang disimpan di `product_prices` yang bersifat append-only dengan `effective_from`. Harga aktif adalah baris terbaru yang `effective_from <= now()`.
- `product_branch_settings` menyimpan per cabang: `is_pos_sellable`, `is_online_sellable`, `min_stock`, dan `is_active`.
- Produk tidak aktif di suatu cabang tidak tampil di POS atau toko online cabang tersebut.
- Perubahan harga tidak mengubah transaksi lama.

### 5.5 Supplier

- Data supplier: nama, nama kontak, telepon, email, alamat, NPWP (opsional), syarat pembayaran dalam hari.
- Supplier dapat dinonaktifkan tetapi tidak dihapus.

---

## 6. Inventori

### 6.1 Lokasi dan batch

- Setiap cabang wajib memiliki tepat satu lokasi `RETAIL` dan tepat satu lokasi `CLINIC_PHARMACY`. Lokasi tambahan bertipe `WAREHOUSE` boleh ada.
- Setiap pasangan (produk, lokasi) memiliki satu baris `inventory_lots` dengan nomor batch `DEFAULT` untuk produk yang tidak memakai batch. Ini menghindari nilai NULL pada kunci unik.
- Untuk produk `track_lot = true`, setiap penerimaan membuat atau menambah batch dengan `lot_number` dan `expiry_date` wajib.
- Batch dengan `expiry_date < hari ini` tidak dapat dipakai untuk penjualan, pemakaian, atau transfer.

### 6.2 Saldo dan konsistensi

`stock_balances` memiliki kunci unik `(product_id, location_id, lot_id)`.

Constraint wajib:
- `on_hand >= 0`
- `reserved >= 0`
- `reserved <= on_hand`

Ketersediaan:

```
available = on_hand − reserved
```

Setiap perubahan saldo menghasilkan baris `inventory_movements` dalam transaksi yang sama. `inventory_movements` bersifat append-only.

Tipe mutasi: `PURCHASE_RECEIPT`, `SALE_POS`, `ORDER_FULFILLMENT`, `RESERVATION`, `RESERVATION_RELEASE`, `CLINIC_USE`, `CUSTOMER_RETURN_RESTOCK`, `CUSTOMER_RETURN_WRITE_OFF`, `SUPPLIER_RETURN`, `ADJUSTMENT`, `TRANSFER_OUT`, `TRANSFER_IN`, `TRANSFER_DISCREPANCY`, `EXPIRY_WRITE_OFF`.

Rekonsiliasi harian membandingkan jumlah `on_hand` dengan total `quantity_delta` di ledger. Selisih menghasilkan alert ke pemilik dan tidak diperbaiki otomatis.

### 6.3 Reservasi

- `stock_reservations` unik per `order_item_id`.
- Reservasi dibuat hanya saat pembayaran pesanan dikonfirmasi dan stok tersedia.
- Reservasi dilepas saat pesanan dibatalkan sebelum diserahkan.
- Saat pesanan diserahkan atau dikirim, reservasi dikonversi menjadi mutasi `ORDER_FULFILLMENT`: `reserved` dan `on_hand` berkurang bersamaan.

### 6.4 Penyesuaian

- Kode alasan tetap: `COUNT_CORRECTION`, `DAMAGED`, `EXPIRY_WRITE_OFF`, `LOST`, `FOUND`, `CLINIC_USE_CORRECTION`, `OTHER`.
- Alasan `OTHER` wajib catatan tambahan.
- Penyesuaian positif memakai biaya rata-rata saat ini. Jika biaya rata-rata belum ada, `unit_cost` wajib diisi.
- Penyesuaian negatif tidak boleh membuat `on_hand` negatif.
- Penyesuaian dilakukan melalui RPC `apply_inventory_adjustment`.

### 6.5 Stok opname

- Stok opname dilakukan dengan membuat sesi opname per lokasi. Staf mengisi jumlah fisik per produk dan batch.
- Sistem membuat penyesuaian untuk setiap selisih dengan kode `COUNT_CORRECTION`.
- Sesi yang sudah ditutup tidak dapat diubah.

### 6.6 Transfer

Transfer dapat terjadi antarlokasi dalam satu cabang atau antarcabang.

Alur:
1. Pengirim (izin `inventory.transfer`) membuat transfer `DRAFT` dengan lokasi asal, lokasi tujuan, dan item.
2. Pemilik menyetujui transfer antarcabang (`inventory.transfer.approve`). Transfer dalam cabang tidak memerlukan persetujuan.
3. Saat dikirim, sistem membuat mutasi `TRANSFER_OUT` dan mengurangi `on_hand` di lokasi asal. Status menjadi `IN_TRANSIT`.
4. Penerima di lokasi tujuan mencatat jumlah diterima per item. Sistem membuat mutasi `TRANSFER_IN`.
5. Selisih antara jumlah dikirim dan diterima membuat mutasi `TRANSFER_DISCREPANCY` dengan alasan wajib. Status menjadi `RECEIVED`.
6. Transfer yang masih `DRAFT` dapat dibatalkan. Transfer `IN_TRANSIT` tidak dapat dibatalkan, hanya diterima.

Barang dalam perjalanan tidak disimpan sebagai saldo. Jumlahnya dihitung dari `inventory_transfer_items` dengan status transfer `IN_TRANSIT`.

Biaya transfer mengikuti biaya rata-rata cabang asal dan diterapkan di cabang tujuan sebagai biaya rata-rata tertimbang.

### 6.7 Biaya persediaan (HPP)

- Metode: rata-rata tertimbang bergerak per produk per cabang. Metode ini dipakai untuk semua produk.
- Biaya disimpan di `product_branch_costs` dengan `avg_unit_cost NUMERIC(14,4)`.
- Setiap penerimaan barang memperbarui biaya:

```
qty_lama = total on_hand produk di cabang sebelum penerimaan
biaya_baru = (qty_lama × biaya_lama + qty_terima × biaya_terima) / (qty_lama + qty_terima)
```

- Jika `qty_lama <= 0`, `biaya_baru = biaya_terima`.
- Perhitungan dilakukan dalam transaksi yang mengunci baris `product_branch_costs` untuk produk dan cabang tersebut.
- Setiap penjualan dan pemakaian menyimpan `unit_cost_snapshot` pada item transaksi. Biaya snapshot tidak pernah berubah.
- Setiap perubahan biaya rata-rata dicatat di `cost_history`.

### 6.8 Pengambilan stok (FEFO)

Untuk produk `track_lot = true`, pengambilan stok dilakukan dari batch dengan urutan:
1. `expiry_date` terdekat (NULL paling akhir).
2. `created_at` batch paling awal.

Batch dengan `expiry_date < hari ini` dilewati. Pengambilan lintas batch diizinkan untuk satu item.

Untuk produk `track_lot = false`, pengambilan dilakukan dari baris `DEFAULT`.

---

## 7. Pembelian dan Penerimaan

### 7.1 Pesanan pembelian

- Status: `DRAFT`, `SUBMITTED`, `PARTIALLY_RECEIVED`, `RECEIVED`, `CANCELLED`.
- `DRAFT` dapat diubah. `SUBMITTED` dan setelahnya tidak dapat diubah; perubahan dilakukan dengan membatalkan dan membuat pesanan baru.
- Pesanan `CANCELLED` hanya dapat dibatalkan jika belum ada penerimaan.
- Nomor pesanan: `PO`.

### 7.2 Penerimaan barang

- Penerimaan dicatat per item dengan jumlah diterima, batch (wajib untuk `track_lot`), tanggal kedaluwarsa (wajib untuk `track_lot`), dan harga perolehan per unit.
- Penerimaan tidak boleh melebihi jumlah pesanan kecuali pemilik memberi izin melalui catatan dan alasan.
- Setiap penerimaan membuat `goods_receipts` dan mutasi `PURCHASE_RECEIPT`, serta memperbarui biaya rata-rata.
- Penerimaan dilakukan melalui RPC `receive_goods` yang atomik.
- Penerimaan tidak dapat dibatalkan. Kesalahan diperbaiki melalui penyesuaian atau retur supplier.

### 7.3 Retur supplier

- Retur supplier membuat mutasi `SUPPLIER_RETURN` dan mengurangi stok.
- Retur mengurangi nilai persediaan sesuai biaya rata-rata saat retur.
- Retur harus memiliki alasan dan referensi penerimaan asal.

### 7.4 Tagihan supplier

- `supplier_bills` dibuat dari penerimaan atau input manual dengan nomor faktur supplier, tanggal jatuh tempo, dan total.
- `supplier_payments` mencatat pembayaran terhadap tagihan.
- Tagihan dan penerimaan barang tercatat terpisah. Penerimaan tidak otomatis berarti tagihan lunas.
- Pembayaran supplier dicatat di buku kas cabang dengan `movement_type = SUPPLIER_PAYMENT`.

---

## 8. Point of Sale (POS)

### 8.1 Shift kasir

- Shift dibuka oleh kasir dengan saldo kas awal.
- Satu kasir hanya boleh memiliki satu shift terbuka pada satu waktu.
- Setiap penjualan, pembayaran, dan retur terkait ke shift aktif.
- Shift ditutup dengan kas fisik yang dihitung. Selisih wajib memiliki alasan.
- Shift yang sudah ditutup tidak dapat diubah.
- Penutupan menghasilkan rekap per metode pembayaran.

### 8.2 Penjualan

Status penjualan: `DRAFT`, `COMPLETED`, `PARTIALLY_RETURNED`, `RETURNED`, `VOIDED`.

Alur:
1. Kasir membuat draft dengan item (produk, jumlah) dan pelanggan opsional.
2. Draft tidak mereservasi stok. Draft yang tidak diselesaikan dalam 24 jam otomatis berubah menjadi `VOIDED` oleh job terjadwal.
3. Saat draft diubah, server menghitung ulang harga, diskon, dan total.
4. Kasir memilih pembayaran. Satu penjualan dapat dibayar dengan beberapa metode.
5. Penyelesaian (`complete_sale`) dijalankan sebagai satu transaksi.

Perhitungan:

```
line_gross = qty × unit_price
line_discount = sesuai tipe diskon (persen atau nominal)
line_total = ROUND(line_gross − line_discount, 0) (half-up)
subtotal = Σ line_gross
discount_total = Σ line_discount
grand_total = Σ line_total
```

Diskon:

- Tipe `PERCENT` atau `AMOUNT`.
- Nilai persen 0–10 untuk peran dengan `pos.discount.apply`.
- Nilai di atas 10% wajib otorisasi ulang pemilik dengan verifikasi kata sandi. Otorisasi dicatat sebagai `pos.discount.override` dan disimpan di audit.
- Diskon tidak boleh membuat total negatif.

Pembayaran:
- Metode diaktifkan per cabang: tunai, transfer, QRIS manual, dan metode tambahan yang didefinisikan pemilik.
- Jumlah pembayaran non-tunai harus tepat sama dengan jumlah yang dialokasikan ke metode tersebut.
- Pembayaran tunai boleh melebihi kebutuhan. Kembalian dihitung otomatis dan tidak dicatat sebagai pembayaran.
- Total pembayaran harus sama dengan `grand_total`.

Penyelesaian (`complete_sale`) melakukan dalam satu transaksi:
1. Validasi draft: status `DRAFT`, shift aktif, cabang sesuai, izin.
2. Hitung ulang harga dan diskon dari `product_prices` dan `product_branch_settings`.
3. Kunci baris `stock_balances` dan `product_branch_costs` terkait, diurutkan berdasarkan `(product_id, location_id, lot_id)`.
4. Pastikan batch tidak kedaluwarsa dan stok tersedia. Jika tidak cukup, rollback dengan `INSUFFICIENT_STOCK`.
5. Alokasikan batch dengan FEFO. Kurangi `on_hand`. Buat mutasi `SALE_POS`.
6. Simpan `unit_cost_snapshot` per item.
7. Buat `sale_payments` dan `cash_movements` per metode.
8. Berikan nomor invoice `POS` dari sequence cabang.
9. Ubah status menjadi `COMPLETED`.
10. Simpan hasil idempotensi.

Penjualan `COMPLETED` tidak dapat diedit. Koreksi dilakukan melalui retur.

Produk `is_prescription` tidak dapat ditambahkan ke draft.

### 8.3 Retur POS

- Retur hanya untuk item dari penjualan `COMPLETED` atau `PARTIALLY_RETURNED`.
- Jumlah retur per item tidak boleh melebihi jumlah yang belum diretur.
- Setiap item retur memiliki kondisi: `RESTOCK` (kembali ke stok) atau `DAMAGED` (dihapus dari stok).
- `RESTOCK` membuat mutasi `CUSTOMER_RETURN_RESTOCK` dengan biaya snapshot asal. `DAMAGED` membuat mutasi `CUSTOMER_RETURN_WRITE_OFF`.
- Retur membuat permintaan refund yang harus disetujui sesuai K19.
- Status penjualan menjadi `PARTIALLY_RETURNED` atau `RETURNED`.
- Retur tidak mengubah baris penjualan asal. Retur disimpan di `sales_returns` dan `sales_return_items`.

### 8.4 Void

- Hanya penjualan `DRAFT` yang dapat di-void oleh kasir.
- Penjualan `COMPLETED` tidak dapat di-void. Koreksi dilakukan melalui retur.

---

## 9. Toko Online dan Pembayaran Manual

### 9.1 Katalog dan keranjang

- Katalog menampilkan produk aktif dengan `is_online_sellable = true` di cabang fulfillment.
- Pelanggan memilih cabang fulfillment saat checkout.
- Keranjang disimpan di `customer_carts` dan `customer_cart_items`. Satu keranjang per pelanggan per cabang.
- Keranjang tidak mereservasi stok.
- Produk `is_prescription` tidak dapat masuk ke keranjang.

### 9.2 Pembuatan pesanan

Saat checkout, server:
1. Memvalidasi item, status produk, dan harga aktif.
2. Menghitung `subtotal`, `delivery_fee` (sesuai metode, lihat 9.5), dan `grand_total`.
3. Membuat `orders` dengan `payment_status = UNPAID` dan `fulfillment_status = NOT_STARTED`.
4. Menyimpan snapshot nama produk, SKU, harga, dan instruksi pembayaran aktif.
5. Tidak mengurangi atau mereservasi stok.

Pelanggan dapat membatalkan pesanan selama `payment_status` masih `UNPAID` atau `SUBMITTED`.

### 9.3 Instruksi pembayaran

- Setiap cabang memiliki satu atau lebih rekening atau metode manual aktif (`branch_payment_accounts`).
- Instruksi yang ditampilkan kepada pelanggan adalah snapshot pada saat pesanan dibuat.
- Mengubah rekening tidak mengubah instruksi pesanan yang sudah ada.

### 9.4 Pembayaran

- Pelanggan mengunggah bukti transfer dan memilih metode. Bukti disimpan di bucket `payment-proofs`.
- Setiap pengiriman membuat `payments` dengan status `SUBMITTED` dan `amount` yang harus sama dengan `grand_total` pesanan.
- Satu pesanan boleh memiliki beberapa `payments` berstatus `REJECTED`, tetapi hanya satu `SUBMITTED` atau `CONFIRMED` pada waktu yang sama.
- Bukti transfer adalah klaim pembayaran, bukan bukti bahwa dana sudah diterima.
- Staf dengan `payment.review` memeriksa bukti dan dapat menolak dengan alasan.
- Staf dengan `payment.confirm` mengonfirmasi pembayaran melalui RPC `confirm_order_payment`.
- Penolakan mengubah pembayaran menjadi `REJECTED` dan order `payment_status` kembali ke `UNPAID`. Pelanggan dapat mengunggah ulang.

### 9.5 Konfirmasi pembayaran dan alokasi stok

`confirm_order_payment(order_id, payment_id, idempotency_key)` dalam satu transaksi:
1. Validasi izin `payment.confirm`, cabang, status pesanan, dan status pembayaran (`SUBMITTED`).
2. Periksa idempotensi.
3. Kunci baris `orders` dan `payments`.
4. Pastikan `payment.amount = orders.grand_total`.
5. Kunci baris `stock_balances` untuk setiap item, diurutkan `(product_id, location_id, lot_id)`.
6. Hitung `available` untuk setiap item dan alokasi FEFO.
7. Jika semua item tersedia:
   - `payments.status = CONFIRMED`
   - `orders.payment_status = CONFIRMED`
   - Buat `stock_reservations` untuk setiap item
   - Naikkan `reserved` sesuai reservasi
   - `orders.fulfillment_status = PROCESSING`
   - Buat `cash_movements` dengan `CASH_IN_ONLINE`
8. Jika sebagian atau seluruh item tidak tersedia:
   - `payments.status = CONFIRMED`
   - `orders.payment_status = CONFIRMED`
   - Tidak ada reservasi
   - `orders.fulfillment_status = STOCK_REVIEW`
   - Buat `cash_movements` dengan `CASH_IN_ONLINE`
   - Catat daftar item kekurangan di `orders.stock_review_details`
9. Catat audit dan hasil idempotensi.

Pembayaran yang sudah dikonfirmasi tidak dibatalkan karena stok kurang. Pembayaran tetap sah dan pesanan masuk `STOCK_REVIEW`.

### 9.6 Penanganan STOCK_REVIEW

Pesanan `STOCK_REVIEW` hanya dapat diselesaikan dengan salah satu tindakan berikut yang dicatat di `order_review_actions`:

| Tindakan | Syarat | Dampak |
|----------|--------|--------|
| `PARTIAL_FULFILL` | Pelanggan menyetujui secara tertulis (catatan di portal atau bukti telepon oleh staf) | Item yang tersedia direservasi. Selisih dihitung sebagai refund dan dibuat permintaan refund. |
| `SUBSTITUTE` | Pelanggan menyetujui produk pengganti dengan harga sama atau lebih rendah | Item diganti. Selisih harga ditangani sebagai refund jika lebih rendah. Jika lebih tinggi, pelanggan membayar selisih melalui pembayaran tambahan yang dikonfirmasi. |
| `CANCEL_AND_REFUND` | Selalu tersedia | Pesanan dibatalkan dan refund penuh dibuat. |

Pesanan `STOCK_REVIEW` tidak boleh masuk `PROCESSING` atau diserahkan sebelum tindakan di atas tercatat dan reservasi (jika ada) dibuat.

### 9.7 Fulfillment

Status fulfillment: `NOT_STARTED`, `STOCK_REVIEW`, `PROCESSING`, `READY_FOR_PICKUP`, `DISPATCHED`, `COMPLETED`, `CANCELLED`.

Transisi valid:

```
NOT_STARTED → PROCESSING                 (pembayaran dikonfirmasi, stok tersedia)
NOT_STARTED → STOCK_REVIEW               (pembayaran dikonfirmasi, stok kurang)
NOT_STARTED → CANCELLED                  (pembatalan sebelum pembayaran)
STOCK_REVIEW → PROCESSING                (setelah tindakan review)
STOCK_REVIEW → CANCELLED                 (CANCEL_AND_REFUND)
PROCESSING → READY_FOR_PICKUP            (pengambilan di toko)
PROCESSING → DISPATCHED                  (pengiriman)
READY_FOR_PICKUP → COMPLETED             (diambil pelanggan)
DISPATCHED → COMPLETED                   (diterima pelanggan)
PROCESSING → CANCELLED                   (dibatalkan dengan refund)
```

Setiap transisi:
- Tercatat di audit dengan pelaku dan waktu.
- Saat `→ COMPLETED`, reservasi dikonversi menjadi mutasi `ORDER_FULFILLMENT`.
- Saat `→ CANCELLED` dari `PROCESSING`, reservasi dilepas (`RESERVATION_RELEASE`) dan refund dibuat.

Pesanan yang sudah dikonfirmasi pembayarannya dan dibatalkan membuat refund dengan status `REQUESTED`. `orders.payment_status` menjadi `REFUND_PENDING`, lalu `REFUNDED` setelah refund selesai (lihat Bagian 13).

Pengiriman mencatat `order_fulfillments` dengan tipe `PICKUP` atau `DELIVERY`, nama penerima, alamat (untuk delivery), waktu dispatch, dan waktu diterima. Pengiriman tidak melacak kurir eksternal dalam versi ini.

### 9.8 Biaya pengiriman

- Biaya pengiriman tetap per cabang, dikonfigurasi di `branch_settings.delivery_fee`.
- Biaya ditentukan saat checkout dan disimpan di `orders.delivery_fee`.
- Pelanggan memilih pengambilan atau pengiriman. Pengambilan tidak dikenai biaya.

---

## 10. Klinik dan Rekam Medis

### 10.1 Jadwal dokter

- `doctor_schedules` berisi hari, jam mulai, jam selesai, durasi slot, dan cabang.
- Janji temu tidak boleh tumpang tindih untuk dokter yang sama. Constraint eksklusi basis data menegakkan ini untuk status aktif.
- Pelanggan membuat janji temu dari portal. Staf membuat janji temu untuk pelanggan walk-in.
- Pembatalan janji oleh pelanggan wajib dilakukan minimal 2 jam sebelum jadwal. Pembatalan lebih dekat dilakukan oleh staf.

Status janji temu: `SCHEDULED`, `CHECKED_IN`, `IN_CONSULTATION`, `COMPLETED`, `CANCELLED`, `NO_SHOW`.

Janji yang berstatus `NO_SHOW` dicatat pada profil pelanggan sebagai informasi.

### 10.2 Kunjungan

- Kunjungan dibuat saat janji temu dimulai (`CHECKED_IN` menjadi `IN_CONSULTATION`) atau saat kunjungan tanpa janji dicatat.
- `visits` berisi: hewan, cabang, dokter, waktu mulai, waktu selesai, status (`OPEN`, `COMPLETED`, `CANCELLED`), berat saat kunjungan, suhu, detak jantung, laju napas, dan nama pemberi persetujuan.
- Persetujuan tindakan dicatat dengan nama wali yang memberikan persetujuan dan metode (`IN_PERSON`, `PHONE`).
- Dokter yang ditugaskan adalah dokter yang membuka kunjungan atau yang ditunjuk pada janji temu.

### 10.3 Rekam medis

Status: `DRAFT`, `SIGNED`, `AMENDED`, `VOIDED`.

- Satu rekam medis per kunjungan (unik).
- `DRAFT` dapat diubah oleh dokter yang ditugaskan.
- `DRAFT` dapat di-`VOIDED` oleh dokter pembuatnya dengan alasan. Rekam medis `VOIDED` tidak dapat diubah.
- Penandatanganan (`sign_medical_record`) memerlukan `medical_record.sign`, penugasan ke kunjungan, dan profil dokter dengan STR.
- Identitas penanda tangan diambil dari sesi server, bukan dari input klien.
- Setelah `SIGNED`, kolom klinis tidak dapat diubah. Trigger basis data menolak `UPDATE` pada kolom tersebut.
- Resep dalam kunjungan yang sama ditandatangani bersamaan dengan rekam medis.

Isi rekam medis:
- Kolom terstruktur: `chief_complaint`, `history`, `physical_exam`, `diagnosis`, `treatment_plan`, `follow_up_date`, `weight_kg`, `temperature_c`, `heart_rate`, `respiratory_rate`.
- `clinical_data JSONB` untuk bagian formulir tambahan. Kolom yang difilter, dilaporkan, atau wajib diisi harus terstruktur.
- `internal_note TEXT` tidak terlihat oleh pelanggan.

### 10.4 Amendemen

- Amendemen dibuat untuk rekam medis `SIGNED` atau `AMENDED`.
- Hanya dokter penanda asli yang dapat membuat amendemen, dengan izin `medical_record.amend`.
- Amendemen menyimpan: alasan (wajib), `previous_content` (snapshot lengkap sebelum perubahan), `new_content`, pelaku, dan waktu.
- Status menjadi `AMENDED` setelah amendemen pertama.
- Snapshot sebelumnya tidak pernah dihapus atau diubah.
- Pelanggan melihat versi terakhir dan tanggal amendemen, tetapi tidak melihat `internal_note`.

### 10.5 Resep dan pengeluaran obat

- Resep berisi: produk, dosis (teks), frekuensi, durasi hari, jumlah, dan instruksi.
- Dokter memasukkan dosis secara manual. Sistem tidak menghitung dosis otomatis.
- Resep yang sudah ditandatangani tidak dapat diubah. Pembatalan dilakukan dengan `CANCELLED` dan alasan.
- Stok tidak berkurang saat resep dibuat.
- Pengeluaran obat (`medication_dispensings`) dilakukan oleh dokter atau petugas dengan `clinic.medication.use`, dari lokasi `CLINIC_PHARMACY`.
- Pengeluaran dapat parsial terhadap resep dan dicatat per kejadian.
- Setiap pengeluaran mengurangi stok dengan FEFO dan membuat mutasi `CLINIC_USE`. Harga dan biaya snapshot disimpan.
- Biaya pengeluaran dimasukkan ke invoice layanan kunjungan atau rawat inap terkait.

### 10.6 Tindakan dan pemakaian bahan habis pakai

- Setiap tindakan berasal dari `service_catalog` kategori `CLINIC`.
- `treatment_records` mencatat tindakan, dokter, waktu, catatan, dan biaya layanan.
- Bahan habis pakai (kategori `CLINIC_SUPPLY`) dicatat melalui `medication_dispensings` dengan `reason = PROCEDURE`.

### 10.7 Laporan layanan dokter

Laporan per dokter menampilkan: jumlah kunjungan selesai, jumlah tindakan, jumlah rawat inap yang ditangani, total nilai layanan dari invoice terkait, dan jumlah rekam medis berstatus `DRAFT` yang belum ditandatangani. Laporan ini tidak menghitung gaji atau komisi (K3).

---

## 11. Rawat Inap

### 11.1 Unit rawat

- `care_units` berisi: cabang, kode, nama, tipe (`KENNEL`, `ICU`, `ISOLATION`, `BOARDING_ROOM`), kapasitas (default 1), dan status aktif.
- Unit `BOARDING_ROOM` digunakan untuk penitipan. Unit `KENNEL`, `ICU`, dan `ISOLATION` digunakan untuk rawat inap klinis.

### 11.2 Penempatan

- Penempatan disimpan di `hospitalization_assignments` dengan `unit_id`, `start_at`, `end_at`, dan `hospitalization_id`.
- Penempatan tidak boleh melampaui kapasitas unit pada rentang waktu yang sama.
- Pemeriksaan kapasitas dilakukan dalam transaksi dengan penguncian baris unit.
- Pindah unit menutup penempatan lama dan membuat penempatan baru dalam transaksi yang sama.

### 11.3 Episode rawat inap

Status: `PLANNED`, `ADMITTED`, `DISCHARGE_PENDING`, `DISCHARGED`, `CANCELLED`.

- `PLANNED` dibuat saat penempatan dijadwalkan.
- `ADMITTED` berarti hewan sudah masuk unit. Dalam status ini, pemantauan aktif.
- `DISCHARGE_PENDING` dibuat saat dokter memulai proses pulang. Invoice layanan final dibuat.
- `DISCHARGED` dicatat setelah dokter menyelesaikan proses pulang. Semua penempatan ditutup dan unit dilepas.
- Pulang tidak memerlukan pelunasan. Sisa tagihan menjadi piutang (Bagian 13).
- Episode `CANCELLED` hanya dapat dibuat untuk `PLANNED`.

### 11.4 Observasi dan tugas

- `care_observations` mencatat tipe (`VITALS`, `MEDICATION_GIVEN`, `FEEDING`, `URINATION`, `DEFECATION`, `PROCEDURE`, `NOTE`), waktu observasi, pencatat, nilai, dan catatan.
- Pemberian obat dari observasi `MEDICATION_GIVEN` harus terhubung dengan pengeluaran obat (`medication_dispensings`) agar stok tercatat.
- `care_tasks` berisi jenis tugas, waktu jatuh tempo, penerima tugas, status (`OPEN`, `DONE`, `CANCELLED`), waktu selesai, dan pelaksana.
- Observasi dan tugas bersifat internal dan tidak terlihat oleh pelanggan.

### 11.5 Update pelanggan

- `shared_updates` adalah update yang dapat dilihat pelanggan. Jenis: `DAILY_SUMMARY`, `PROGRESS`, `DISCHARGE_SUMMARY`.
- Dokter yang berwenang (`hospitalization.publish_update`) membuat update dan mempublikasikannya. Update dapat dibuat dari ringkasan observasi atau ditulis bebas.
- Update yang sudah dipublikasikan tidak dapat diubah. Koreksi dibuat sebagai update baru.
- Setiap update mencatat pembuat, waktu publikasi, dan hewan terkait.
- Pelanggan wali dari hewan tersebut dapat melihat update yang dipublikasikan.
- Update tidak menyalin observasi internal secara otomatis. Dokter memilih isi yang ditampilkan.

---

## 12. Grooming dan Penitipan

### 12.1 Katalog layanan

- `service_catalog` berisi: kode, nama, kategori (`GROOMING`, `BOARDING`, `CLINIC`), durasi menit, harga per cabang (`product_prices` berlaku untuk layanan dengan tipe `SERVICE`), dan status aktif.
- Layanan tambahan didefinisikan dengan `is_addon = true` dan harus disetujui wali utama sebelum ditagih.

### 12.2 Kapasitas

- `service_capacity` mendefinisikan kapasitas per cabang, layanan, dan slot waktu.
- Slot grooming diambil dari durasi layanan dan jam operasional cabang.
- Booking yang melebihi kapasitas ditolak dengan `CONFLICT` atau `INSUFFICIENT_CAPACITY`.
- Penitipan memakai `care_units` bertipe `BOARDING_ROOM` dengan pemeriksaan kapasitas yang sama seperti rawat inap.

### 12.3 Booking

Status: `REQUESTED`, `CONFIRMED`, `CHECKED_IN`, `IN_PROGRESS`, `COMPLETED`, `CANCELLED`, `NO_SHOW`.

- Pelanggan dapat membuat `REQUESTED`. Staf mengonfirmasi menjadi `CONFIRMED`.
- Staf dapat membuat booking langsung dalam status `CONFIRMED`.
- Pembatalan pelanggan minimal 4 jam sebelum slot. Pembatalan lebih dekat dilakukan oleh staf.
- Perubahan jadwal membuat booking baru dan membatalkan yang lama dalam transaksi, dengan pencatatan audit.

### 12.4 Penitipan

- Penitipan dibuat sebagai booking bertipe `BOARDING` dengan `boarding_stays` (tanggal masuk, tanggal keluar, unit).
- Check-in mencatat kondisi awal hewan (berat, catatan kondisi, dokumentasi opsional) dan membuat penempatan.
- Check-out mencatat kondisi akhir, menutup penempatan, dan membuat invoice layanan final.

### 12.5 Layanan tambahan dan persetujuan

- Setiap layanan tambahan memiliki status persetujuan: `PENDING`, `APPROVED`, `DECLINED`.
- Persetujuan dapat diberikan oleh wali utama melalui portal, atau dicatat oleh staf dengan metode `PHONE` dan nama pemberi persetujuan.
- Layanan tambahan yang berstatus `PENDING` tidak masuk invoice.
- Layanan tambahan `DECLINED` tidak masuk invoice dan dicatat sebagai keputusan pelanggan.

### 12.6 Check-in dan check-out grooming

- Check-in mencatat kondisi awal dan waktu mulai.
- Check-out mencatat kondisi akhir, catatan petugas, layanan final, dan membuat invoice layanan.

---

## 13. Tagihan Layanan, Piutang, dan Refund

### 13.1 Invoice layanan

- `service_invoices` dibuat untuk kunjungan klinik, episode rawat inap, booking grooming, dan penitipan.
- Status: `OPEN`, `PARTIALLY_PAID`, `PAID`, `VOIDED`.
- `service_invoice_items` berisi: jenis (`CONSULTATION`, `TREATMENT`, `MEDICATION`, `SUPPLY`, `BOARDING_DAY`, `GROOMING`, `ADDON`, `OTHER`), referensi sumber, deskripsi, jumlah, harga snapshot, diskon, dan total.
- Total dihitung server dengan pembulatan half-up ke rupiah penuh.
- Diskon invoice hanya dapat diberikan oleh pemilik (`service_invoice.discount`).
- Invoice dapat ditambah item selama `OPEN` atau `PARTIALLY_PAID`. Penambahan ke invoice yang sudah dibayar sebagian dicatat sebagai item baru.
- Invoice `VOIDED` hanya dapat dibuat jika belum ada pembayaran, dengan alasan dan izin `service_invoice.void`.
- Setelah pembayaran pertama, koreksi dilakukan dengan item penyesuaian, bukan penghapusan.

### 13.2 Pembayaran layanan

- `service_payments` mencatat jumlah, metode, waktu, dan pencatat.
- Pembayaran parsial diizinkan. Jumlah pembayaran tidak boleh melebihi sisa tagihan.
- Setiap pembayaran membuat `cash_movements` dengan `CASH_IN_SERVICE` untuk cabang invoice.
- Status invoice diperbarui dalam transaksi yang sama.
- Pembayaran tidak dapat dihapus. Koreksi dilakukan dengan `service_payments` berjumlah negatif dengan alasan dan izin `finance.cash.read` pada level pemilik.

### 13.3 Piutang

- Piutang adalah total sisa tagihan (`balance_due`) dari invoice berstatus `OPEN` atau `PARTIALLY_PAID` untuk pelanggan.
- Laporan piutang menampilkan umur piutang dalam 0–30, 31–60, 61–90, dan di atas 90 hari.
- Piutang ditampilkan di portal pelanggan untuk wali utama.

### 13.4 Refund

Refund berasal dari:

- Pembatalan pesanan online yang sudah dibayar.
- Retur POS.
- Tindakan `PARTIAL_FULFILL` atau `SUBSTITUTE` pada pesanan `STOCK_REVIEW`.
- Kelebihan pembayaran pada invoice layanan (lihat 13.5).

Alur refund:
1. Pemohon (`refund.request`) membuat refund dengan jumlah, alasan, dan referensi asal. Status `REQUESTED`.
2. Pemilik atau pengguna dengan `refund.approve`, selain pemohon, menyetujui. Status `APPROVED`.
3. Pengguna dengan `refund.complete` mencatat pengembalian dana secara manual, termasuk metode, referensi transfer, dan bukti (opsional). Status `COMPLETED`.
4. Penolakan mengubah status menjadi `REJECTED` dengan alasan.

Jumlah refund tidak boleh melebihi jumlah yang dibayar dikurangi refund yang sudah ada.

Refund `COMPLETED` membuat `cash_movements` dengan `CASH_OUT_REFUND`.

### 13.5 Kelebihan pembayaran layanan

- Kelebihan pembayaran pada invoice layanan tidak dianggap tip atau saldo pelanggan otomatis.
- Kelebihan dicatat sebagai `customer_credit` yang dapat dipakai untuk invoice berikutnya dari pelanggan yang sama, atau dikembalikan melalui refund.
- Penggunaan kredit dicatat sebagai pembayaran dengan metode `CREDIT`.

---

## 14. Keuangan, Kas, dan Laporan

### 14.1 Buku kas

- `cash_movements` mencatat seluruh uang masuk dan keluar per cabang, metode, dan referensi.
- Tipe: `CASH_IN_SALE`, `CASH_IN_ONLINE`, `CASH_IN_SERVICE`, `CASH_OUT_REFUND`, `CASH_OUT_EXPENSE`, `CASH_OUT_SUPPLIER`, `SHIFT_OPENING`, `SHIFT_CLOSING_DIFFERENCE`.
- `cash_movements` bersifat append-only.
- Saldo kas tunai per shift dihitung dari metode `CASH`.
- Saldo per rekening transfer dihitung dari metode transfer dan rekening yang digunakan.

### 14.2 Pengeluaran operasional

- Pengeluaran dicatat dengan kategori, jumlah, tanggal, cabang, pemohon, dan bukti (opsional).
- Pengeluaran di atas Rp 1.000.000 wajib disetujui pengguna lain dengan `finance.expense.approve`.
- Pengeluaran yang sudah disetujui atau dicatat tidak dapat diubah. Koreksi dilakukan dengan pengeluaran penyesuaian.
- Pengeluaran membuat `cash_movements` dengan `CASH_OUT_EXPENSE` saat dibayar.

### 14.3 Periode keuangan

- Periode bulanan ditutup oleh pemilik (`finance.period.close`).
- Setelah periode ditutup, tidak ada transaksi baru yang dapat dicatat dengan tanggal dalam periode tersebut.
- Koreksi periode tertutup dilakukan pada periode aktif berikutnya dengan catatan referensi.
- Penutupan menghasilkan ringkasan: penjualan, penerimaan kas per metode, pengeluaran, piutang, nilai persediaan, HPP, dan laba kotor.

### 14.4 Definisi laporan

| Laporan | Definisi | Sumber |
|---------|----------|--------|
| Penjualan | Total `grand_total` penjualan POS berstatus `COMPLETED`, `PARTIALLY_RETURNED`, atau `RETURNED` dalam rentang tanggal, setelah dikurangi retur | `sales`, `sales_returns` |
| Penjualan online | Total `grand_total` pesanan dengan `payment_status = CONFIRMED` atau setelahnya, setelah dikurangi refund | `orders`, `refunds` |
| Penerimaan kas | Seluruh `cash_movements` masuk per metode | `cash_movements` |
| Pengeluaran | Pengeluaran yang disetujui atau dicatat dan belum dibatalkan | `operating_expenses` |
| Piutang | Sisa tagihan invoice layanan yang belum lunas | `service_invoices` |
| Nilai persediaan | Σ `on_hand × avg_unit_cost` per cabang | `stock_balances`, `product_branch_costs` |
| HPP | Σ `quantity × unit_cost_snapshot` dari penjualan, pemakaian klinik, dan pesanan yang dikonfirmasi | `sale_items`, `medication_dispensings`, `order_items` |
| Laba kotor | Penjualan dikurangi HPP | Turunan |
| Laba rugi sederhana | Laba kotor dikurangi pengeluaran operasional dalam periode | Turunan |
| Arus kas sederhana | Penerimaan kas dikurangi pengeluaran kas dalam periode | `cash_movements` |

Catatan:
- Pembelian stok bukan beban pada hari pembelian. Beban HPP diakui saat barang terjual atau dipakai.
- Laba rugi berbeda dari arus kas. Piutang dan pembayaran supplier tidak masuk laba rugi secara langsung.
- Semua laporan dapat ditelusuri ke ID transaksi sumber melalui tautan.

### 14.5 Ekspor

- Setiap laporan dapat diekspor ke CSV dan PDF.
- Ekspor PDF memakai template invoice dan laporan yang disimpan di `settings`.
- Ekspor berisi tanggal cetak, nama pengguna, dan cabang.

---

## 15. Portal Pelanggan

Pelanggan dengan akun login dapat:
- Melihat hewan yang terhubung dengan akunnya, beserta peran wali (utama atau sekunder).
- Membuat janji temu klinik dan booking grooming atau penitipan.
- Melihat riwayat kunjungan dan rekam medis yang sudah ditandatangani, dengan isi terstruktur tanpa `internal_note`. Pelanggan tidak melihat observasi internal.
- Melihat update rawat inap yang sudah dipublikasikan.
- Melihat pesanan online sendiri, membatalkan pesanan yang belum dibayar, dan mengunggah bukti pembayaran.
- Melihat invoice dan pembayaran untuk hewan yang terhubung. Invoice layanan ditampilkan kepada wali utama. Wali sekunder melihat riwayat tanpa angka tagihan.
- Menyetujui layanan tambahan sebagai wali utama.
- Melihat riwayat pembelian POS yang tercatat atas nomor telepon atau email yang sama dengan akunnya.

Pelanggan tidak dapat melihat stok internal, harga pokok, data pelanggan lain, atau data pengguna staf.

---

## 16. Notifikasi

| Peristiwa | Penerima | Saluran |
|-----------|----------|---------|
| Pembayaran pesanan diterima untuk diperiksa | Staf dengan `payment.review` di cabang | Dalam aplikasi |
| Pembayaran dikonfirmasi | Pelanggan | Dalam aplikasi, email |
| Pembayaran ditolak | Pelanggan | Dalam aplikasi, email |
| Pesanan `STOCK_REVIEW` | Staf cabang dan pelanggan | Dalam aplikasi, email |
| Status fulfillment berubah | Pelanggan | Dalam aplikasi, email |
| Janji temu dibuat, diubah, atau dibatalkan | Pelanggan dan dokter | Dalam aplikasi, email |
| Booking grooming atau penitipan dikonfirmasi | Pelanggan | Dalam aplikasi, email |
| Update rawat inap dipublikasikan | Wali hewan | Dalam aplikasi, email |
| Tagihan baru atau jatuh tempo | Wali utama | Dalam aplikasi, email |
| Refund disetujui atau selesai | Pemohon dan pelanggan terkait | Dalam aplikasi |
| Stok minimum tercapai | Pemilik dan admin cabang | Dalam aplikasi |
| Batch kedaluwarsa dalam 90 hari | Pemilik | Dalam aplikasi |
| Job terjadwal gagal | Pemilik dan developer on-call | Email |

Aturan isi:
- Email dan notifikasi tidak memuat diagnosis, nama obat, hasil pemeriksaan, atau isi update klinis.
- Email hanya memuat judul umum, nomor referensi, dan tautan ke portal.
- Notifikasi dalam aplikasi untuk staf dapat memuat nama pelanggan dan nama hewan karena pengguna sudah terautentikasi.
- Notifikasi disimpan di `notifications` dan dapat ditandai dibaca.

---

## 17. Basis Data

### 17.1 Konvensi

- Setiap tabel bisnis memiliki `id UUID` (default `gen_random_uuid()`), `created_at TIMESTAMPTZ`, `created_by UUID`.
- Tabel yang dapat diubah memiliki `updated_at TIMESTAMPTZ` dan `updated_by UUID`, diperbarui trigger.
- Tabel `branch_id` wajib untuk setiap entitas yang terikat cabang.
- Nama kolom `snake_case` dalam bahasa Inggris. Nilai enum dalam huruf besar.
- Uang: `NUMERIC(14,2)`. Biaya satuan: `NUMERIC(14,4)`. Kuantitas: `COUNT` memakai `NUMERIC(14,0)`; `WEIGHT` memakai `NUMERIC(14,3)`.
- Semua nilai uang dan kuantitas memiliki `CHECK` nonnegatif kecuali yang secara eksplisit bertanda negatif (`service_payments` koreksi).
- Foreign key wajib untuk relasi bisnis.
- `ON DELETE RESTRICT` untuk semua foreign key. `CASCADE` tidak digunakan.
- Enum status menggunakan `CHECK (status IN (...))` agar mudah diubah melalui migration.
- Tabel append-only (`inventory_movements`, `cash_movements`, `audit_logs`, `medical_record_amendments`, `shared_updates`, `payments`, `sale_payments`, `product_prices`, `cost_history`, `stock_reservations` setelah dibuat): `UPDATE` dan `DELETE` diblokir dengan trigger dan tidak diberikan grant.
- Penghapusan fisik data bisnis tidak tersedia melalui aplikasi.

### 17.2 Daftar tabel

**Identitas dan akses**
`branches`, `branch_settings`, `profiles`, `doctor_profiles`, `roles`, `permissions`, `role_permissions`, `user_roles`, `user_branch_access`, `user_permission_grants`, `invitations`, `audit_logs`, `idempotency_keys`, `rate_limit_counters`, `notifications`, `job_runs`

**Pelanggan dan hewan**
`customers`, `customer_addresses`, `pets`, `pet_guardians`, `pet_documents`

**Produk dan supplier**
`product_categories`, `products`, `product_barcodes`, `product_branch_settings`, `product_prices`, `suppliers`, `branch_payment_accounts`, `payment_methods`

**Inventori**
`inventory_locations`, `inventory_lots`, `stock_balances`, `inventory_movements`, `stock_reservations`, `inventory_adjustments`, `count_sessions`, `count_session_items`, `inventory_transfers`, `inventory_transfer_items`, `product_branch_costs`, `cost_history`

**Pembelian**
`purchase_orders`, `purchase_order_items`, `goods_receipts`, `goods_receipt_items`, `supplier_returns`, `supplier_return_items`, `supplier_bills`, `supplier_payments`

**POS**
`cash_shifts`, `sales`, `sale_items`, `sale_payments`, `sales_returns`, `sales_return_items`, `refunds`

**Toko online**
`customer_carts`, `customer_cart_items`, `orders`, `order_items`, `payments`, `order_review_actions`, `order_fulfillments`, `stock_reservations` (bersama)

**Klinik**
`service_catalog`, `doctor_schedules`, `appointments`, `visits`, `medical_records`, `medical_record_amendments`, `prescriptions`, `prescription_items`, `treatment_records`, `medication_dispensings`

**Rawat inap**
`care_units`, `hospitalizations`, `hospitalization_assignments`, `care_observations`, `care_tasks`, `shared_updates`

**Grooming dan penitipan**
`service_bookings`, `service_booking_items`, `boarding_stays`, `service_capacity`

**Tagihan dan keuangan**
`service_invoices`, `service_invoice_items`, `service_payments`, `customer_credits`, `cash_movements`, `operating_expenses`, `financial_periods`, `invoice_sequences`

### 17.3 Kolom kritis

| Tabel | Kolom wajib (selain standar) | Constraint utama |
|-------|------------------------------|------------------|
| `branches` | `code`, `name`, `timezone`, `is_active` | `code` unik; `timezone` dari daftar yang diizinkan |
| `profiles` | `id` (sama dengan `auth.users.id`), `display_name`, `phone`, `is_active` | `id` PK dan FK |
| `doctor_profiles` | `user_id`, `license_number` (STR) | `license_number` unik; satu profil per user |
| `customers` | `auth_user_id` (nullable), `name`, `phone`, `email`, `is_active` | `auth_user_id` unik jika tidak NULL |
| `pets` | `customer_id` (tidak lagi pemilik tunggal; lihat `pet_guardians`), `name`, `species`, `breed`, `sex`, `is_neutered`, `birth_date`, `birth_date_is_estimate`, `microchip`, `is_deceased`, `deceased_at` | Enum spesies dan jenis kelamin |
| `pet_guardians` | `pet_id`, `customer_id`, `is_primary` | Indeks unik parsial: satu `is_primary = true` per `pet_id` aktif |
| `products` | `sku`, `name`, `category_id`, `unit_type`, `base_unit`, `track_lot`, `is_prescription`, `is_active` | `sku` unik |
| `product_prices` | `product_id`, `branch_id`, `price`, `effective_from`, `created_by` | Append-only; `price >= 0` |
| `product_branch_settings` | `product_id`, `branch_id`, `is_pos_sellable`, `is_online_sellable`, `min_stock`, `is_active` | Unik `(product_id, branch_id)` |
| `inventory_locations` | `branch_id`, `code`, `type` (`RETAIL`, `CLINIC_PHARMACY`, `WAREHOUSE`) | Unik `(branch_id, code)`; indeks unik parsial untuk satu `RETAIL` dan satu `CLINIC_PHARMACY` per cabang |
| `inventory_lots` | `product_id`, `location_id`, `lot_number`, `expiry_date` (nullable untuk `DEFAULT`), `received_at` | Unik `(product_id, location_id, lot_number)` |
| `stock_balances` | `product_id`, `location_id`, `lot_id`, `on_hand`, `reserved` | Unik `(product_id, location_id, lot_id)`; `CHECK (on_hand >= 0 AND reserved >= 0 AND reserved <= on_hand)` |
| `inventory_movements` | `product_id`, `location_id`, `lot_id`, `quantity_delta`, `movement_type`, `reference_type`, `reference_id`, `unit_cost_snapshot`, `reason_code`, `actor_id` | Append-only; `quantity_delta <> 0` |
| `stock_reservations` | `order_item_id` (unik), `product_id`, `location_id`, `lot_id`, `quantity`, `status` (`ACTIVE`, `RELEASED`, `FULFILLED`) | Unik `order_item_id`; `quantity > 0` |
| `product_branch_costs` | `product_id`, `branch_id`, `avg_unit_cost NUMERIC(14,4)`, `updated_at` | Unik `(product_id, branch_id)`; `avg_unit_cost >= 0` |
| `sales` | `branch_id`, `shift_id`, `customer_id` (nullable), `invoice_number`, `status`, `subtotal`, `discount_total`, `grand_total`, `completed_at`, `completed_by` | Unik `invoice_number`; `CHECK (grand_total >= 0)` |
| `sale_items` | `sale_id`, `product_id`, `quantity`, `unit_price`, `discount_type`, `discount_value`, `line_total`, `unit_cost_snapshot` | `quantity > 0` |
| `sale_payments` | `sale_id`, `method_id`, `amount`, `tendered_amount`, `change_amount` | Append-only |
| `cash_shifts` | `branch_id`, `cashier_id`, `status` (`OPEN`, `CLOSED`), `opening_cash`, `closing_cash_counted`, `difference`, `difference_reason` | Indeks unik parsial: satu shift `OPEN` per kasir |
| `orders` | `customer_id`, `branch_id`, `payment_status`, `fulfillment_status`, `subtotal`, `delivery_fee`, `grand_total`, `payment_instructions_snapshot`, `stock_review_details` | `grand_total = subtotal + delivery_fee` (CHECK) |
| `payments` | `order_id`, `method_id`, `amount`, `status`, `proof_path`, `submitted_at`, `reviewed_by`, `confirmed_at`, `confirmed_by`, `rejection_reason` | Indeks unik parsial: satu `SUBMITTED` atau `CONFIRMED` per pesanan |
| `refunds` | `source_type`, `source_id`, `amount`, `reason`, `status`, `requested_by`, `approved_by`, `completed_by`, `payout_method`, `payout_reference` | `approved_by <> requested_by` (CHECK); `amount > 0` |
| `visits` | `pet_id`, `branch_id`, `doctor_id`, `appointment_id` (nullable), `status`, `started_at`, `completed_at`, `consent_given_by_name`, `consent_method`, `weight_kg`, `temperature_c`, `heart_rate`, `respiratory_rate` | Satu kunjungan per janji temu jika ada |
| `medical_records` | `visit_id` (unik), `status`, `chief_complaint`, `history`, `physical_exam`, `diagnosis`, `treatment_plan`, `follow_up_date`, `clinical_data`, `internal_note`, `signed_by`, `signed_at`, `doctor_license_snapshot` | Trigger blokir perubahan kolom klinis saat `SIGNED` |
| `medical_record_amendments` | `medical_record_id`, `reason`, `previous_content`, `new_content`, `amended_by`, `amended_at` | Append-only |
| `prescriptions` | `visit_id`, `doctor_id`, `status` (`DRAFT`, `SIGNED`, `CANCELLED`), `signed_at` | — |
| `prescription_items` | `prescription_id`, `product_id`, `dose_text`, `frequency_text`, `duration_days`, `quantity`, `instructions` | `quantity > 0` |
| `medication_dispensings` | `visit_id` atau `hospitalization_id`, `prescription_item_id` (nullable), `product_id`, `location_id`, `quantity`, `unit_cost_snapshot`, `unit_price_snapshot`, `reason` (`PRESCRIPTION`, `PROCEDURE`, `WARD`), `dispensed_by` | Append-only |
| `care_units` | `branch_id`, `code`, `type`, `capacity`, `is_active` | Unik `(branch_id, code)`; `capacity >= 1` |
| `hospitalization_assignments` | `hospitalization_id`, `unit_id`, `start_at`, `end_at` | `end_at > start_at`; kapasitas diperiksa dalam RPC |
| `care_observations` | `hospitalization_id`, `observed_at`, `observation_type`, `values` (JSONB), `note`, `recorded_by` | Append-only |
| `shared_updates` | `hospitalization_id`, `update_type`, `body`, `published_at`, `published_by` | Append-only; `published_at` wajib |
| `service_bookings` | `branch_id`, `pet_id`, `customer_id`, `service_id`, `type` (`GROOMING`, `BOARDING`), `status`, `starts_at`, `ends_at` | Constraint kapasitas di RPC; `ends_at > starts_at` |
| `service_booking_items` | `booking_id`, `service_id`, `is_addon`, `approval_status`, `approved_by`, `approval_method`, `approved_at` | — |
| `service_invoices` | `branch_id`, `customer_id`, `source_type`, `source_id`, `invoice_number`, `status`, `subtotal`, `discount_total`, `grand_total`, `amount_paid`, `balance_due` | `balance_due = grand_total − amount_paid` (CHECK); `balance_due >= 0` |
| `service_invoice_items` | `invoice_id`, `item_type`, `source_type`, `source_id`, `description`, `quantity`, `unit_price`, `discount_amount`, `line_total`, `unit_cost_snapshot` | `line_total >= 0` |
| `service_payments` | `invoice_id`, `method_id`, `amount`, `paid_at`, `recorded_by`, `reference` | Append-only; `amount <> 0` |
| `customer_credits` | `customer_id`, `source_type`, `source_id`, `amount`, `used_amount` | `used_amount <= amount` |
| `cash_movements` | `branch_id`, `shift_id` (nullable), `movement_type`, `method_id`, `amount`, `reference_type`, `reference_id`, `occurred_at` | Append-only; `amount <> 0` |
| `operating_expenses` | `branch_id`, `category`, `amount`, `expense_date`, `requested_by`, `approved_by`, `status`, `period_id` | `approved_by <> requested_by` untuk jumlah di atas batas |
| `financial_periods` | `branch_id`, `period_start`, `period_end`, `status`, `closed_by`, `closed_at` | Tidak ada overlap periode per cabang |
| `invoice_sequences` | `branch_id`, `doc_type`, `year`, `last_number` | Unik `(branch_id, doc_type, year)`; diperbarui dengan `FOR UPDATE` |
| `audit_logs` | `actor_id`, `action`, `entity_type`, `entity_id`, `branch_id`, `occurred_at`, `request_id`, `reason`, `metadata` | Append-only |
| `idempotency_keys` | `user_id`, `scope`, `key`, `request_hash`, `status` (`IN_PROGRESS`, `DONE`), `response`, `expires_at` | Unik `(user_id, scope, key)`; kedaluwarsa 24 jam |
| `job_runs` | `job_name`, `started_at`, `finished_at`, `status`, `detail` | Append-only |

### 17.4 Constraint eksklusi

- `appointments`: `EXCLUDE USING gist (doctor_id WITH =, tstzrange(starts_at, ends_at) WITH &&) WHERE (status IN ('SCHEDULED','CHECKED_IN','IN_CONSULTATION') AND doctor_id IS NOT NULL)`.
- `service_bookings` untuk penitipan memakai pemeriksaan kapasitas dalam RPC, bukan constraint eksklusi, karena kapasitas dapat lebih dari satu.

### 17.5 Indeks

Indeks dibuat hanya untuk pola kueri yang dipakai, yaitu:
- Pencarian produk: `products(sku)`, `product_barcodes(barcode)`, `products` trigram untuk nama.
- Saldo: `stock_balances(location_id, product_id)`.
- Ledger: `inventory_movements(product_id, location_id, occurred_at)`, `cash_movements(branch_id, occurred_at)`.
- Pesanan: `orders(branch_id, fulfillment_status)`, `orders(customer_id, created_at)`.
- Penjualan: `sales(branch_id, completed_at)`.
- Klinik: `visits(pet_id, started_at)`, `appointments(branch_id, starts_at)`.
- Audit: `audit_logs(entity_type, entity_id)`, `audit_logs(branch_id, occurred_at)`.

Indeks lain ditambahkan hanya setelah `EXPLAIN ANALYZE` pada data uji menunjukkan kebutuhan.

---

## 18. Operasi Atomik dan Fungsi Database

### 18.1 Aturan umum fungsi

- Semua fungsi mutasi memakai `SECURITY DEFINER` dengan `SET search_path = public, pg_temp`.
- Setiap fungsi memanggil `auth.uid()` dan memeriksa: pengguna aktif, izin yang diperlukan, cabang yang diizinkan, status objek, dan idempotensi bila berlaku.
- `EXECUTE` dicabut dari `PUBLIC` dan `anon`, dan diberikan hanya kepada `authenticated`.
- Fungsi tidak menerima `role`, `branch_id`, `customer_id`, `doctor_id`, atau `actor_id` dari input klien sebagai sumber otorisasi. Nilai tersebut diturunkan dari sesi dan basis data.
- Fungsi mengembalikan hasil dengan `request_id`.
- Batas waktu transaksi: 5 detik untuk POS dan pembayaran; 10 detik untuk operasi laporan dan job.
- Deadlock atau `serialization_failure` ditangani dengan retry maksimal 3 kali dengan jeda acak 50–150 ms di sisi server. Setelah itu, `CONFLICT` dikembalikan.

### 18.2 Daftar fungsi

| Fungsi | Izin | Ringkasan |
|--------|------|-----------|
| `complete_sale(sale_id, payments, idempotency_key)` | `pos.sale` | Lihat 8.2 |
| `create_sales_return(sale_id, items, idempotency_key)` | `pos.return.request` | Retur dan permintaan refund |
| `open_cash_shift(branch_id, opening_cash)` | `cash_shift.manage` | Satu shift terbuka per kasir |
| `close_cash_shift(shift_id, counted_cash, reason)` | `cash_shift.manage` | Penutupan dengan selisih |
| `confirm_order_payment(order_id, payment_id, idempotency_key)` | `payment.confirm` | Lihat 9.5 |
| `reject_order_payment(payment_id, reason)` | `payment.review` | Penolakan |
| `resolve_stock_review(order_id, action, details, idempotency_key)` | `payment.confirm` | Lihat 9.6 |
| `advance_fulfillment(order_id, to_status, details)` | `payment.confirm` atau `pos.sale` | Lihat 9.7 |
| `apply_inventory_adjustment(...)` | `inventory.adjust` | Lihat 6.4 |
| `create_transfer`, `dispatch_transfer`, `receive_transfer` | `inventory.transfer` | Lihat 6.6 |
| `receive_goods(po_id, lines, idempotency_key)` | `purchase.manage` | Lihat 7.2 |
| `record_supplier_return(...)` | `purchase.manage` | Lihat 7.3 |
| `sign_medical_record(record_id, idempotency_key)` | `medical_record.sign` | Lihat 10.3 |
| `amend_medical_record(record_id, reason, new_content, idempotency_key)` | `medical_record.amend` | Lihat 10.4 |
| `dispense_medication(...)` | `clinic.medication.use` | Lihat 10.5 |
| `admit_hospitalization(...)`, `transfer_unit(...)`, `discharge_hospitalization(...)` | `hospitalization.manage` | Lihat 11 |
| `publish_shared_update(hospitalization_id, body, type, idempotency_key)` | `hospitalization.publish_update` | Lihat 11.5 |
| `reserve_booking_slot(...)` | `grooming.manage` atau pelanggan miliknya | Pemeriksaan kapasitas |
| `create_service_invoice_from_source(source_type, source_id)` | `service_invoice.manage` | Pembuatan invoice layanan |
| `record_service_payment(invoice_id, method_id, amount, reference, idempotency_key)` | `service_invoice.manage` | Lihat 13.2 |
| `approve_refund`, `complete_refund`, `reject_refund` | `refund.approve` / `refund.complete` | Lihat 13.4 |
| `approve_expense`, `close_period` | `finance.expense.approve` / `finance.period.close` | Lihat 14 |
| `generate_invoice_number(branch_id, doc_type)` | Internal | Sequence per cabang, jenis, dan tahun |

### 18.3 Urutan penguncian

Setiap fungsi yang mengubah stok mengunci baris dalam urutan:
1. `orders` atau `sales` atau `visits` (dokumen induk)
2. `payments` atau `sale_payments` terkait
3. `stock_balances` sesuai `(product_id, location_id, lot_id)` menaik
4. `product_branch_costs` sesuai `(product_id, branch_id)` menaik
5. `invoice_sequences` untuk nomor dokumen

Urutan ini wajib dipatuhi untuk mencegah deadlock.

### 18.4 Idempotensi

- Operasi yang mengubah data bisnis wajib menerima header `Idempotency-Key` (UUID v4).
- Kunci unik: `(user_id, scope, key)`. `scope` adalah nama operasi.
- `request_hash` adalah SHA-256 dari payload kanonik.
- Aturan:
  - Kunci belum ada: buat baris `IN_PROGRESS`, jalankan operasi, simpan `DONE` dengan `response`.
  - Kunci ada, `DONE`, hash sama: kembalikan `response` yang tersimpan.
  - Kunci ada, `DONE`, hash berbeda: `IDEMPOTENCY_CONFLICT`.
  - Kunci ada, `IN_PROGRESS`: `CONFLICT` dengan status 409 dan `Retry-After: 2`.
- Kunci kedaluwarsa setelah 24 jam dibersihkan oleh job harian.

---

## 19. Keamanan

### 19.1 RLS

- RLS diaktifkan untuk setiap tabel di schema `public`.
- Tidak ada tabel bisnis yang dapat ditulis langsung oleh peran `authenticated`. Penulisan melalui RPC atau Route Handler.
- Pengecualian penulisan langsung: `profiles` (kolom tampilan pribadi), `customers` (kolom kontak milik sendiri), `customer_cart_items`, `customer_addresses`, dan `notifications` (tandai dibaca).
- Helper `has_permission(code text, branch uuid)` dan `can_access_branch(branch uuid)` adalah `SECURITY DEFINER`, hanya mengembalikan boolean, dan tidak dapat dipakai untuk membaca data.

### 19.2 Kebijakan baca per jenis data

| Data | Kebijakan |
|------|-----------|
| Profil staf | Pengguna sendiri; pemilik dan `users.manage` |
| Produk aktif (publik) | `products` dan `product_branch_settings` dengan `is_online_sellable` atau `is_pos_sellable` dapat dibaca pelanggan melalui view publik tanpa kolom internal |
| Stok | `inventory.read` pada cabang tersebut; pelanggan tidak punya akses |
| Harga pokok (`product_branch_costs`, `cost_history`, `unit_cost_snapshot`) | `inventory.read_cost` atau `finance.report.read` |
| Pesanan | Pemilik pesanan atau staf dengan `payment.review` / `payment.confirm` / `pos.sale` pada cabang fulfillment |
| Bukti pembayaran | Pemilik pesanan atau staf `payment.review` pada cabang |
| Pelanggan dan hewan | Wali pelanggan atau staf `customer.manage` / `pet.manage` pada cabang |
| Janji temu | Pelanggan wali, dokter yang ditugaskan, staf `appointment.manage` |
| Kunjungan dan rekam medis | Dokter sesuai K15, pemilik `medical_record.read`, pelanggan wali untuk rekam medis `SIGNED` tanpa `internal_note` |
| Observasi rawat inap | Dokter dan staf klinis di cabang |
| `shared_updates` | Pelanggan wali jika `published_at IS NOT NULL`; staf klinis di cabang |
| Invoice layanan | Wali utama untuk lihat tagihan; staf `service_invoice.manage`; wali sekunder tidak melihat angka |
| Kas dan pengeluaran | `finance.cash.read` / `finance.expense.create` |
| Audit | `audit.read` dengan batasan K22 dan 4.1 |

### 19.3 Pola pemeriksaan

Setiap operasi sensitif memeriksa berurutan:
1. Sesi valid dan pengguna aktif.
2. MFA terpenuhi bila izin termasuk K22.
3. Izin untuk operasi tersedia.
4. Cabang objek termasuk dalam cakupan pengguna.
5. Objek berada dalam cakupan pelanggan atau penugasan.
6. Status objek mengizinkan operasi.
7. Persetujuan tambahan tersedia bila diperlukan.

Kegagalan pemeriksaan 1–5 menghasilkan `NOT_FOUND` untuk objek yang tidak boleh dilihat, atau `FORBIDDEN` untuk tindakan yang tidak diizinkan pada objek yang boleh dilihat.

### 19.4 Rekam medis

- Penandatanganan hanya melalui `sign_medical_record`.
- Trigger basis data menolak `UPDATE` pada kolom klinis `medical_records` saat `status = SIGNED`.
- Trigger menolak `DELETE` pada rekam medis yang tidak `DRAFT`.
- Amendemen menyimpan snapshot lengkap sebelum dan sesudah.

### 19.5 Storage

| Bucket | Akses | Ukuran maks | Tipe berkas | Retensi |
|--------|-------|-------------|-------------|---------|
| `payment-proofs` | Privat | 5 MB | JPEG, PNG, PDF | 10 tahun |
| `medical-documents` | Privat | 20 MB | PDF, JPEG, PNG | 10 tahun sejak kunjungan terakhir |
| `product-images` | Publik baca | 2 MB | JPEG, PNG, WEBP | Selama produk aktif |
| `pet-photos` | Privat | 5 MB | JPEG, PNG, WEBP | Selama hewan terdaftar |

- Setiap berkas disimpan dengan nama acak (UUID), bukan nama asli.
- Unduhan dari bucket privat melalui signed URL berumur 5 menit yang dibuat server setelah pemeriksaan izin.
- Validasi tipe berkas dilakukan dengan memeriksa signature byte di server, bukan hanya ekstensi.
- Berkas tidak dipindai untuk virus dalam versi ini. Pembatasan tipe dan ukuran menjadi kontrol utama.

### 19.6 Audit

- Trigger mencatat perubahan pada tabel kritis (`sales`, `orders`, `payments`, `refunds`, `medical_records`, `hospitalizations`, `cash_movements`, `operating_expenses`, `product_prices`, `user_roles`, `user_permission_grants`, `pet_guardians`, `financial_periods`).
- Aplikasi mencatat aksi yang tidak tercakup trigger (login, gagal login, akses berkas, ekspor laporan).
- Audit tidak menyimpan kata sandi, token, nomor kartu, isi rekam medis, atau isi bukti pembayaran.
- `metadata` hanya berisi kolom yang berubah dan nilai lama atau baru untuk kolom non-sensitif.
- `audit_logs` tidak dapat diubah atau dihapus oleh peran aplikasi.
- Ekspor audit harian ke penyimpanan terpisah dengan akses terbatas pada pemilik dan developer on-call.

### 19.7 Pembatasan laju

- Login: 5 kegagalan per akun per 15 menit.
- Endpoint publik (registrasi, reset kata sandi): 10 request per IP per menit.
- Endpoint tulis pengguna: 60 request per pengguna per menit.
- Endpoint pembayaran dan konfirmasi: 20 request per pengguna per menit.
- Pelanggaran menghasilkan `RATE_LIMITED` dengan `Retry-After`.
- Penghitung disimpan di `rate_limit_counters` dan dibersihkan oleh job.

### 19.8 Keamanan aplikasi

- Header keamanan: `Content-Security-Policy` ketat, `Strict-Transport-Security`, `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`, `Permissions-Policy` membatasi akses kamera ke halaman unggah bukti.
- Cookie sesi: `HttpOnly`, `Secure`, `SameSite=Lax`.
- Validasi input dengan Zod di setiap Route Handler dan RPC wrapper.
- Output di-escape oleh React. Tidak ada `dangerouslySetInnerHTML` untuk data pengguna.
- Permintaan perubahan dari browser menggunakan token CSRF bawaan Next.js Server Actions atau header `Origin` yang diperiksa.
- Dependensi dipindai di CI dengan `npm audit`. Kerentanan kritis dan tinggi memblokir merge.
- Data pribadi dan klinis dihapus dari log, URL, pesan error, dan Sentry sebelum dikirim.

---

## 20. Kontrak API

### 20.1 Konvensi

- Prefix `/api/v1`. Format JSON. Encoding UTF-8.
- Waktu dalam ISO 8601 UTC.
- Uang dalam bentuk string desimal, misalnya `"125000.00"`.
- Header wajib: `Authorization` (cookie sesi), `Idempotency-Key` (operasi tulis penting), `X-Request-Id` (opsional dari klien, selalu dihasilkan ulang oleh server).

### 20.2 Format respons

Sukses:

```json
{
  "data": { "id": "uuid", "status": "COMPLETED" },
  "meta": { "request_id": "srv-01HZX..." }
}
```

Daftar:

```json
{
  "data": [],
  "meta": { "request_id": "srv-01HZX...", "next_cursor": null, "page_size": 20 }
}
```

Pagination memakai cursor, bukan offset, untuk daftar besar. `page_size` maksimal 100.

Error:

```json
{
  "error": {
    "code": "INSUFFICIENT_STOCK",
    "message": "Stok tersedia tidak mencukupi untuk satu atau lebih item.",
    "details": [{ "field": "items[0].quantity", "issue": "available=3" }],
    "request_id": "srv-01HZX..."
  }
}
```

`details` hanya untuk error validasi dan kekurangan stok, dan tidak pernah berisi data milik pelanggan lain.

### 20.3 Kode error

| Kode | HTTP | Makna |
|------|------|-------|
| `VALIDATION_ERROR` | 400 | Input tidak valid |
| `UNAUTHENTICATED` | 401 | Sesi tidak ada atau tidak valid |
| `MFA_REQUIRED` | 403 | MFA belum aktif untuk izin ini |
| `FORBIDDEN` | 403 | Izin tidak tersedia |
| `NOT_FOUND` | 404 | Tidak ditemukan atau di luar cakupan akses |
| `CONFLICT` | 409 | Konflik status atau data |
| `IDEMPOTENCY_CONFLICT` | 409 | Kunci dipakai untuk payload berbeda |
| `INVALID_STATE_TRANSITION` | 422 | Transisi status tidak diizinkan |
| `INSUFFICIENT_STOCK` | 422 | Stok tidak mencukupi |
| `INSUFFICIENT_CAPACITY` | 422 | Kapasitas penuh |
| `PAYMENT_REVIEW_REQUIRED` | 422 | Pembayaran harus diperiksa sebelum tindakan |
| `RATE_LIMITED` | 429 | Batas laju terlampaui |
| `INTERNAL_ERROR` | 500 | Kesalahan internal tanpa detail |

### 20.4 Endpoint

| Area | Endpoint |
|------|----------|
| Sesi | `GET /me`, `POST /auth/logout`, `POST /auth/mfa/enroll`, `POST /auth/mfa/verify` |
| Pengguna | `GET /users`, `POST /invitations`, `PATCH /users/{id}`, `POST /users/{id}/permission-grants` |
| Cabang | `GET /branches`, `PATCH /branches/{id}`, `GET /branches/{id}/payment-accounts` |
| Pelanggan dan hewan | `GET/POST /customers`, `PATCH /customers/{id}`, `GET/POST /pets`, `POST /pets/{id}/guardians`, `PATCH /pets/{id}/primary-guardian` |
| Produk | `GET/POST /products`, `PATCH /products/{id}`, `POST /products/{id}/prices`, `PATCH /products/{id}/branch-settings/{branch_id}` |
| Inventori | `GET /inventory/balances`, `POST /inventory/adjustments`, `POST /inventory/count-sessions`, `POST /inventory/transfers`, `POST /inventory/transfers/{id}/dispatch`, `POST /inventory/transfers/{id}/receive` |
| Pembelian | `POST /purchases`, `POST /purchases/{id}/submit`, `POST /purchases/{id}/receive`, `POST /supplier-returns` |
| POS | `POST /cash-shifts`, `POST /cash-shifts/{id}/close`, `POST /sales`, `PATCH /sales/{id}`, `POST /sales/{id}/complete`, `POST /sales/{id}/void`, `POST /sales/{id}/returns` |
| Toko online | `POST /orders`, `POST /orders/{id}/cancel`, `POST /orders/{id}/payments`, `POST /payments/{id}/confirm`, `POST /payments/{id}/reject`, `POST /orders/{id}/review`, `POST /orders/{id}/fulfillment` |
| Refund | `POST /refunds/{id}/approve`, `POST /refunds/{id}/reject`, `POST /refunds/{id}/complete` |
| Klinik | `GET/POST /schedules`, `POST /appointments`, `PATCH /appointments/{id}`, `POST /visits`, `POST /visits/{id}/complete`, `PATCH /medical-records/{id}`, `POST /medical-records/{id}/sign`, `POST /medical-records/{id}/amendments`, `POST /prescriptions`, `POST /prescriptions/{id}/sign`, `POST /medication-dispensings` |
| Rawat inap | `POST /hospitalizations`, `POST /hospitalizations/{id}/assignments`, `POST /hospitalizations/{id}/observations`, `POST /hospitalizations/{id}/tasks`, `POST /hospitalizations/{id}/shared-updates`, `POST /hospitalizations/{id}/discharge` |
| Grooming dan penitipan | `POST /service-bookings`, `PATCH /service-bookings/{id}`, `POST /service-bookings/{id}/check-in`, `POST /service-bookings/{id}/check-out`, `POST /service-booking-items/{id}/approval` |
| Tagihan layanan | `POST /service-invoices`, `POST /service-invoices/{id}/items`, `POST /service-invoices/{id}/payments`, `POST /service-invoices/{id}/void` |
| Keuangan | `POST /expenses`, `POST /expenses/{id}/approve`, `POST /financial-periods/{id}/close` |
| Laporan | `GET /reports/sales`, `GET /reports/online-sales`, `GET /reports/inventory`, `GET /reports/stock-expiry`, `GET /reports/clinic-doctors`, `GET /reports/receivables`, `GET /reports/cash-flow`, `GET /reports/profit-loss` |
| Audit | `GET /audit-logs` |

Setiap endpoint tulis mengembalikan `data` dengan ID dan status terbaru.

### 20.5 Aturan kontrak

- Harga final, subtotal, diskon, dan total tidak diterima dari klien. Klien mengirim item, jumlah, metode, dan pilihan diskon. Server menghitung.
- Jumlah pembayaran dari klien divalidasi terhadap total server.
- Kode alasan dikirim sebagai nilai dari daftar tetap, bukan teks bebas, kecuali kolom catatan.
- Respons tidak mengembalikan kolom internal seperti `internal_note`, `request_hash`, `unit_cost_snapshot` kecuali kepada pengguna yang memiliki izin terkait.
- Endpoint daftar memfilter cabang dan rentang waktu di server.

---

## 21. Antarmuka Pengguna

### 21.1 Navigasi per peran

| Peran | Menu |
|-------|------|
| Pemilik | Dashboard, POS, Inventori, Pembelian, Pesanan, Klinik, Rawat inap, Grooming & penitipan, Tagihan, Keuangan, Laporan, Pelanggan, Pengaturan |
| Dokter | Jadwal, Pasien, Kunjungan, Rekam medis, Resep, Rawat inap, Tugas klinis, Laporan saya |
| Admin | POS, Pesanan, Pembayaran, Pelanggan & hewan, Janji temu, Grooming & penitipan, Stok (baca), Pembelian, Tagihan layanan, Pengeluaran |
| Pelanggan | Beranda, Katalog, Keranjang, Pesanan saya, Hewan saya, Janji temu, Rawat inap, Tagihan |

### 21.2 Halaman wajib

| Halaman | Elemen |
|---------|--------|
| Login | Email, kata sandi, reset, MFA bila diperlukan, pesan error generik |
| Dashboard pemilik | Penjualan hari ini, kas per metode, stok minimum, batch kedaluwarsa 90 hari, janji temu hari ini, pesanan `STOCK_REVIEW`, piutang |
| Dashboard dokter | Jadwal hari ini, pasien, rekam medis `DRAFT`, rawat inap aktif, update yang belum dipublikasikan |
| Dashboard admin | Shift aktif, pembayaran `SUBMITTED`, pesanan siap diambil, booking hari ini, tagihan jatuh tempo |
| POS | Pencarian produk (nama, SKU, barcode), keranjang, diskon, total, pembayaran terpisah, cetak invoice |
| Inventori | Filter cabang dan lokasi, saldo, batch, mutasi, penyesuaian, transfer, stok opname |
| Detail hewan | Identitas, wali (utama dan sekunder), riwayat kunjungan, dokumen, rawat inap |
| Rekam medis | Form klinis, resep, status tanda tangan, amendemen, riwayat versi |
| Rawat inap | Denah unit, penempatan, observasi, tugas, update publik, proses pulang |
| Pesanan online | Item, pembayaran, bukti, status, review stok, fulfillment |
| Tagihan layanan | Item, pembayaran parsial, saldo, piutang |
| Keuangan | Kas per shift dan metode, pengeluaran, refund, periode |
| Portal pelanggan | Hewan, janji temu, pesanan, invoice, update rawat inap, persetujuan layanan tambahan |
| Pengaturan | Pengguna, undangan, izin, cabang, rekening pembayaran, metode pembayaran, kode alasan, template, job |

### 21.3 Perilaku UI

- Responsif: desain mobile-first. POS dirancang untuk tablet dengan area layar minimal 768 px. Tabel desktop berubah menjadi kartu di layar di bawah 768 px.
- Setiap halaman memiliki status loading, empty, success, error, dan forbidden.
- Operasi berisiko (void, retur, refund, penyesuaian, pembatalan pesanan, penutupan shift, penutupan periode) memakai dialog konfirmasi yang menyebutkan dampaknya.
- Tombol submit dinonaktifkan selama request berjalan. Idempotensi tetap diterapkan di server.
- Status ditampilkan dengan label teks dan warna, tidak hanya warna.
- Pencarian dan daftar besar memakai pagination dan pencarian server-side.
- Navigasi keyboard penuh, label form eksplisit, dan fokus terlihat. Target WCAG 2.2 AA.
- Format: Rupiah `Rp 1.250.000`; tanggal `DD/MM/YYYY`; waktu 24 jam dalam zona cabang.
- Notifikasi browser tidak menampilkan data medis.
- Form dengan lebih dari satu langkah menyimpan draft di server, bukan di browser.
- Di POS, dukungan pemindai barcode berbasis keyboard wedge: fokus input tetap di kolom pencarian.

### 21.4 Cetak dan invoice

- Invoice POS, pesanan online, dan invoice layanan memakai template HTML yang dapat dicetak.
- Invoice memuat: nomor, tanggal, cabang, alamat cabang, pelanggan (jika ada), item, diskon, total, metode pembayaran, saldo (untuk layanan), dan teks pajak sesuai K11.
- Nomor invoice ditampilkan dalam format K8.
- Invoice tidak memuat diagnosis atau isi klinis.

---

## 22. Pengujian dan Kriteria Penerimaan

### 22.1 Jenis pengujian

- Unit (Vitest): fungsi perhitungan harga, diskon, pembulatan, biaya rata-rata tertimbang, transisi status, validasi Zod.
- Basis data (pgTAP): constraint, trigger immutability, RLS per peran dan per cabang, fungsi RPC, idempotensi, dan konkurensi dua sesi.
- Integrasi (Vitest dengan Supabase lokal): Route Handler dan alur RPC dengan sesi nyata untuk setiap peran.
- E2E (Playwright): alur POS, pesanan online, rekam medis, rawat inap, dan portal, di desktop, tablet, dan ponsel.
- Keamanan: uji IDOR untuk setiap endpoint dengan ID milik pelanggan lain, serta uji pemeriksaan MFA.

Cakupan minimum: 85% baris untuk `src/modules/*` dan 100% untuk fungsi RPC transaksi.

### 22.2 Acceptance criteria

| Kode | Kriteria | Lulus bila |
|------|----------|------------|
| AC-01 | RBAC | Setiap peran hanya dapat memanggil endpoint dan RPC sesuai matriks 4.1 |
| AC-02 | Isolasi cabang | Pengguna cabang A mendapat `NOT_FOUND` untuk objek cabang B |
| AC-03 | Isolasi pelanggan | Pelanggan A tidak dapat membaca hewan, pesanan, invoice, atau rekam medis pelanggan B dengan mengubah ID |
| AC-04 | POS | Total dan stok benar; request dengan kunci idempotensi sama tidak membuat penjualan atau mutasi ganda |
| AC-05 | Konflik stok | Dua transaksi bersamaan atas stok 5 unit dengan permintaan 4 dan 3 tidak mengurangi stok di bawah nol dan tidak keduanya berhasil |
| AC-06 | Bukti pembayaran | Unggah bukti tidak mengubah `payment_status` menjadi `CONFIRMED` |
| AC-07 | Alokasi online | Reservasi hanya dibuat saat `confirm_order_payment` dan dengan pemeriksaan ulang |
| AC-08 | STOCK_REVIEW | Pesanan `STOCK_REVIEW` tidak dapat masuk `PROCESSING` tanpa `resolve_stock_review` |
| AC-09 | Rekam medis | Rekam medis `SIGNED` tidak dapat diubah langsung, dihapus, atau ditandatangani ulang |
| AC-10 | Kapasitas rawat | Penempatan tidak melampaui kapasitas unit pada rentang waktu bertabrakan |
| AC-11 | Update pelanggan | Pelanggan hanya melihat `shared_updates` yang dipublikasikan untuk hewannya |
| AC-12 | Keuangan | Kas, pengeluaran, piutang, dan HPP dihitung dari sumber terpisah dan tidak tercampur |
| AC-13 | Audit | Perubahan kritis memiliki pelaku, waktu, objek, dan alasan bila diwajibkan |
| AC-14 | Storage | Berkas privat tidak dapat diakses tanpa signed URL yang diterbitkan setelah pemeriksaan izin |
| AC-15 | Responsif | Alur POS, pesanan, dan janji temu dapat diselesaikan di ponsel, tablet, dan desktop |
| AC-16 | Pemulihan | Restore ke lingkungan uji menghasilkan jumlah baris dan fungsi kritis yang sama |
| AC-17 | Deployment | Build, tes, migration, dan rilis berjalan melalui pipeline yang tercatat |
| AC-18 | Konsistensi stok | Rekonsiliasi `on_hand` dengan ledger `inventory_movements` menghasilkan selisih nol |
| AC-19 | Kas | Total `cash_movements` per shift sesuai rekap penutupan shift |
| AC-20 | Batch | Penjualan dan pemakaian tidak mengambil batch kedaluwarsa |

### 22.3 Skenario wajib

1. Konflik stok: stok 5, POS menjual 4, pesanan online mengalokasikan 3 bersamaan. Hasil: total yang berhasil tidak melebihi 5.
2. Konfirmasi ganda: `confirm_order_payment` dipanggil dua kali dengan kunci sama. Hasil: satu pembayaran `CONFIRMED`, satu set reservasi, satu `cash_movement`.
3. Konfirmasi dengan payload berbeda dan kunci sama: `IDEMPOTENCY_CONFLICT`.
4. Admin mencoba menandatangani rekam medis: `FORBIDDEN`.
5. Dokter tidak ditugaskan mencoba mengubah rekam medis: `FORBIDDEN` atau `NOT_FOUND` sesuai cakupan.
6. Pelanggan mengganti ID hewan atau pesanan di URL dan payload: `NOT_FOUND`.
7. Admin mencoba membatalkan pesanan `PROCESSING` yang sudah dibayar tanpa refund: `INVALID_STATE_TRANSITION`.
8. Peminta refund mencoba menyetujui refund sendiri: `FORBIDDEN`.
9. Pemindahan unit ke unit penuh: `INSUFFICIENT_CAPACITY`.
10. Backup dipulihkan ke lingkungan uji: jumlah baris, akses RLS, berkas, dan fungsi kritis diverifikasi; durasi dicatat.

### 22.4 Gerbang rilis production

Rilis production hanya boleh dilakukan jika:
- Seluruh AC-01 sampai AC-20 lulus di staging.
- Tidak ada kerentanan kritis atau tinggi yang terbuka.
- Pengujian RLS mencakup setiap peran dan cabang dengan akun nyata.
- Migration sudah berjalan di staging tanpa error dan dengan rollback yang sudah diuji.
- Pemulihan backup sudah diuji dalam 30 hari terakhir.
- Pemilik menyetujui hasil UAT untuk POS, klinik, rawat inap, pesanan online, dan laporan.

---

## 23. Operasional

### 23.1 Backup dan pemulihan

- Basis data menggunakan PITR Supabase dengan retensi minimal 7 hari.
- Dump logis harian ke penyimpanan di luar Supabase dengan enkripsi; retensi 35 hari.
- Bucket Storage disalin harian ke penyimpanan terpisah dengan enkripsi; retensi versi 35 hari.
- Uji pemulihan dilakukan setiap bulan ke lingkungan uji terpisah. Hasil dicatat di runbook.
- Penanggung jawab pemulihan: developer lead, dengan pemilik sebagai penyetuju.
- Target: RPO 15 menit dan RTO 4 jam (K27).

### 23.2 Monitoring

Dipantau:
- Error server dan rasio request gagal per endpoint.
- Kegagalan login yang tidak biasa (lonjakan per IP atau per akun).
- Query basis data lambat di atas 500 ms untuk POS dan pembayaran.
- Konflik stok, `IDEMPOTENCY_CONFLICT`, dan deadlock.
- Kegagalan upload berkas.
- Kegagalan job terjadwal di `job_runs`.
- Kapasitas basis data (koneksi, CPU, disk) dan Storage.
- Status deployment dan health check `/api/health` yang memeriksa koneksi basis data.
- Selisih rekonsiliasi stok harian.

Alert dikirim ke email developer on-call dan pemilik untuk kategori kritis.

### 23.3 Insiden

Prosedur minimum:
1. Catat waktu deteksi, sumber, dan dampak.
2. Hentikan operasi yang berpotensi memperburuk inkonsistensi (misalnya nonaktifkan POS sementara melalui flag di `branch_settings`).
3. Simpan request ID dan log terkait. Jangan menyalin data pribadi ke tiket.
4. Rekonsiliasi stok, kas, dan pembayaran dengan query baca.
5. Perbaikan data dilakukan melalui RPC koreksi berotorisasi, bukan UPDATE langsung ke tabel produksi.
6. Pulihkan layanan.
7. Dokumentasikan penyebab, dampak, dan tindakan pencegahan dalam 5 hari kerja.

### 23.4 Retensi dan penghapusan

- Data bisnis dan klinis mengikuti K28.
- Log aplikasi disimpan 30 hari.
- Permintaan penghapusan data pribadi dari pelanggan diproses sebagai anonimisasi: nama, telepon, email, dan alamat diganti nilai tanda; baris transaksi dan klinis tetap ada.
- Setiap permintaan dicatat di audit.

### 23.5 Job terjadwal

| Job | Jadwal | Fungsi |
|-----|--------|--------|
| `void_stale_drafts` | Setiap jam | Mengubah draft POS yang lebih dari 24 jam menjadi `VOIDED` |
| `expire_idempotency_keys` | Harian | Menghapus kunci yang lebih dari 24 jam |
| `expire_invitations` | Harian | Menandai undangan kedaluwarsa |
| `stock_reconciliation` | Harian | Membandingkan saldo dengan ledger dan membuat alert selisih |
| `expiry_alerts` | Harian | Membuat notifikasi batch kedaluwarsa dalam 90 hari |
| `min_stock_alerts` | Harian | Membuat notifikasi stok minimum |
| `receivable_reminders` | Harian | Mengirim pengingat tagihan jatuh tempo |
| `audit_export` | Harian | Mengekspor audit ke penyimpanan terpisah |
| `backup_copy` | Harian | Dump logis dan salinan Storage |
| `rate_limit_cleanup` | Harian | Menghapus penghitung yang kedaluwarsa |

Setiap job menulis hasil ke `job_runs`. Job yang gagal dua kali berturut-turut menghasilkan alert.

---

## 24. CI/CD dan Deployment

### 24.1 Alur

1. Developer membuat branch dari `main`.
2. Pull request dibuka ke `main`.
3. GitHub Actions menjalankan: `npm ci`, `npm run lint`, `npm run typecheck`, `npm run test:unit`, `npm run test:db` (pgTAP pada Supabase lokal), `npm run test:integration`, `npm run build`.
4. Vercel membuat Preview Deployment yang terhubung ke proyek Supabase staging.
5. `npm run test:e2e` dijalankan terhadap Preview.
6. Review kode oleh minimal satu reviewer berbeda dari penulis.
7. Merge ke `main`.
8. Migration diterapkan otomatis ke staging.
9. Deployment staging berjalan. Smoke test dijalankan.
10. Deployment production hanya dimulai setelah persetujuan manual melalui GitHub Environment `production` dengan reviewer pemilik atau developer lead.
11. Migration production diterapkan sebelum deployment aplikasi baru, dan hanya jika kompatibel mundur dengan versi aplikasi sebelumnya.

### 24.2 Migration

- Semua perubahan skema melalui file migration Supabase CLI di `supabase/migrations`.
- Migration bersifat forward-only. Rollback dilakukan dengan migration baru.
- Perubahan berisiko memakai pola expand and contract: tambah struktur baru, deploy kode yang kompatibel, backfill, verifikasi, lalu hapus struktur lama pada rilis terpisah.
- Migration yang menghapus kolom, tabel, atau constraint memerlukan persetujuan pemilik dan catatan di pull request.
- Migration tidak dijalankan secara manual di production di luar pipeline, kecuali untuk prosedur insiden yang tercatat.

### 24.3 Vercel

- Region fungsi: `sin1`.
- Node.js 22.
- Branch production: `main`.
- Environment variables dipisah per lingkungan (Development, Preview, Production). Preview memakai variabel staging.
- Domain production `app.{domain}` dan staging `staging.{domain}` dengan HTTPS wajib.
- Redirect URL Supabase Auth hanya mengizinkan URL canonical (`APP_CANONICAL_URL`) untuk production dan staging. Preview Deployment tidak mengirim tautan email yang mengarah ke URL preview; pengujian alur email dilakukan di staging.
- Vercel Deployment Protection aktif untuk Preview.

### 24.4 Supabase

- Proyek production dan staging terpisah.
- Proyek production memakai PITR dan paket yang mendukungnya.
- Data staging sintetis. Data production tidak boleh disalin ke staging.
- Akses langsung ke database dari luar hanya melalui pooler Supabase yang dibatasi IP atau melalui jaringan tepercaya. Tidak ada akses publik langsung ke port PostgreSQL.
- Ekstensi yang dipakai: `pgcrypto` (UUID), `btree_gist` (constraint eksklusi), `pg_trgm` (pencarian nama), `pg_cron` (job).

---

## 25. Struktur Repository

```text
petshop-clinic/
├── src/
│   ├── app/
│   │   ├── (public)/
│   │   ├── (auth)/
│   │   ├── (owner)/
│   │   ├── (admin)/
│   │   ├── (doctor)/
│   │   ├── (customer)/
│   │   └── api/v1/
│   ├── components/
│   │   ├── ui/
│   │   ├── forms/
│   │   └── tables/
│   ├── modules/
│   │   ├── identity/
│   │   ├── branches/
│   │   ├── customers/
│   │   ├── products/
│   │   ├── inventory/
│   │   ├── purchasing/
│   │   ├── pos/
│   │   ├── online-orders/
│   │   ├── clinic/
│   │   ├── hospitalization/
│   │   ├── grooming/
│   │   ├── billing/
│   │   ├── finance/
│   │   ├── reports/
│   │   └── notifications/
│   ├── lib/
│   │   ├── server/          # satu-satunya tempat SUPABASE_SECRET_KEY
│   │   ├── supabase/
│   │   ├── auth/
│   │   ├── validation/
│   │   ├── money/           # pembulatan dan perhitungan uang
│   │   └── errors/
│   └── types/
├── supabase/
│   ├── migrations/
│   ├── functions/           # RPC dan fungsi database
│   ├── seed.sql             # data nonproduksi saja
│   └── tests/               # pgTAP
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
├── docs/
│   ├── permissions.md
│   ├── deployment.md
│   └── runbook.md
├── .github/workflows/
├── .env.example
├── package.json
└── README.md
```

Aturan:
- Modul bisnis tidak mengimpor satu sama lain secara langsung. Komunikasi melalui fungsi layanan di `modules/*/index.ts` atau RPC.
- Komponen presentasi tidak memanggil RPC mutasi langsung. Panggilan dilakukan melalui Server Action atau Route Handler.
- Linter melarang impor `src/lib/server` dari komponen klien.
- `any` dilarang pada kode transaksi, otorisasi, dan perhitungan uang.

---

## 26. Data Awal

Sebelum operasi dimulai, data berikut wajib tersedia dan diverifikasi oleh pemilik:

- Profil bisnis, daftar cabang, kode cabang, zona waktu, dan alamat.
- Akun pemilik (K25), pengguna staf dengan undangan, peran, dan cabang.
- Profil dokter dengan nomor STR untuk setiap dokter.
- Master produk: SKU, barcode, kategori, satuan, `track_lot`, `is_prescription`.
- Harga per cabang dan status jual POS dan online.
- Stok awal per lokasi dengan batch dan tanggal kedaluwarsa untuk produk `track_lot`, serta biaya rata-rata awal per cabang.
- Supplier.
- Pelanggan dan hewan yang datanya boleh dimigrasikan, lengkap dengan wali utama.
- Katalog layanan (klinik, grooming, penitipan) dengan harga per cabang.
- Unit rawat dan kapasitasnya.
- Metode pembayaran dan rekening atau instruksi manual per cabang.
- Kode alasan untuk penyesuaian, refund, pengeluaran, dan selisih kas.
- Saldo kas awal per cabang, piutang awal, dan tagihan supplier yang masih berlaku.
- Template invoice dan laporan.

Migrasi data awal dilakukan melalui skrip impor terverifikasi di staging, dengan rekonsiliasi jumlah dan nilai per cabang sebelum disetujui pemilik.

---

## 27. Definition of Done

Pekerjaan dianggap selesai bila seluruh kondisi berikut terpenuhi:

1. Seluruh modul dalam Bagian 3 tersedia dan memenuhi aturan di dokumen ini.
2. Skema, migration, constraint, RLS, dan fungsi RPC tersimpan di repository.
3. Seluruh acceptance criteria di Bagian 22 lulus di staging.
4. Hak akses setiap peran diverifikasi di server dan basis data dengan akun nyata.
5. Stok, pembayaran, dan kas tidak menunjukkan inkonsistensi dalam pengujian konkurensi.
6. Rekam medis yang sudah ditandatangani tidak dapat diubah langsung.
7. Laporan dapat direkonsiliasi ke transaksi sumber.
8. Deployment production, backup, pemulihan, monitoring, dan prosedur insiden terdokumentasi di `docs/`.
9. Tidak ada secret di source code, log, atau browser.
10. Pemilik menyetujui UAT untuk POS, klinik, rawat inap, pesanan online, dan laporan, dan menandatangani berita acara serah terima.

---

## 28. Catatan Status Dokumen

Dokumen ini adalah spesifikasi implementasi, bukan bukti bahwa kode, database, atau deployment sudah dibuat. Repository, migration, fungsi, tes, dan konfigurasi Vercel maupun Supabase harus dibangun dan diverifikasi terhadap spesifikasi ini. Setiap perubahan terhadap keputusan di Bagian 1 harus dilakukan dengan merevisi dokumen ini dan mencatat alasannya.
