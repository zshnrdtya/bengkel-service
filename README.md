# Sistem Booking Service Bengkel

> Aplikasi web untuk manajemen booking service bengkel dengan 3 role: User, Mekanik, dan Admin.

---

## Gambaran Umum Aplikasi

Aplikasi ini adalah **Sistem Booking Service Bengkel** berbasis web yang dibuat menggunakan **Laravel 13** (framework PHP).

Aplikasi memiliki **3 role pengguna**:
| Role | Akses |
|------|-------|
| **User / Pelanggan** | Daftar, login, buat booking, lihat nomor antrian, batalkan booking |
| **Mekanik** | Lihat daftar service yang ditugaskan, update status pengerjaan |
| **Admin** | Kelola semua data: booking, service, barang, pelanggan, mekanik, transaksi |

---

## Struktur Folder Penting

```
app/
├── Http/
│   ├── Controllers/        ← Logika utama aplikasi
│   │   ├── AuthController.php
│   │   ├── UserController.php
│   │   ├── MekanikController.php
│   │   └── AdminController.php
│   └── Middleware/
│       └── RoleMiddleware.php  ← Penjaga akses berdasarkan role
├── Models/                 ← Representasi tabel database
│   ├── User.php
│   ├── Booking.php
│   ├── Service.php
│   ├── Barang.php
│   └── Transaksi.php
database/
├── migrations/             ← Struktur tabel database
└── seeders/
    └── DatabaseSeeder.php  ← Data awal / dummy
resources/views/            ← Tampilan HTML (Blade template)
routes/
└── web.php                 ← Daftar semua URL / halaman
bootstrap/
└── app.php                 ← Konfigurasi awal aplikasi
```

---

## 1. Database & Migrations

Migrations adalah cara Laravel membuat tabel di database secara otomatis lewat kode PHP, tanpa perlu nulis SQL manual.

### Tabel yang dibuat:

**`users`** — menyimpan semua pengguna (pelanggan, mekanik, admin)
```
id | name | email | password | role | telepon | alamat | created_at
```
- Kolom `role` berisi nilai: `user`, `mekanik`, atau `admin`
- Satu tabel untuk semua role, dibedakan lewat kolom `role`

**`services`** — jenis-jenis layanan bengkel
```
id | nama_service | deskripsi | harga | estimasi_jam | aktif
```

**`barangs`** — stok suku cadang / barang bengkel
```
id | nama_barang | kode_barang | stok | harga | satuan
```

**`bookings`** — data pemesanan service oleh pelanggan
```
id | nomor_antrian | user_id | service_id | mekanik_id | tanggal_booking
   | kendaraan | plat_nomor | keluhan | status | catatan_mekanik | total_harga
```
- `user_id` → siapa yang booking (relasi ke tabel users)
- `service_id` → service apa yang dipesan (relasi ke tabel services)
- `mekanik_id` → mekanik mana yang ditugaskan (relasi ke tabel users)
- `status` bisa berisi: `menunggu`, `diproses`, `selesai`, `dibatalkan`

**`transaksis`** — data pembayaran
```
id | nomor_transaksi | booking_id | total | status_bayar | metode_bayar | dibayar_at
```
- Dibuat otomatis saat mekanik menandai booking sebagai `selesai`

---

## 2. Models (`app/Models/`)

Model adalah kelas PHP yang merepresentasikan satu tabel di database. Lewat model, kita bisa ambil, simpan, dan hapus data tanpa nulis SQL.

### `User.php`
```php
protected $fillable = ['name', 'email', 'password', 'role', 'telepon', 'alamat'];
```
- `$fillable` → kolom mana saja yang boleh diisi saat create/update data

```php
public function bookings() {
    return $this->hasMany(Booking::class);
}
```
- Relasi: satu user bisa punya banyak booking (one-to-many)

```php
public function isAdmin(): bool { return $this->role === 'admin'; }
```
- Helper method untuk cek role, dipakai di view dan controller

### `Booking.php`
```php
public static function generateNomorAntrian(): string {
    $prefix = 'ANT-' . date('Ymd') . '-';
    $last   = self::whereDate('created_at', today())->count() + 1;
    return $prefix . str_pad($last, 3, '0', STR_PAD_LEFT);
}
```
- Fungsi untuk buat nomor antrian otomatis, contoh hasilnya: `ANT-20260424-001`
- `date('Ymd')` → tanggal hari ini
- `str_pad(..., 3, '0', STR_PAD_LEFT)` → angka urut 3 digit, misal `001`, `002`

```php
public function mekanik() {
    return $this->belongsTo(User::class, 'mekanik_id');
}
```
- Relasi ke tabel users tapi lewat kolom `mekanik_id`, bukan `user_id`

### `Transaksi.php`
```php
public static function generateNomor(): string {
    $prefix = 'TRX-' . date('Ymd') . '-';
    ...
}
```
- Sama seperti nomor antrian, tapi untuk nomor transaksi. Contoh: `TRX-20260424-001`

