# Dokumentasi Kode Sekarang + Rencana Awal Multitenant

kondisi codebase hari ini, apa yang udah jalan buat tenant, bagian yang masih rawan, dan konsep multitenant yang aku pegang sebagai acuan.

## Gambaran cepat proyek

Stack utama:

- Laravel 12
- Filament 3.3
- Livewire 3
- MySQL multi-connection

Fitur yang aktif di sistem:

- POS / transaksi
- produk dan inventori
- cash flow
- buyback
- approval
- cetak PDF (invoice/barcode)
- printer thermal

## Alur tenant yang berjalan sekarang

Secara request flow, tenant sudah di-resolve dari awal request:

1. `TenantResolver` dipasang global di `bootstrap/app.php`.
2. Resolver baca tenant dari:
- `STORE_CODE` di `.env` (prioritas pertama)
- mapping host/domain di `config/tenants.php`
- fallback ke default tenant
3. `TenantService` nyimpen context runtime:
- `currentStore`
- `currentStoreCode`
- `isAuditMode`
4. Beberapa config juga di-set dinamis per tenant:
- cache prefix
- nama cookie session per host
- path log per store

## Yang sudah beres terkait multitenant

### 1. Konfigurasi tenant

Sudah ada single source di `config/tenants.php` untuk:

- mapping domain -> store code
- definisi store
- grouping store
- mapping store -> DB connection

### 2. Multi koneksi database

Di `config/database.php` sudah ada koneksi terpisah:

- `wates`
- `sentolo`
- `member` (shared)

Model yang pakai trait `BelongsToTenant` akan ambil koneksi tenant aktif.

### 3. Isolasi data level row (`store_code`)

Kolom `store_code` sudah ditambah di tabel transaksional utama:

- `transactions`
- `buybacks`
- `inventories`
- `cash_flows`
- `products`

Ref migration:
`database/migrations/2026_01_06_000001_add_store_code_to_transactional_tables.php`

### 4. Isolasi file/storage

Sudah ada helper `TenantStorage` (`app/Helpers/TenantStorage.php`).
Struktur path file mengikuti pola:
`storage/app/public/{store_code}/...`

### 5. Status Soft Delete

Saat ini soft delete masih kepakai, bukan hard delete full.

Model yang pakai `SoftDeletes`:

- `Category`
- `Product`
- `PaymentMethod`
- `Transaction`
- `Report`

Tabel yang punya `deleted_at`:

- `categories`
- `products`
- `payment_methods`
- `transactions`
- `sub_categories`

## Gap yang masih perlu diberesin

Bagian ini yang paling berpengaruh ke keamanan data antar tenant.

1. Tenant scope belum dipaksa global.
- Trait `BelongsToTenant` sudah punya `scopeCurrentStore()`, tapi belum dijadikan global scope default.
- Efeknya: query yang lupa filter `store_code` masih bisa kebaca lintas store (terutama store yang share DB yang sama).

2. Masih ada query raw yang belum tenant-aware.
- Contoh: widget yang query `cash_flows` pakai `DB::table(...)` tanpa filter `store_code`.
- Cache key widget juga ada yang belum include store code.

3. Model shared belum konsisten.
- Trait `BelongsToMemberDatabase` sudah ada, tapi model `Member` belum pakai trait itu.
- Ini rawan bikin koneksi data member tidak konsisten.

4. Endpoint maintenance masih terbuka via web.
- Route `/update-server` ngejalanin migrate + clear cache via HTTP.
- Untuk production/multitenant ini terlalu riskan.

5. Beberapa tempat masih hard-coded tenant list.
- Contoh di `StorageController::serveStore()`.
- Harusnya ambil dari config tenant, bukan array manual.

## Arah arsitektur yang dipilih (awal)

Untuk kondisi sekarang, paling masuk akal tetap pakai model hybrid:

- isolasi database per group store
- isolasi row pakai `store_code` untuk store yang ada di DB yang sama

Kenapa ini dipilih:
- nggak butuh rewrite besar
- bisa dinaikkan level keamanan tenant sekarang
- tetap fleksibel kalau nanti mau full DB per tenant

## Konsep multitenant yang mau diterapkan diawal

Bagian ini bukan daftar kerja, tapi patokan yang aku pakai waktu ngerjain multitenant.

### 1) Model tenancy: hybrid

Aku pakai model hybrid. Lapisan pertama dipisah per group database (contoh `wates` dan `sentolo`). Lapisan kedua, tenant yang masih satu DB tetap dipisah pakai `store_code`. Jadi isolasinya tetap aman tanpa bongkar sistem dari nol.

### 2) Boundary data: tenant vs shared

Data aku bedain jelas jadi dua: data tenant dan data shared. Data tenant itu transaksi, produk, inventori, cash flow, buyback, dan laporan operasional. Data shared itu member (dan nanti entitas lintas toko lain kalau memang dibutuhkan). Implikasinya simpel: model tenant harus ikut tenant aktif, model shared harus pakai koneksi shared khusus.

### 3) Aturan akses data tenant

Default sistem harus `tenant-safe`. Artinya query tenant otomatis ke tenant aktif, bukan sebaliknya. Query lintas tenant nggak boleh jadi perilaku default. Kalau ada kebutuhan lintas tenant, itu harus explicit dan terkendali (misalnya mode khusus super admin).

### 4) Resolusi tenant sebagai acuan utama

Tenant ditentukan di awal request, lalu jadi acuan utama sampai request selesai. Resolve-nya dari `STORE_CODE` atau mapping domain, disimpan di `TenantService`, terus dipakai konsisten ke koneksi DB, filter data, cache key, dan storage path. Intinya semua modul baca sumber yang sama, jangan bikin logika tenant sendiri-sendiri.

### 5) Isolasi non-data (cache, session, file, log)

Yang diisolasi bukan cuma tabel data. Cache, session, file, dan log juga harus kepisah per tenant. Makanya cache prefix per store, cookie session per host/domain, storage path per store, dan log file per store wajib konsisten. Ini ngurangin risiko tabrakan data/runtime antar tenant.

### 6) Keamanan operasional

Operasi sensitif kayak migrate, clear cache, atau bootstrap tenant jangan dibuka lewat endpoint web publik. Jalurnya lewat command atau CI aja. Endpoint web cukup buat fitur bisnis aplikasi.

### 7) Arah jangka menengah

Kalau fondasi ini udah stabil, aku tetap punya dua opsi: lanjut ke full database per tenant atau tetap di hybrid kalau itu paling cocok buat operasional. Yang penting boundary tenant udah kuat dari sekarang, jadi kalau nanti ada perubahan arsitektur, migrasinya tetap aman.
