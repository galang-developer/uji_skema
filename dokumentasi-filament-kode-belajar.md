```md
# UJI SKEMA

# Laravel + Filament Setup Guide

Dokumentasi setup project Laravel dengan Filament, role-based access (admin & user), relasi User–Role–Profile–Info, serta custom info di halaman login.
```

## 📌 1. Persiapan Awal

### Buat Project Laravel

```bash
composer create-project laravel/laravel nama-folder
```

Masuk ke folder project:

```bash
cd nama-folder
```

### Install Filament

```bash
composer require filament/filament
```

## 📦 2. Model & Migrasi

### Buat Model & Migration

```bash
php artisan make:model Role -m
php artisan make:model Profile -m
php artisan make:model Info -m
```

Urutan Migration (WAJIB URUT)

1. `roles`
2. `users`
3. `profiles`
4. `infos`

Cara mengurutkannya itu setelah mmembuat model `Role` kalian copy semua tanggal yang ada di migration `roles` contohnya
`2026_02_04_121141_create_roles_table.php`
kalian ambil bagian `2026_02_04_121141_` lalu kalian ubah tanggal pada migration `users` lalu paste dibagian tanggalnya saja jadi bagian `_create_users_table.php` tetap sama lalu kalian ubah akhiran nomor dari yang kalian copy tadi contoh disitu akhirannya `41` maka kalian ubah paling akhir dengan cara mengganti jadi `42` (tambah)

## 🔗 3. Relasi Model

### User.php → `profile()`, `role()` & `setPasswordAttribute`

```php
// Relasi ke Profile
public function profile() {
    return $this->hasOne(Profile::class, 'id_user', 'id');
}

// Relasi ke Role
public function role() {
    return $this->belongsTo(Role::class, 'id_role', 'id');
}
```

### Profile.php → `user()`

```php
// Relasi ke User
public function user() {
    return $this->belongsTo(User::class, 'id_user', 'id');
}
```

### Info.php → `user()`, `booted()` + auto `dibuat_oleh`

```php
// Relasi pembuat
public function user() {
    return $this->belongsTo(User::class, 'dibuat_oleh');
}

// Auto-set pembuat saat dibuat
protected static function booted() {
    static::creating(function ($info) {
        if (!$info->dibuat_oleh) {
            $info->dibuat_oleh = auth()->id();
        }
    });
}
```

## 🗃️ 4. Struktur Tabel (Migration)

### Roles Table

```php
$table->string('role_name');
```

### Users Table (+ foreign key)

```php
$table->foreignId('id_role')
    ->default(2) // default role 'user'
    ->constrained('roles')
    ->cascadeOnDelete()
    ->cascadeOnUpdate();
$table->enum('status', ['OK','NO']);
```

### Profiles Table

```php
$table->foreignId('id_user')
    ->constrained('users')
    ->cascadeOnDelete();
$table->string('image')->nullable();
$table->string('bio')->nullable();
```

### Infos Table

```php
$table->string('title');
$table->string('teks');
$table->foreignId('dibuat_oleh')
    ->constrained('users')
    ->cascadeOnDelete();
```

## 🌱 5. Seeder & Migrasi

### Buat Seeder

```bash
php artisan make:seed RoleSeeder
```

### `RoleSeeder.php` (admin & user)

```php
// Import modulenya jangan lupa
// Double klik terus ctrl + space kalo mau import lewat auto complete
use Illuminate\Support\Facades\DB;

DB::table('roles')->insert([
    ['id' => 1, 'role_name' => 'admin'],
    ['id' => 2, 'role_name' => 'user'],
]);
```

### `DatabaseSeeder.php`

```php
$this->call(RoleSeeder::class);
```

### Jalankan Migrasi + Seeder

```bash
php artisan migrate --seed
```

## 🚀 6. Setup Filament Panel

### Install Panel

```bash
php artisan filament:install --panels
```

Jika muncul pertanyaan:

```bash
What is the panel's ID? [admin]
```

Isi saja `auth`

Berfungsi untuk membuat path uri/url yang dimana nanti urlnya itu seperti ini `http://127.0.0.1:8000/auth/login`

### Aktifkan Registrasi & Login

Edit File `app/Providers/Filament/AuthPanelProvider.php`:

Tambahkan:

```php
->registration() // Aktifkan halaman registrasi
->login() // Aktifkan halaman login (defaultnya sudah ada)
->spa() // Single-Page Application, fungsi lengkapnya bisa cari di google (opsional)
```

### Buat Pengguna (Admin User)

```php
php artisan make:filament-user
```

