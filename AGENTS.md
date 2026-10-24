# AGENTS.md

Dokumen ini mengatur perilaku agen AI (coding agent) dalam mengembangkan Sistem Operasional Pet Shop dan Klinik Hewan dari awal sampai siap produksi. Sumber kebenaran tunggal untuk fitur dan aturan bisnis adalah **Spesifikasi Lengkap Sistem Operasional Pet Shop dan Klinik Hewan Versi 3.0**. Jika instruksi di file ini bertentangan dengan spesifikasi, spesifikasi yang berlaku, dan agen wajib melaporkan konflik tersebut sebelum melanjutkan.

---

## 1. Identitas Proyek

- Nama proyek: `petshop-clinic`
- Bahasa antarmuka dan pesan ke pengguna akhir: Bahasa Indonesia.
- Bahasa kode, nama variabel, nama tabel, kolom, komentar teknis, dan commit message: Bahasa Inggris.
- Stack: Next.js (App Router) + TypeScript strict, Tailwind CSS, Radix UI, Zod, React Hook Form, Supabase (PostgreSQL 15, Auth, Storage, RLS), Resend, Sentry, Vitest, pgTAP, Playwright.
- Hosting: Vercel (region `sin1`), Supabase (region `ap-southeast-1`).
- Repository: GitHub, branch `main` sebagai basis.

---

## 2. Prinsip Perilaku Agen

1. **Spesifikasi adalah sumber kebenaran.** Jangan menambah fitur, aturan, atau endpoint yang tidak ada di spesifikasi. Jika ada kebutuhan yang tidak tercakup, hentikan dan tanyakan kepada pengguna.
2. **Jangan menebak keputusan bisnis.** Semua keputusan sudah dikunci di Bagian 1 spesifikasi (K1–K30). Jangan mengubah keputusan tersebut.
3. **Jujur tentang status.** Jangan pernah menyatakan fitur selesai, teruji, atau ter-deploy jika belum dijalankan dan diverifikasi. Laporkan dengan jelas apa yang sudah lulus, apa yang gagal, dan apa yang belum dikerjakan.
4. **Keamanan lebih utama dari kecepatan.** Setiap perubahan pada otorisasi, RLS, RPC transaksi, stok, pembayaran, kas, dan rekam medis wajib disertai tes.
5. **Perubahan kecil dan terverifikasi.** Kerjakan satu unit kerja kecil, jalankan pemeriksaan, lalu lanjut. Jangan menumpuk banyak perubahan tanpa verifikasi.
6. **Jangan menghapus atau menonaktifkan tes** agar build lulus. Jika tes salah, perbaiki tes dengan alasan tertulis di pull request.
7. **Jangan membuat asumsi tersembunyi.** Jika mengambil asumsi teknis yang kecil, tulis di deskripsi pull request.
8. **Hindari dokumentasi berlebihan.** Hanya tulis atau perbarui dokumen yang diminta atau yang tercantum di Bagian 9.

---

## 3. Larangan Mutlak

Agen **dilarang**:

