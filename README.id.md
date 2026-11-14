# hop/auth

[English](README.md) · [简体中文](README.zh-CN.md) · [Deutsch](README.de.md) · **Bahasa Indonesia** · [日本語](README.ja.md) · [Português (Brasil)](README.pt-BR.md) · [Español](README.es.md) · [Français](README.fr.md) · [Русский](README.ru.md)

Klien Laravel untuk HOP ID, layanan akun keluarga HOP ([hel.ink](https://hel.ink)). Dengan paket ini, aplikasi dalam keluarga bisa memasukkan pengguna lewat hub di `hub.hel.ink`, memeriksa tiket masuk yang dikirim kembali oleh hub, menerima token API dengan scope, dan mengeluarkan pengguna lagi.

Hub sendiri hanya memakai model `ApiToken` dan alias middleware. Rute SSO ditujukan untuk aplikasi yang memakai HOP ID, seperti LinkHop dan BioHop.

## Isi paket

- Tiga rute untuk alur masuk: start, callback, dan logout.
- `Hop\Auth\Ticket`, yang memverifikasi tiket bertanda tangan HMAC-SHA256. Kelas ini memeriksa tanda tangan, audiens (aplikasi ini), masa berlaku, dan `state` yang dikirim aplikasi. Selama rotasi, secret sebelumnya juga diterima.
- Middleware `hop.auth`: seperti `auth` milik Laravel, tetapi tamu diarahkan lewat hub, bukan ke halaman login lokal.
- Middleware `hop.token`: token API bearer dari tabel bersama `api_tokens`, dengan scope, masa berlaku, dan batas laju per jam untuk setiap token.
- Logout berantai yang hanya mengikuti tautan ke host tepercaya.
- Penerima event akun dari hub (`POST /hop/events`): pemeriksaan tanda tangan, id event yang hanya berlaku sekali, dan hook untuk aplikasi. Keluar di hub, admin yang mengakhiri sesi, penangguhan, atau penghapusan sampai ke aplikasi lewat jalur ini, meskipun aplikasi sedang mati saat kejadian itu terjadi.

## Persyaratan

- PHP 8.3 dan Laravel 12 (`illuminate/support` ^12.0).
- Tabel bersama `users` milik keluarga. Paket ini memasukkan pengguna yang id-nya sama dengan `sub` pada tiket.
- Untuk token API, tabel bersama `api_tokens`. Paket ini tidak menyertakan migrasi untuk tabel itu; token dibuat dan dicabut di hub.
- Penyimpanan sesi dan penyimpanan cache. Cache menyimpan penanda sekali pakai setiap tiket dan penghitung batas laju.

## Instalasi

Paket ini tidak ada di Packagist. Tambahkan repositori ke `composer.json` aplikasi:

```json
"repositories": [
    {
        "type": "vcs",
        "url": "https://github.com/navi-crwn/hop-auth.git"
    }
],
"require": {
    "hop/auth": "^0.2.0"
}
```

Lalu jalankan:

```bash
composer update hop/auth
```

Package discovery Laravel mendaftarkan `Hop\Auth\AuthServiceProvider`. Provider ini tidak memuat rute SSO jika `config('hop.current')` bernilai `hub`: hub menerbitkan tiket dan tidak pernah memakainya.

## Konfigurasi

Atur nilai-nilai ini di `.env` aplikasi. Biasanya hanya `HOP_APP` dan `HOP_SSO_SECRET` yang diperlukan:

```ini
HOP_APP=linkhop
HOP_HUB_URL=https://hub.hel.ink
HOP_SSO_SECRET=at-least-32-characters-shared-with-the-hub
# HOP_SSO_SECRET_PREV=only-while-rotating
# HOP_USER_MODEL=App\Models\User
# HOP_TRUSTED_HOSTS=only-for-a-self-hosted-family
```

| Variabel | Bawaan | Arti |
| --- | --- | --- |
| `HOP_APP` | `linkhop` | Slug aplikasi ini di `config/hop.php` milik hub. Nilai ini juga menjadi audiens tiket. |
| `HOP_HUB_URL` | `https://hub.hel.ink` | Hub yang menerbitkan tiket. |
| `HOP_SSO_SECRET` | tidak ada | Secret HMAC bersama. Secret yang lebih pendek dari 32 karakter diabaikan, dan tanpa secret yang bisa dipakai, setiap tiket ditolak sebagai `not_configured`. |
| `HOP_SSO_SECRET_PREV` | tidak ada | Secret sebelumnya, hanya diterima selama Anda melakukan rotasi. |
| `HOP_USER_MODEL` | `App\Models\User` | Model yang dimasukkan, sekaligus pemilik token API. |
| `HOP_TRUSTED_HOSTS` | host keluarga produksi | Host yang boleh menjadi tujuan pengalihan logout berantai. Lihat di bawah. |
| `HOP_RECHECK` | `true` | Pengecekan ulang akun secara senyap saat halaman dimuat (`false` mematikannya). Lihat di bawah. |
| `HOP_RECHECK_MAX_AGE` | `300` | Detik sebuah sesi dianggap segar sejak hub terakhir mengesahkannya. `0` mematikan pengecekan ulang. |

Opsi lainnya tidak punya variabel lingkungan. Terbitkan file konfigurasi untuk mengubahnya:

```bash
php artisan vendor:publish --tag=hop-auth-config
```

| Kunci | Bawaan | Arti |
| --- | --- | --- |
| `leeway` | `30` | Selisih jam yang ditoleransi, dalam detik, saat memeriksa `exp` dan `iat` pada tiket. |
| `home` | `/dashboard` | Tujuan setelah masuk jika aplikasi tidak meminta halaman tertentu. |
| `api_rate_limit` | `100` | Jumlah permintaan per jam untuk token API yang tidak punya batas sendiri. |
| `events.hub_app`, `events.max_age`, `events.seen_ttl` | `helink`, `120`, `172800` | Event akun: slug aplikasi yang dipakai hub untuk menandatangani, selisih jam yang ditoleransi dalam detik (setiap nonce berlaku sekali dalam dua kali rentang ini), dan berapa lama id event diingat. |
| `routes.events` | `null` | Path penerima event akun. Mati secara bawaan; lihat di bawah. |
| `routes.start`, `routes.callback`, `routes.logout` | `/auth/sso`, `/auth/sso/callback`, `/auth/sso/logout` | Path rute. Atur `routes.start` ke `null` untuk melewati rute paket dan mendaftarkan rute Anda sendiri. |

## Cara proses masuk berjalan

1. Seorang tamu membuka halaman di balik `hop.auth`. Middleware mengarahkannya ke `/auth/sso?next=<page>`. Permintaan JSON mendapat `401` sebagai gantinya.
2. `start` menyimpan `state` acak dan halaman `next` di sesi, lalu mengalihkan ke `HOP_HUB_URL/sso/{HOP_APP}?state=…&return_to=<callback url>`. Hanya path dari situs yang sama yang disimpan sebagai `next`.
3. Hub memasukkan pengguna dan mengalihkan ke `/auth/sso/callback?ticket=…`.
4. `callback` memverifikasi tiket dan state. Tiket yang sudah pernah dipakai ditolak (`jti` diingat selama 5 menit), begitu juga akun yang ditangguhkan (`status` bernilai `suspended`).
5. Pengguna dengan id `sub` dimasukkan, sesi dibuat ulang, payload tiket disimpan di sesi sebagai `hop_profile`, lalu pengguna dialihkan ke `next` atau ke `home`.

Tiket adalah `base64url(json)` dan `base64url(HMAC-SHA256)` yang digabung dengan titik. JSON wajib berisi `sub`, `exp`, dan `state`. `aud`, `iat`, `jti`, dan `status` bersifat opsional. `Ticket::verify()` melempar `Hop\Auth\TicketException` dengan salah satu alasan berikut: `not_configured`, `malformed`, `bad_signature`, `expired`, `wrong_audience`, `state_mismatch`.

### Rotasi secret

Taruh secret baru di `HOP_SSO_SECRET` dan secret lama di `HOP_SSO_SECRET_PREV`. Tiket yang ditandatangani dengan salah satunya akan diterima. Hapus nilai lama setelah hub hanya menandatangani dengan secret baru.

## Melindungi rute

```php
Route::middleware('hop.auth')->group(function () {
    Route::get('/dashboard', DashboardController::class);
});

Route::middleware('hop.token:link:create')->post('/api/links', StoreLinkController::class);
```

### Token API

Klien mengirim `Authorization: Bearer hlk_…`. Paket ini hanya menyimpan hash SHA-256 dari token. `hop.token` menjawab dengan JSON berbentuk `{"success": false, "error": "…", "message": "…"}` dan status berikut:

| Status | Penyebab |
| --- | --- |
| 401 | Header tidak ada atau rusak, token tidak dikenal, atau token kedaluwarsa. |
| 403 | Akun ditangguhkan, atau token tidak punya scope yang dibutuhkan. |
| 429 | Batas per jam token sudah habis. |

Jika berhasil, permintaan membawa atribut `api_user` dan `api_token`, dan `$request->user()` mengembalikan pemilik token.

`hop.token` tanpa scope hanya memeriksa bahwa token valid. `hop.token:link:create` juga membutuhkan scope `link:create`. Token tanpa scope (null atau daftar kosong) boleh melakukan apa saja. Selain itu, `can()` menerima kecocokan persis, wildcard awalan seperti `link:*`, atau `*`. Scope yang ditetapkan keluarga ada di `ApiToken::SCOPES`: `link:read`, `link:create`, `link:write`, `bio:read`, `bio:write`, `pixel:read`, dan `pixel:write`. Hub HOP hanya menawarkan scope `link:*` saat membuat token baru (token hub untuk LinkHop) dan menolak token baru yang membawa scope `pixel:*`, karena PixelHop menerbitkan kuncinya sendiri. Scope `bio:*` ada di daftar, tetapi tidak ditawarkan di formulir. Token yang dibuat sebelumnya dengan scope `pixel:*` tetap sah, tetapi scope itu tidak berarti apa-apa lagi: PixelHop hanya menerima kunci miliknya sendiri (`ph_`), bukan token hub.

### Batas laju token API

Setiap token boleh membuat `rate_limit` permintaan per jam. Token yang `rate_limit`-nya kosong atau tidak lebih dari nol mendapat `hop-auth.api_rate_limit` (100 secara bawaan). Jendela waktu dimulai pada permintaan pertama token dan berakhir satu jam kemudian. Penghitung disimpan di penyimpanan cache rate limiter dan dinaikkan sebelum dibandingkan dengan batas, sehingga permintaan paralel tidak bisa lolos bersamaan.

Permintaan yang ditolak mendapat `429` dengan `retry_after` di body dan header `Retry-After` yang sesuai, keduanya dalam detik sampai jendela berakhir.

## Cek ulang akun senyap

Sesi di aplikasi Anda bisa bertahan lebih lama daripada akun yang dipakai orang itu di hub: ia masuk sebagai akun lain, atau keluar, di hub, dan aplikasi Anda tetap memakai akun lama. `hop.auth` menutup celah itu tanpa migrasi:

- Sesi dianggap **segar** selama `recheck.max_age` detik (bawaan 300) sejak hub terakhir mengesahkannya, lewat proses masuk atau jawaban senyap. Waktunya disimpan di sesi.
- Bila tidak segar, pemuatan halaman (sebuah `GET` yang bukan XHR, bukan JSON, dan bukan prefetch) melakukan satu perjalanan `prompt=none` ke hub ([SSO senyap](https://github.com/navi-crwn/HEL.ink/blob/main/docs/silent-sso.md)). Akun sama: halaman dimuat seolah tidak terjadi apa-apa. Akun lain: sesi berganti ke akun itu. `login_required`: orang itu dikeluarkan dari sini dan melihat halaman sebagai tamu. Halaman terlindung lalu memulai proses masuk biasa; halaman publik tidak menampilkan proses masuk sama sekali.
- Pengaman: permintaan POST atau XHR tidak pernah dialihkan. Selama hub tidak menjawab URL kesehatannya (`/up`) dengan 2xx, misalnya 502 atau waktu habis, tidak ada pengalihan dan tidak ada yang dikeluarkan: sesi tetap dipercaya, dan pemuatan halaman berikutnya setelah hasil cache kedaluwarsa (60 detik) mencoba lagi. Paling banyak satu percobaan per `recheck.cooldown` detik (30), jadi tidak ada loop. Galat hub lain, misalnya `temporarily_unavailable`, mempertahankan sesi dan menunggu `recheck.retry_after` detik (300). `state` harus sama dengan yang dimulai sesi.
- Halaman yang tidak memerlukan login tetapi menampilkan akun (halaman depan dengan tombol "Dasbor") diberi `hop.recheck` di middleware-nya agar ikut diperiksa.
- Kuncinya ada di `hop-auth.recheck` (`enabled`, `max_age`, `cooldown`, `retry_after`, `probe`, `probe_path`, `probe_timeout`, `probe_ttl`). `HOP_RECHECK=false` mematikan seluruh pengecekan.

## Logout berantai dan host tepercaya

`/auth/sso/logout` mengeluarkan pengguna di sini, membatalkan sesi, dan membuat ulang token CSRF. Jika `?next=` berisi URL tepercaya, pengguna lalu dialihkan ke sana. Dengan cara inilah hub mengeluarkan pengguna dari setiap aplikasi. Selain itu, pengguna diarahkan ke `/`.

URL `next` dianggap tepercaya jika semua syarat ini terpenuhi:

- Host-nya persis salah satu host tepercaya. Tidak ada pencocokan wildcard: `evil.hel.ink` tidak tepercaya hanya karena `hel.ink` tepercaya.
- URL tidak berisi info pengguna, spasi atau karakter kontrol, dan backslash.
- URL memakai `https`. `http` biasa hanya diterima untuk `localhost`, `127.0.0.1`, dan `[::1]`, dan hanya jika host itu ada di daftar tepercaya.

Host tepercaya adalah `hop-auth.trusted_hosts` ditambah hub (`HOP_HUB_URL`), aplikasi ini (`APP_URL`), dan URL di `config/hop.php` bersama (`hop.public_url`, `hop.hub_url`, dan `url` setiap aplikasi).

Daftar bawaan adalah keluarga produksi: `hel.ink`, `www.hel.ink`, `hub.hel.ink`, `link.hel.ink`, `bio.hel.ink`, `note.hel.ink`, `pixel.hel.ink`, `file.hel.ink`, dan `pay.hel.ink`. `HOP_TRUSTED_HOSTS` menggantikan daftar itu. Jika Anda menjalankan keluarga ini di domain sendiri, atur host aplikasi Anda:

```ini
HOP_TRUSTED_HOSTS=example.org,hub.example.org,link.example.org,bio.example.org
```

## Event akun dari hub

Hub memberi tahu setiap aplikasi yang menyatakan kapabilitas `events` saat sesuatu terjadi pada sebuah akun, dengan `POST {app}/hop/events` dan body `{id, type, at, data}`. Hub menandatangani permintaan dengan secret aplikasi (HOP App Protocol v1: header `X-HOP-App: helink`, `X-HOP-Timestamp`, `X-HOP-Nonce`, `X-HOP-Signature`), dan mencoba ulang dengan backoff selama sekitar enam belas jam jika jawabannya bukan 2xx. Jenis-jenisnya:

| Jenis | Dikirim saat | Bawaan di sini |
| --- | --- | --- |
| `session.revoked` | pengguna keluar di hub, atau admin mengakhiri sesinya | Mengakhiri setiap sesi akun itu di sini dan merotasi `remember_token`-nya. |
| `account.deleted` | akun dihapus | Mengakhiri setiap sesi akun itu di sini. Hapus data milik aplikasi dengan handler Anda sendiri. |
| `account.status` | status berubah (`data.status`: `active`, `suspended`) | Tidak ada. LinkHop dan BioHop sudah mengeluarkan akun yang ditangguhkan pada permintaan berikutnya; tambahkan handler jika aplikasi Anda butuh lebih. |
| `profile.updated` | profil berubah | Tidak ada. |

`data.sub` adalah id akun di tabel bersama `users`. Jenis lain mendapat 200 `{handled: false}`, sehingga hub tidak pernah mencoba ulang hal yang tidak dikenal aplikasi ini.

Mengapa remember token penting: proses masuk dari paket ini memakai `remember: true`, jadi menghapus baris sesi seseorang saja tidak cukup. Cookie remember (400 hari secara bawaan) akan langsung memasukkannya kembali. `SessionRevoker` menghapus sesi (driver `database` dan `file`; driver lain tidak bisa dicari per akun, dan `complete` bernilai `false` di hasilnya) dan selalu merotasi token.

Dua cara menerimanya:

1. Atur `routes.events` ke `/hop/events`. Paket mendaftarkan rute di balik `hop.event`, yang memeriksa tanda tangan dengan `HOP_SSO_SECRET` dan `HOP_SSO_SECRET_PREV` yang sama seperti tiket masuk, menerima timestamp dalam rentang `events.max_age` detik, dan memakai setiap nonce hanya sekali. Setiap penolakan berupa `401 {"ok":false,"error":"unauthorized"}` polos yang sama.
2. Jika aplikasi sudah punya rute dengan pemeriksaan tanda tangan sendiri, arahkan rute itu ke `Hop\Auth\Http\Controllers\EventController` (controller invokable biasa). Controller ini memeriksa amplop, mengingat setiap id event di cache (percobaan ulang dijawab `{duplicate: true}` dan tidak diproses dua kali), lalu menjalankan handler. Handler yang melempar exception menghasilkan 500 tanpa detail dan id-nya dilupakan, sehingga percobaan ulang dari hub menjalankannya lagi.

Tambahkan handler Anda sendiri di service provider:

```php
use Hop\Auth\HopEvents;

HopEvents::on('account.deleted', fn (array $data, array $event) => $this->purge((int) $data['sub']));
HopEvents::on('account.status', function (array $data) {
    // return false when the event is not about anyone you know
});
HopEvents::forget('session.revoked'); // drop the built-in handler
```

Id event membutuhkan penyimpanan cache yang bertahan antarpermintaan (`database`, `file`, `redis`), bukan `array`. Hub harus punya baris `apps` untuk aplikasi itu dengan kapabilitas `events`, atau hub tidak mengirim apa pun.

## Tes

Tes di `tests/Feature` adalah tes Pest yang membutuhkan aplikasi Laravel dengan tabel bersama `users`, jadi tes ini berjalan di dalam aplikasi yang memakai paket ini (HEL.ink menyertakan salinannya di `packages/auth`). CI repositori ini sendiri memeriksa bahwa `composer.json` valid, bahwa setiap file PHP lolos lint di PHP 8.3, dan bahwa tabel keluarga di bawah sama dengan tabel kanonis.

## Keamanan

Laporkan kerentanan sesuai petunjuk di [SECURITY.md](SECURITY.md), bukan lewat issue publik.

## Catatan perubahan

Lihat [CHANGELOG.md](CHANGELOG.md).

## Hosting sendiri

Anda tidak perlu memasang paket ini secara manual untuk menjalankan HOP. LinkHop dan BioHop mencantumkan repositori publik ini di `composer.json` mereka, jadi `composer install` mengambilnya tanpa token, dan hub ([HEL.ink](https://github.com/navi-crwn/HEL.ink)) membawa salinannya di `packages/auth`. Aplikasi yang memakainya membutuhkan hub yang berjalan di database yang sama dan pengaturan berikut: `HOP_HUB_URL`, `HOP_SSO_SECRET` (milik hub), `HOP_APP`, dan di domain Anda sendiri `HOP_TRUSTED_HOSTS`, karena daftar bawaan hanya berisi host hel.ink. Imperium, panel admin, bersifat opsional. Panduan keluarga membahas sisanya: [docs/self-hosting.md](https://github.com/navi-crwn/HEL.ink/blob/main/docs/self-hosting.md).

## Keluarga HOP

<!-- hop-family:start -->
<!-- Generated from HEL.ink resources/hop-assets/apps.source.json; edit there, then run scripts/hop_assets.py build. -->
| App | Address | Repository | What it does |
| --- | --- | --- | --- |
| **HOP ID** | [hel.ink](https://hel.ink) · accounts at `hub.hel.ink` | [navi-crwn/HEL.ink](https://github.com/navi-crwn/HEL.ink) | One account for every HOP app. |
| **LinkHop** | [link.hel.ink](https://link.hel.ink) · `hel.ink/l/…` | [navi-crwn/LinkHop](https://github.com/navi-crwn/LinkHop) | Short links, QR codes and click analytics. |
| **BioHop** | [bio.hel.ink](https://bio.hel.ink) · `hel.ink/b/…` | [navi-crwn/BioHop](https://github.com/navi-crwn/BioHop) | One link-in-bio page for everything you do. |
| **PixelHop** | [pixel.hel.ink](https://pixel.hel.ink) · `p.hel.ink/…` | [navi-crwn/PixelHop](https://github.com/navi-crwn/PixelHop) | Image hosting with albums and direct links. |
| **NoteHop** | [note.hel.ink](https://note.hel.ink) · `n.hel.ink/…` | [navi-crwn/NoteHop](https://github.com/navi-crwn/NoteHop) | Encrypted notes you share with a link. |
| **FileHop** | [file.hel.ink](https://file.hel.ink) | [navi-crwn/FileHop](https://github.com/navi-crwn/FileHop) | Convert files in your browser; nothing is uploaded. |
| **PayHop** | [pay.hel.ink](https://pay.hel.ink) | [navi-crwn/PayHop](https://github.com/navi-crwn/PayHop) | Make a payment QR and split a bill; nothing leaves your browser. |
| **Imperium** | not public-facing: optional, for admins | [navi-crwn/Imperium](https://github.com/navi-crwn/Imperium) | The HOP control panel. |
<!-- hop-family:end -->

## Lisensi

Paket bersama HOP adalah perangkat lunak open-source berlisensi [MIT](LICENSE).