Isi saja sesuai apa yang diminta seperti `Name`, `Email`, `Password`

Karena default id_role nya `2` pada table `users` jadi kalian bisa ubah di phpMyAdmin pada table `users` klik dua kali pada `id_role` dan pilih `admin` atau `1`

## 🔧 7. Resource & Authorization

### Buat Resource CRUD

```bash
php artisan make:filament-resource User --generate --view
php artisan make:filament-resource Profile --generate --view
php artisan make:filament-resource Info --generate --view
```

Jika muncul pertanyaan:

```bash
What is the title attribute for this model?
```

Isi saja `nama model`

Pastikan `sebelum membuat resource` itu di modelnya sudah ada isinya, isi yang di maksud:

```php
protected $fillable = [
    ...
];
```

### Atur Hak Akses (Authorization)

### UserResource.php → Hanya admin bisa buat/edit:

```php
// Import modulenya jangan lupa
// Double klik terus ctrl + space kalo mau import lewat auto complete
use Illuminate\Support\Facades\Auth;

// Kata kunci: canAccess, canCreate, canEdit, canViewAny
public static function canCreate(): bool {
    return Auth::user()->role?->role_name === 'admin';
}
// Sama untuk canEdit dan can lainnya
```

### ProfileResource.php → Auto redirect ke edit profile user:

```php
// Kata kunci: canAccess, canCreate, canEdit, canViewAny, getNavigationUrl
public static function canCreate(): bool {
    return Auth::user()->role?->role_name === 'admin';
}

// Opsional
public static function getNavigationUrl(): string {
    $user = Auth::user();
    $profile = $user->profile ?? $user->profile()->create(['id_user' => $user->id]);
    return static::getUrl('edit', ['record' => $profile]);
}
// Sama untuk canEdit dan can lainnya
```

### InfoResource.php → Hanya admin bisa buat/edit:

```php
// Kata kunci: canAccess, canCreate, canEdit, canViewAny
public static function canCreate(): bool {
    return Auth::user()->role?->role_name === 'admin';
}
```

## 🎨 8. Custom Form & Table

### UserForm.php → Select role dengan default

```php
Select::make('id_role')
    ->relationship('role', 'role_name')
    ->default(2) // default 'user'
    ->required()
```

### ProfileForm.php → Sembunyikan id_user

```php
TextInput::make('id_user')->hidden()
```

### ListProfiles.php → Auto redirect ke EditProfile.php

```php
public function mount(): void
{
    parent::mount();

    $user = Auth::user();
    $profile = $user->profile ?? $user->profile()->create([
        'id_user' => $user->id,
    ]);

    $this->redirect(
        ProfileResource::getUrl('edit', ['record' => $profile])
    );
}
```

Kode diatas bisa kalian ketik setelah kode:

```php
protected static string $resource = ProfileResource::class;
```

### UsersTable.php → Tampilkan role name

```php
TextColumn::make('role.role_name')->label('Role')
```

## 📢 9. Info Panel di Halaman Login

### Tambahkan Hook di `AuthPanelProvider.php`

```php
// Kata kunci: renderHook, SIMPLE_LAYOUT_START
->renderHook(
    PanelsRenderHook::SIMPLE_LAYOUT_START,
    fn () => view('info-panel')
)
```

### Buat View `info-panel.blade.php`

```html
@php
    use App\Models\Info;
    $infos = Info::latest()->get(['title', 'teks']);
@endphp

<style>
    .notif {
        position: fixed;
        top: 64px;
        left: 50%;
        transform: translateX(-50%);
        background: #ffbb00;
        color: #fff;
        padding: 12px 20px;
        border-radius: 8px;
        z-index: 40;
    }
</style>

<div x-data="infoLoop({{ $infos->toJson() }})" x-init="init()" class="notif">
    <template x-if="c">
        <div>
            <h4 x-text="c.title"></h4>
            <p x-text="c.teks"></p>
        </div>
    </template>
</div>

<script>
    function infoLoop(items) {
        return {
            c: items[0],
            init() {
                setInterval(() => this.c = items[Math.random() * items.length | 0], 2000)
            }
        }
    }
</script>
```

## ✅ Fitur Siap Pakai
1. `Role-based access` → Admin vs User
2. `Auto-profile` → Otomatis dibuat saat user registrasi
3. `CRUD lengkap` → User, Profile, Info
4. `Dynamic info panel` → Rotasi info di halaman login
5. `Foreign key relations` → Cascade update/delete

## 📌 Catatan:
- Pastikan urutan migrasi sesuai
- Jalankan seeder untuk role default
- Tes hak akses dengan login admin/user berbeda