- Menulis `SUPABASE_SECRET_KEY` atau secret apa pun ke source code, file `.env` yang di-commit, log, komentar, fixture, atau dokumen.
- Memakai prefix `NEXT_PUBLIC_` untuk secret.
- Mengimpor `src/lib/server` dari komponen klien (`"use client"`) atau dari modul yang dapat dibundel ke browser.
- Menonaktifkan RLS pada tabel apa pun, atau membuat kebijakan `USING (true)` untuk tabel bisnis.
- Membuat kebijakan yang hanya memeriksa `auth.uid() IS NOT NULL` untuk tabel sensitif.
- Memberikan grant `UPDATE` atau `DELETE` pada tabel append-only (lihat spesifikasi 17.1).
- Menambahkan `ON DELETE CASCADE`.
- Menghitung total, harga final, diskon, atau biaya hanya di browser atau menerima total dari klien sebagai sumber kebenaran.
- Mengurangi stok, mengubah saldo kas, atau mengubah status pembayaran lewat `UPDATE` langsung dari aplikasi. Semua lewat RPC atomik.
- Mengubah kolom klinis rekam medis yang berstatus `SIGNED` atau menghapus rekam medis non-`DRAFT`.
- Mengubah data produksi dengan SQL manual di luar prosedur insiden.
- Menyalin data produksi ke staging atau preview.
- Menggunakan `any` pada kode transaksi, otorisasi, atau perhitungan uang.
- Menonaktifkan ESLint, TypeScript strict, atau pemeriksaan tipe dengan `@ts-ignore` tanpa alasan dan persetujuan.
- Menjalankan migration destruktif (drop kolom, tabel, constraint) tanpa persetujuan pemilik yang tercatat.
- Membuat fitur di luar cakupan Bagian 2 dan Bagian 3 spesifikasi (misalnya payment gateway, WhatsApp, perhitungan pajak, multi-mata uang).
- Melakukan `git push --force` ke `main`.
- Men-deploy ke production tanpa persetujuan manual.

---

## 4. Urutan Kerja Wajib

Agen mengerjakan fase teknis berikut secara berurutan. Setiap tahap harus lulus gerbang sebelum tahap berikutnya dimulai. Ini adalah urutan kerja teknis, bukan roadmap bisnis.

### Tahap A — Fondasi
1. Inisialisasi proyek Next.js + TypeScript strict di `petshop-clinic/`.
2. Konfigurasi ESLint, Prettier, `tsconfig` strict, Vitest, dan script `package.json` sesuai Bagian 24.1.
3. Konfigurasi Supabase CLI dan proyek lokal.
4. Buat `.env.example` sesuai Bagian 2.3 spesifikasi tanpa nilai rahasia.
5. Siapkan workflow GitHub Actions dasar (lint, typecheck, unit test, build).

**Gerbang A:** `npm ci`, `npm run lint`, `npm run typecheck`, `npm run test:unit`, dan `npm run build` lulus.

### Tahap B — Identitas, Cabang, dan Keamanan Dasar
1. Migration: `branches`, `branch_settings`, `profiles`, `roles`, `permissions`, `role_permissions`, `user_roles`, `user_branch_access`, `user_permission_grants`, `invitations`, `audit_logs`, `doctor_profiles`, `idempotency_keys`, `rate_limit_counters`, `job_runs`, `notifications`.
2. Seed peran (`OWNER`, `DOCTOR`, `ADMIN`, `CUSTOMER`) dan izin sesuai matriks 4.1.
3. Fungsi `has_permission` dan `can_access_branch` dengan `SECURITY DEFINER`, `search_path` tetap.
4. RLS untuk semua tabel di atas.
5. Trigger audit dan trigger immutability.
6. Autentikasi: login, logout, reset kata sandi, MFA TOTP, undangan staf, pembatasan login (K22–K26, 4.4, 4.5).
7. Skrip setup pemilik pertama (K25) yang hanya bisa dijalankan sekali dan tercatat.
8. Middleware route protection dan halaman `forbidden`.

**Gerbang B:** Tes pgTAP untuk RLS per peran dan per cabang lulus. Tes IDOR lulus. MFA terverifikasi untuk izin dalam K22.

### Tahap C — Master Data dan Inventori
1. Migration: `customers`, `customer_addresses`, `pets`, `pet_guardians`, `pet_documents`, `product_categories`, `products`, `product_barcodes`, `product_branch_settings`, `product_prices`, `suppliers`, `inventory_locations`, `inventory_lots`, `stock_balances`, `inventory_movements`, `stock_reservations`, `inventory_adjustments`, `count_sessions`, `count_session_items`, `inventory_transfers`, `inventory_transfer_items`, `product_branch_costs`, `cost_history`, `purchase_orders`, `purchase_order_items`, `goods_receipts`, `goods_receipt_items`, `supplier_returns`, `supplier_return_items`, `supplier_bills`, `supplier_payments`.
2. Trigger append-only dan constraint `CHECK` stok.
3. RPC: `apply_inventory_adjustment`, `create_transfer`, `dispatch_transfer`, `receive_transfer`, `receive_goods`, `record_supplier_return`.
4. Implementasi FEFO dan biaya rata-rata tertimbang (6.7, 6.8) di modul `inventory` dengan unit test untuk perhitungan.
5. Rekonsiliasi stok (`stock_reconciliation`).
6. Penyimpanan berkas: bucket `product-images`, `pet-photos`.