---

## 3. Middleware (`app/Http/Middleware/RoleMiddleware.php`)

Middleware adalah "penjaga pintu" sebelum request masuk ke controller.

```php
public function handle(Request $request, Closure $next, string ...$roles)
{
    if (!Auth::check() || !in_array(Auth::user()->role, $roles)) {
        abort(403, 'Akses ditolak.');
    }
    return $next($request);
}
```

**Cara kerjanya:**
1. Cek apakah user sudah login (`Auth::check()`)
2. Cek apakah role user ada di daftar role yang diizinkan
3. Kalau tidak → tampilkan error 403 (Forbidden)
4. Kalau iya → lanjutkan request ke controller

**Didaftarkan di `bootstrap/app.php`:**
```php
$middleware->alias([
    'role' => \App\Http\Middleware\RoleMiddleware::class,
]);
```
- Alias `role` supaya bisa dipakai di routes dengan cara `->middleware('role:admin')`

---

## 4. Routes (`routes/web.php`)

Routes adalah daftar URL yang tersedia di aplikasi beserta controller yang menanganinya.

### Konsep Route Group
```php
Route::middleware(['auth', 'role:user'])->prefix('user')->name('user.')->group(function () {
    Route::get('/dashboard', [UserController::class, 'dashboard'])->name('dashboard');
    ...
});
```
- `middleware(['auth', 'role:user'])` → hanya user yang sudah login dan punya role `user` yang bisa akses
- `prefix('user')` → semua URL di grup ini diawali `/user/...`
- `name('user.')` → semua nama route diawali `user.`, misal `user.dashboard`, `user.booking`

### Jenis HTTP Method yang dipakai:
| Method | Fungsi |
|--------|--------|
| `GET` | Menampilkan halaman |
| `POST` | Mengirim/menyimpan data baru |
| `PUT` | Mengupdate data yang sudah ada |
| `DELETE` | Menghapus data |

### Contoh alur URL:
```
GET  /user/booking/buat  → tampilkan form booking
POST /user/booking       → simpan data booking ke database
GET  /user/booking/1     → tampilkan detail booking id=1
POST /user/booking/1/batal → batalkan booking id=1
```

---

## 5. Controllers

Controller adalah tempat logika utama aplikasi. Menerima request dari user, proses data, lalu kirim response (tampilkan view atau redirect).

### `AuthController.php`

**`login()`**
```php
if (Auth::attempt($request->only('email', 'password'))) {
    $request->session()->regenerate();
    return redirect()->route('dashboard');
}
```
- `Auth::attempt()` → Laravel cek email & password ke database secara otomatis
- Kalau cocok → buat session login, redirect ke dashboard
- `session()->regenerate()` → keamanan: buat session ID baru supaya tidak bisa dibajak

**`dashboard()`**
```php
if ($user->isAdmin())   return redirect()->route('admin.dashboard');
if ($user->isMekanik()) return redirect()->route('mekanik.dashboard');
return redirect()->route('user.dashboard');
```
- Setelah login, user diarahkan ke dashboard sesuai rolenya masing-masing

**`register()`**
```php
$request->validate([
    'email' => 'required|email|unique:users',
    'password' => 'required|min:6|confirmed',
]);
```
- `unique:users` → email tidak boleh sudah ada di tabel users
- `confirmed` → password harus sama dengan field `password_confirmation`
- `Hash::make($request->password)` → password di-enkripsi sebelum disimpan, tidak disimpan plain text

### `UserController.php`

**`bookingStore()`**
```php
Booking::create([
    'nomor_antrian' => Booking::generateNomorAntrian(),
    'plat_nomor'    => strtoupper($request->plat_nomor),
    'status'        => 'menunggu',
    'total_harga'   => $service->harga,
]);
```
- Nomor antrian dibuat otomatis
- Plat nomor diubah ke huruf kapital otomatis (`strtoupper`)
- Status awal selalu `menunggu`
- Harga diambil dari data service yang dipilih

**`bookingShow()`**
```php
abort_if($booking->user_id !== Auth::id(), 403);
```
- Keamanan: user hanya bisa lihat booking miliknya sendiri
- Kalau coba akses booking orang lain → error 403

### `MekanikController.php`

**`updateStatus()`**
```php
if ($request->status === 'selesai') {
    Transaksi::create([
        'nomor_transaksi' => Transaksi::generateNomor(),
        'booking_id'      => $booking->id,
        'total'           => $booking->total_harga,
        'status_bayar'    => 'belum_bayar',
    ]);
}
```
- Ketika mekanik tandai `selesai`, sistem **otomatis membuat data transaksi**
- Transaksi awalnya `belum_bayar`, nanti admin yang konfirmasi pembayaran

### `AdminController.php`

**`dashboard()`**
```php
$stats = [
    'total_booking'   => Booking::count(),
    'pendapatan'      => Transaksi::where('status_bayar', 'lunas')->sum('total'),
];
```
- Mengambil statistik ringkasan untuk ditampilkan di dashboard
- `sum('total')` → menjumlahkan semua transaksi yang sudah lunas = total pendapatan