**Gerbang C:** Tes konkurensi: dua penerimaan dan dua penyesuaian bersamaan menghasilkan saldo dan biaya yang benar. AC-18 lulus. Tes batch kedaluwarsa lulus.

### Tahap D — POS dan Pembayaran Toko
1. Migration: `cash_shifts`, `sales`, `sale_items`, `sale_payments`, `sales_returns`, `sales_return_items`, `refunds`, `payment_methods`, `branch_payment_accounts`, `invoice_sequences`, `cash_movements`.
2. RPC: `open_cash_shift`, `close_cash_shift`, `complete_sale`, `create_sales_return`, `generate_invoice_number`.
3. Perhitungan harga, diskon, dan pembulatan di `src/lib/money` dengan unit test lengkap, termasuk kasus diskon 10% dan 10,01%, pembulatan half-up, dan total tidak negatif.
4. Idempotensi (18.4) dengan tabel `idempotency_keys` dan header `Idempotency-Key`.
5. Pembayaran toko online: `orders`, `order_items`, `payments`, `customer_carts`, `customer_cart_items`, `order_fulfillments`, `order_review_actions`, serta RPC `confirm_order_payment`, `reject_order_payment`, `resolve_stock_review`, `advance_fulfillment`.
6. Refund: `approve_refund`, `reject_refund`, `complete_refund`, dengan pemisahan tugas (K19).
7. Invoice POS dan pesanan online dalam format HTML cetak (21.4).

**Gerbang D:** AC-04, AC-05, AC-06, AC-07, AC-08 lulus. Skenario wajib 1, 2, 3, 7, dan 8 pada 22.3 lulus.

### Tahap E — Klinik, Rawat Inap, Grooming, dan Tagihan Layanan
1. Migration: `service_catalog`, `doctor_schedules`, `appointments`, `visits`, `medical_records`, `medical_record_amendments`, `prescriptions`, `prescription_items`, `treatment_records`, `medication_dispensings`, `care_units`, `hospitalizations`, `hospitalization_assignments`, `care_observations`, `care_tasks`, `shared_updates`, `service_bookings`, `service_booking_items`, `boarding_stays`, `service_capacity`, `service_invoices`, `service_invoice_items`, `service_payments`, `customer_credits`, `operating_expenses`, `financial_periods`.
2. Constraint eksklusi janji temu dokter (17.4).
3. RPC: `sign_medical_record`, `amend_medical_record`, `dispense_medication`, `admit_hospitalization`, `transfer_unit`, `discharge_hospitalization`, `publish_shared_update`, `reserve_booking_slot`, `create_service_invoice_from_source`, `record_service_payment`.
4. Trigger immutability rekam medis dan pencegahan `DELETE` non-`DRAFT`.
5. Portal pelanggan untuk janji temu, booking, rekam medis ringkas, update rawat inap, dan persetujuan layanan tambahan (Bagian 15).
6. Laporan layanan dokter (10.7) tanpa perhitungan gaji atau komisi.

**Gerbang E:** AC-09, AC-10, AC-11 lulus. Skenario wajib 4, 5, 6, dan 9 pada 22.3 lulus. Pengujian bahwa `internal_note` dan observasi internal tidak pernah muncul di respons pelanggan.

### Tahap F — Keuangan, Laporan, Notifikasi, dan Operasional
1. Modul `finance`: buku kas, pengeluaran dengan persetujuan (K18), periode keuangan, penutupan periode.
2. Modul `reports` sesuai definisi 14.4, termasuk ekspor CSV dan PDF.
3. Modul `notifications` sesuai Bagian 16 dengan aturan isi email tanpa data medis.
4. Job terjadwal (23.5) melalui `pg_cron` atau Supabase Edge Function, dengan pencatatan `job_runs`.
5. Pembatasan laju (19.7) dan header keamanan (19.8).
6. Audit ekspor harian dan endpoint `/api/health`.
7. Sentry dengan pembersihan data pribadi.

**Gerbang F:** AC-12, AC-13, AC-14, AC-18, AC-19 lulus.

### Tahap G — Pengujian Akhir dan Kesiapan Rilis
1. E2E Playwright untuk alur POS, pesanan online, rekam medis, rawat inap, dan portal di desktop, tablet, dan ponsel (AC-15).
2. Uji pemulihan backup ke lingkungan uji (AC-16, 23.1).
3. Verifikasi seluruh AC-01 sampai AC-20.
4. Siapkan `docs/runbook.md`, `docs/deployment.md`, dan `docs/permissions.md`.
5. Siapkan berita acara UAT sebagai templat untuk pemilik. Jangan mengisinya sendiri.

**Gerbang G:** Seluruh gerbang rilis pada 22.4 terpenuhi. Pengajuan ke pemilik untuk UAT dan persetujuan.

---

## 5. Aturan Kerja Setiap Tugas

Untuk setiap tugas, agen wajib mengikuti langkah berikut:

1. **Baca** bagian spesifikasi yang relevan dan tuliskan ringkasan kebutuhan dalam 3–5 poin.
2. **Periksa** kode dan migration yang sudah ada agar tidak terjadi duplikasi.
3. **Rencanakan** perubahan: file yang akan diubah, migration yang dibutuhkan, dan tes yang akan ditulis.
4. **Tulis tes dulu** untuk aturan bisnis, otorisasi, dan transaksi (test-first untuk bagian kritis).
5. **Implementasikan** perubahan seminimal mungkin.
6. **Jalankan** `npm run lint`, `npm run typecheck`, `npm run test:unit`, dan untuk perubahan database `npm run test:db`.
7. **Laporkan** hasil dengan format di bagian 7.
8. **Jangan** melanjutkan ke tugas berikutnya jika ada tes yang gagal di tugas saat ini.

---

## 6. Standar Kode

### 6.1 TypeScript dan React
- `strict: true`, `noUncheckedIndexedAccess: true`, `exactOptionalPropertyTypes: true`.
- Komponen server secara default. Tambahkan `"use client"` hanya bila diperlukan interaktivitas.
- Komponen presentasi tidak memanggil RPC mutasi secara langsung.
- Gunakan Server Actions atau Route Handler untuk mutasi, dengan validasi Zod.
- Nama file `kebab-case`. Nama komponen `PascalCase`. Nama fungsi dan variabel `camelCase`. Nama tabel dan kolom `snake_case`.

### 6.2 Validasi
- Setiap input dari klien divalidasi dengan Zod di Route Handler atau Server Action.
- Schema Zod untuk bentuk input yang sama dipakai bersama di klien dan server, tetapi validasi server tidak boleh dilewati.

### 6.3 Uang dan kuantitas
- Gunakan tipe `Money` atau fungsi di `src/lib/money`. Jangan gunakan `number` JavaScript untuk nilai uang yang disimpan. Gunakan string desimal atau library desimal presisi.
- Pembulatan transaksi ke rupiah penuh dengan half-up (K11).
- Biaya satuan memakai presisi 4 desimal (K12).

### 6.4 Error
- Semua error domain menggunakan kode dari 20.3.
- Jangan mengirim stack trace, detail SQL, atau pesan internal ke klien.
- Gunakan `request_id` dari server pada setiap respons error.

### 6.5 Database
- Setiap perubahan skema melalui migration Supabase CLI. Jangan mengedit migration yang sudah pernah di-merge; buat migration baru.
- Setiap tabel bisnis memiliki `id`, `created_at`, `created_by`, dan bila relevan `updated_at`, `updated_by`.
- Setiap RPC mutasi memanggil `auth.uid()`, memeriksa izin, cabang, status, dan idempotensi (18.1).
- Urutan penguncian mengikuti 18.3.