**`bookingAssign()`**
```php
$booking->update(['mekanik_id' => $request->mekanik_id, 'status' => 'diproses']);
```
- Admin pilih mekanik → booking langsung berubah status jadi `diproses`

---

## 6. Seeder (`database/seeders/DatabaseSeeder.php`)

Seeder adalah script untuk mengisi database dengan data awal / contoh.

```php
User::create([
    'email'    => 'admin@bengkel.com',
    'password' => Hash::make('password'),
    'role'     => 'admin',
]);
```

Data yang dibuat otomatis saat `php artisan migrate:fresh --seed`:
- 1 akun Admin
- 2 akun Mekanik (Budi & Andi)
- 1 akun User / Pelanggan demo
- 6 jenis Service (Ganti Oli, Tune Up, dll)
- 5 data Barang (Oli, Busi, Filter, dll)

---

## 7. Views (Tampilan)

Views menggunakan **Blade** — template engine bawaan Laravel.

### Layout Utama (`resources/views/layouts/app.blade.php`)
Semua halaman setelah login menggunakan layout ini. Berisi:
- Navbar atas (nama user, badge role, tombol keluar)
- Sidebar kiri (menu navigasi sesuai role)
- Area konten utama (`{{ $slot }}`)
- CSS inline bergaya sekolah (biru tua + kuning)

Cara pakai di halaman lain:
```blade
<x-layouts.app title="Judul Halaman">
    <!-- konten halaman di sini -->
</x-layouts.app>
```

### Blade Syntax yang dipakai:
```blade
{{ $variable }}          ← tampilkan variabel (auto-escape XSS)
@if / @else / @endif     ← kondisi
@foreach / @endforeach   ← perulangan
@csrf                    ← token keamanan form (wajib di setiap form POST)
@method('PUT')           ← override method HTTP untuk form HTML
{{ route('nama.route') }} ← generate URL dari nama route
```

---

## 8. Alur Kerja Lengkap (Flow)

```
1. PELANGGAN
   Daftar/Login
       ↓
   Pilih jenis service → isi form booking → submit
       ↓
   Sistem buat nomor antrian otomatis (ANT-20260424-001)
   Status booking: MENUNGGU
       ↓
2. ADMIN
   Lihat daftar booking yang menunggu
       ↓
   Pilih mekanik → klik "Tugaskan"
   Status booking: DIPROSES
       ↓
3. MEKANIK
   Lihat daftar service yang ditugaskan
       ↓
   Kerjakan → update status + isi catatan
   Status booking: SELESAI
       ↓
   Sistem otomatis buat data TRANSAKSI (belum bayar)
       ↓
4. ADMIN
   Lihat daftar transaksi
       ↓
   Pilih metode bayar → klik "Konfirmasi Lunas"
   Status transaksi: LUNAS → masuk ke total pendapatan
```

---

## 9. Konsep Laravel yang Digunakan

| Konsep | Penjelasan Singkat |
|--------|-------------------|
| **MVC** | Model-View-Controller, pemisahan logika data, tampilan, dan kontrol |
| **Eloquent ORM** | Cara Laravel berinteraksi dengan database pakai kelas PHP, bukan SQL mentah |
| **Blade Template** | Template engine untuk tampilan HTML dengan sintaks `{{ }}` dan `@directive` |
| **Middleware** | Filter/penjaga yang berjalan sebelum request sampai ke controller |
| **Route Group** | Mengelompokkan routes yang punya middleware/prefix yang sama |
| **Validation** | Validasi input form bawaan Laravel sebelum data disimpan |
| **Auth Facade** | Sistem login/logout/session bawaan Laravel |
| **Relationship** | Relasi antar tabel: `hasMany`, `belongsTo`, `hasOne` |
| **Migration** | Versi kontrol struktur database, bisa dijalankan ulang kapan saja |
| **Seeder** | Mengisi database dengan data awal secara otomatis |

---

## 10. Akun untuk Testing

| Role | Email | Password | Keterangan |
|------|-------|----------|------------|
| Admin | admin@bengkel.com | password | Dibuat via seeder, tidak bisa daftar sendiri |
| Mekanik | budi@bengkel.com | password | Dibuat via seeder, tidak bisa daftar sendiri |
| Mekanik | andi@bengkel.com | password | Dibuat via seeder, tidak bisa daftar sendiri |
| Mekanik | riko@bengkel.com | password | Dibuat via seeder, tidak bisa daftar sendiri |
| User | user@bengkel.com | password | Demo, pelanggan bisa daftar sendiri di /register |

> Admin dan Mekanik tidak bisa daftar lewat halaman register. Akun mereka hanya bisa dibuat lewat `DatabaseSeeder.php` atau ditambahkan manual oleh admin lewat menu Kelola Mekanik.

---

*Dibuat dengan Laravel 13 + MySQL + Blade Template*