### 6.6 Keamanan
- Jangan menaruh data klinis di log, URL, atau pesan error.
- Unduhan berkas memakai signed URL berumur 5 menit yang dibuat server setelah pemeriksaan izin.
- Validasi tipe berkas dengan signature byte di server (19.5).

### 6.7 Penamaan commit dan branch
- Branch: `feat/<modul>-<ringkas>`, `fix/<modul>-<ringkas>`, `chore/<ringkas>`, `db/<ringkas>`.
- Commit: Conventional Commits dalam bahasa Inggris, misalnya `feat(pos): add complete_sale rpc`.

---

## 7. Format Laporan Agen

Setiap selesai satu tugas, agen menulis laporan dengan struktur ini:
Ringkasan
[satu paragraf singkat]
Bagian spesifikasi yang dipenuhi
[kode bagian, misal 8.2, 9.5]
Perubahan
[file dan migration yang berubah]
Hasil verifikasi
lint: lulus / gagal
typecheck: lulus / gagal
unit: X lulus, Y gagal
db (pgTAP): lulus / gagal / tidak dijalankan (alasan)
integrasi/E2E: lulus / gagal / tidak dijalankan (alasan)
Asumsi teknis
[asumsi kecil yang diambil, atau "tidak ada"]
Belum dikerjakan / risiko
[daftar, atau "tidak ada"]

Agen tidak boleh menulis "selesai" tanpa mencantumkan hasil verifikasi yang sebenarnya.

---

## 8. Penanganan Konflik dan Ketidakpastian

- Jika spesifikasi tidak jelas atau bertentangan dengan dirinya sendiri, hentikan tugas dan ajukan pertanyaan spesifik dengan opsi yang diusulkan. Jangan memilih diam-diam.
- Jika tes yang sudah ada gagal setelah perubahan, perbaiki perubahan, bukan tesnya, kecuali tes jelas melanggar spesifikasi.
- Jika menemukan celah keamanan atau inkonsistensi data di kode yang sudah ada, laporkan segera dengan prioritas tinggi dan jangan menunda ke tugas berikutnya.
- Jika tugas membutuhkan akses ke layanan eksternal yang belum dikonfigurasi (Supabase production, Resend, Sentry), berhenti dan minta kredensial melalui prosedur aman. Jangan meminta secret dalam percakapan.
- Jika sebuah perintah destruktif diperlukan (reset database, hapus data staging), konfirmasi dulu kepada pengguna.

---

## 9. Dokumen yang Boleh Dibuat atau Diperbarui

Hanya dokumen berikut yang boleh ditulis agen:
- `AGENTS.md` (file ini), hanya oleh pemilik proyek.
- `README.md` (cara menjalankan lokal).
- `docs/permissions.md` (sinkron dengan migration seed izin).
- `docs/deployment.md` (langkah deployment nyata yang sudah diverifikasi).
- `docs/runbook.md` (backup, restore, insiden).
- Dokumentasi API otomatis dari kode jika diperlukan.

Agen tidak membuat dokumen tambahan seperti ringkasan status, laporan naratif, atau dokumen perencanaan kecuali diminta.

---

## 10. Definisi Selesai untuk Agen

Sebuah tugas dianggap selesai jika:
1. Kode dan migration tersimpan di branch dan lolos CI.
2. Tes yang relevan ditulis dan lulus.
3. Laporan agen sesuai bagian 7 sudah dibuat.
4. Tidak ada larangan di bagian 3 yang dilanggar.
5. Perubahan pada keamanan atau transaksi sudah ditinjau oleh reviewer manusia.

Sebuah tahap dianggap selesai jika seluruh gerbangnya di bagian 4 lulus.

Proyek dianggap siap rilis hanya jika Definisi Selesai di Bagian 27 spesifikasi terpenuhi dan pemilik menyetujui UAT secara tertulis.
