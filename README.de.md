# hop/auth

[English](README.md) · [简体中文](README.zh-CN.md) · **Deutsch** · [Bahasa Indonesia](README.id.md) · [日本語](README.ja.md) · [Português (Brasil)](README.pt-BR.md) · [Español](README.es.md) · [Français](README.fr.md) · [Русский](README.ru.md)

Der Laravel-Client für HOP ID, den Kontodienst der HOP-Familie ([hel.ink](https://hel.ink)). Damit kann eine App der Familie Personen über den Hub unter `hub.hel.ink` anmelden, das Anmeldeticket prüfen, das der Hub zurückschickt, API-Tokens mit Scopes annehmen und Personen wieder abmelden.

Der Hub selbst nutzt nur das Modell `ApiToken` und die Middleware-Aliase. Die SSO-Routen sind für die Apps gedacht, die HOP ID nutzen, etwa LinkHop und BioHop.

## Was das Paket bietet

- Drei Routen für den Anmeldeablauf: Start, Callback und Logout.
- `Hop\Auth\Ticket`, das ein mit HMAC-SHA256 signiertes Ticket prüft. Es prüft die Signatur, die Zielgruppe (diese App), den Ablauf und den `state`, den die App gesendet hat. Während einer Rotation wird auch ein vorheriges Secret akzeptiert.
- Die Middleware `hop.auth`: wie `auth` von Laravel, aber ein Gast wird über den Hub geleitet statt auf eine lokale Login-Seite.
- Die Middleware `hop.token`: Bearer-API-Tokens aus der gemeinsamen Tabelle `api_tokens`, mit Scopes, Ablaufzeit und einem stündlichen Rate Limit pro Token.
- Ein verketteter Logout, der nur Links zu vertrauenswürdigen Hosts folgt.
- Ein Empfänger für die Kontoereignisse des Hubs (`POST /hop/events`): Signaturprüfung, Ereignis-IDs, die nur einmal gelten, und Hooks für die App. Eine Abmeldung am Hub, ein Admin, der Sessions beendet, eine Sperrung oder eine Löschung erreichen die App auf diesem Weg, auch wenn die App zu dem Zeitpunkt nicht lief.

## Voraussetzungen

- PHP 8.3 und Laravel 12 (`illuminate/support` ^12.0).
- Die gemeinsame Tabelle `users` der Familie. Das Paket meldet den Benutzer an, dessen ID dem `sub` des Tickets entspricht.
- Für API-Tokens die gemeinsame Tabelle `api_tokens`. Das Paket liefert dafür keine Migration mit; Tokens werden am Hub erstellt und widerrufen.
- Ein Session-Store und ein Cache-Store. Der Cache hält die Einmal-Markierung jedes Tickets und die Zähler für das Rate Limit.

## Installation

Das Paket ist nicht auf Packagist. Füge das Repository zur `composer.json` der App hinzu:

```json
"repositories": [
    {
        "type": "vcs",
        "url": "https://github.com/navi-crwn/hop-auth.git"
    }
],
"require": {
    "hop/auth": "^0.1.2"
}
```

Dann führe aus:

```bash
composer update hop/auth
```

Die Package Discovery von Laravel registriert `Hop\Auth\AuthServiceProvider`. Der Provider lädt die SSO-Routen nicht, wenn `config('hop.current')` den Wert `hub` hat: Der Hub stellt Tickets aus und nimmt nie welche an.

## Konfiguration

Setze diese Werte in der `.env` der App. Meist reichen `HOP_APP` und `HOP_SSO_SECRET`:

```ini
HOP_APP=linkhop
HOP_HUB_URL=https://hub.hel.ink
HOP_SSO_SECRET=at-least-32-characters-shared-with-the-hub
# HOP_SSO_SECRET_PREV=only-while-rotating
# HOP_USER_MODEL=App\Models\User
# HOP_TRUSTED_HOSTS=only-for-a-self-hosted-family
```

| Variable | Standard | Bedeutung |
| --- | --- | --- |
| `HOP_APP` | `linkhop` | Der Slug dieser App in der `config/hop.php` des Hubs. Er ist auch die Zielgruppe des Tickets. |
| `HOP_HUB_URL` | `https://hub.hel.ink` | Der Hub, der Tickets ausstellt. |
| `HOP_SSO_SECRET` | keiner | Das gemeinsame HMAC-Secret. Secrets mit weniger als 32 Zeichen werden ignoriert, und ohne brauchbares Secret wird jedes Ticket als `not_configured` abgelehnt. |
| `HOP_SSO_SECRET_PREV` | keiner | Das vorherige Secret, nur während der Rotation akzeptiert. |
| `HOP_USER_MODEL` | `App\Models\User` | Das Modell, das angemeldet wird, und der Besitzer eines API-Tokens. |
| `HOP_TRUSTED_HOSTS` | die Hosts der Produktionsfamilie | Hosts, zu denen der verkettete Logout weiterleiten darf. Siehe unten. |
| `HOP_RECHECK` | `true` | Stille Kontoprüfung beim Seitenaufruf (`false` schaltet sie ab). Siehe unten. |
| `HOP_RECHECK_MAX_AGE` | `300` | Sekunden, die eine Sitzung als frisch gilt, nachdem der Hub sie zuletzt bestätigt hat. `0` schaltet die Prüfung ab. |

Die übrigen Optionen haben keine Umgebungsvariable. Veröffentliche die Konfigurationsdatei, um sie zu ändern:

```bash
php artisan vendor:publish --tag=hop-auth-config
```

| Schlüssel | Standard | Bedeutung |
| --- | --- | --- |
| `leeway` | `30` | Tolerierte Uhrabweichung in Sekunden beim Prüfen von `exp` und `iat` eines Tickets. |
| `home` | `/dashboard` | Ziel nach der Anmeldung, wenn die App keine bestimmte Seite angefragt hat. |
| `api_rate_limit` | `100` | Anfragen pro Stunde für ein API-Token ohne eigenes Limit. |
| `events.hub_app`, `events.max_age`, `events.seen_ttl` | `helink`, `120`, `172800` | Kontoereignisse: der App-Slug, mit dem der Hub signiert, die tolerierte Uhrabweichung in Sekunden (jede Nonce gilt einmal innerhalb des Doppelten davon) und wie lange eine Ereignis-ID gespeichert bleibt. |
| `routes.events` | `null` | Pfad des Empfängers für Kontoereignisse. Standardmäßig aus; siehe unten. |
| `routes.start`, `routes.callback`, `routes.logout` | `/auth/sso`, `/auth/sso/callback`, `/auth/sso/logout` | Die Routenpfade. Setze `routes.start` auf `null`, um die Routen des Pakets zu überspringen und eigene zu registrieren. |

## Ablauf einer Anmeldung

1. Ein Gast öffnet eine Seite hinter `hop.auth`. Die Middleware schickt ihn zu `/auth/sso?next=<page>`. Eine JSON-Anfrage bekommt stattdessen eine `401`.
2. `start` speichert einen zufälligen `state` und die `next`-Seite in der Session und leitet dann zu `HOP_HUB_URL/sso/{HOP_APP}?state=…&return_to=<callback url>` weiter. Als `next` wird nur ein Pfad auf derselben Site behalten.
3. Der Hub meldet die Person an und leitet zu `/auth/sso/callback?ticket=…` weiter.
4. `callback` prüft das Ticket und den State. Es lehnt ein bereits benutztes Ticket ab (die `jti` wird 5 Minuten lang gespeichert), ebenso ein gesperrtes Konto (`status` mit dem Wert `suspended`).
5. Es meldet den Benutzer mit der ID `sub` an, erneuert die Session, legt die Nutzdaten des Tickets als `hop_profile` in der Session ab und leitet zu `next` oder zu `home` weiter.

Ein Ticket besteht aus `base64url(json)` und `base64url(HMAC-SHA256)`, verbunden durch einen Punkt. Das JSON braucht `sub`, `exp` und `state`. `aud`, `iat`, `jti` und `status` sind optional. `Ticket::verify()` wirft `Hop\Auth\TicketException` mit einem dieser Gründe: `not_configured`, `malformed`, `bad_signature`, `expired`, `wrong_audience`, `state_mismatch`.

### Secret-Rotation

Trage das neue Secret in `HOP_SSO_SECRET` und das alte in `HOP_SSO_SECRET_PREV` ein. Ein Ticket, das mit einem der beiden signiert ist, wird akzeptiert. Entferne den alten Wert, sobald der Hub nur noch mit dem neuen signiert.

## Routen schützen

```php
Route::middleware('hop.auth')->group(function () {
    Route::get('/dashboard', DashboardController::class);
});

Route::middleware('hop.token:link:create')->post('/api/links', StoreLinkController::class);
```

### API-Tokens

Ein Client sendet `Authorization: Bearer hlk_…`. Das Paket speichert nur den SHA-256-Hash eines Tokens. `hop.token` antwortet mit JSON der Form `{"success": false, "error": "…", "message": "…"}` und diesen Statuscodes:

| Status | Ursache |
| --- | --- |
| 401 | Header fehlt oder ist fehlerhaft, Token unbekannt oder Token abgelaufen. |
| 403 | Das Konto ist gesperrt, oder dem Token fehlt der nötige Scope. |
| 429 | Das stündliche Limit des Tokens ist aufgebraucht. |

Bei Erfolg trägt die Anfrage die Attribute `api_user` und `api_token`, und `$request->user()` gibt den Besitzer des Tokens zurück.

`hop.token` ohne Scope prüft nur, ob das Token gültig ist. `hop.token:link:create` braucht zusätzlich den Scope `link:create`. Ein Token ohne Scopes (null oder eine leere Liste) darf alles. Sonst akzeptiert `can()` eine exakte Übereinstimmung, ein Präfix-Wildcard wie `link:*` oder `*`. Die Scopes, die die Familie definiert, stehen in `ApiToken::SCOPES`: `link:read`, `link:create`, `link:write`, `bio:read`, `bio:write`, `pixel:read` und `pixel:write`.

### Rate Limit für API-Tokens

Jedes Token darf `rate_limit` Anfragen pro Stunde stellen. Ein Token, dessen `rate_limit` leer oder nicht größer als null ist, bekommt `hop-auth.api_rate_limit` (standardmäßig 100). Das Zeitfenster beginnt mit der ersten Anfrage des Tokens und endet eine Stunde später. Der Zähler liegt im Cache-Store des Rate Limiters und wird erhöht, bevor er mit dem Limit verglichen wird. So können parallele Anfragen nicht alle gleichzeitig durchkommen.

Eine abgelehnte Anfrage bekommt eine `429` mit `retry_after` im Body und einem passenden `Retry-After`-Header, beide in Sekunden bis zum Ende des Fensters.

## Stille Kontoprüfung

Eine Sitzung in deiner App kann das Konto überdauern, das die Person am Hub hat: Meldet sie sich am Hub als jemand anderes an oder ab, behält deine App das alte Konto. `hop.auth` schließt diese Lücke ohne Migration:

- Die Sitzung ist `recheck.max_age` Sekunden lang (Standard 300) **frisch**, nachdem der Hub sie zuletzt bestätigt hat, durch eine Anmeldung oder eine stille Antwort. Der Zeitpunkt steht in der Sitzung.
- Ist sie nicht frisch, macht ein Seitenaufruf (ein `GET`, der weder XHR noch JSON noch ein Prefetch ist) genau eine `prompt=none`-Runde zum Hub ([Silent SSO](https://github.com/navi-crwn/HEL.ink/blob/main/docs/silent-sso.md)). Gleiches Konto: Die Seite lädt, als wäre nichts geschehen. Anderes Konto: Die Sitzung wechselt dorthin. `login_required`: Die Person wird hier abgemeldet und sieht die Seite als Gast. Eine geschützte Seite startet dann die normale Anmeldung; eine öffentliche Seite zeigt gar keine Anmeldung.
- Schutzmaßnahmen: Ein POST oder XHR-Request wird nie umgeleitet. Es wird nichts umgeleitet, solange der Hub keine Verbindung annimmt (kurze Prüfung, 30 Sekunden zwischengespeichert). Höchstens ein Versuch pro `recheck.cooldown` Sekunden (30), also keine Schleife. Jeder andere Hub-Fehler, etwa `temporarily_unavailable`, behält die Sitzung und wartet `recheck.retry_after` Sekunden (300). Der `state` muss der sein, den die Sitzung gestartet hat.
- Seiten ohne Anmeldepflicht, die das Konto zeigen (eine Startseite mit „Dashboard“-Button), bekommen `hop.recheck` in ihre Middleware und werden so ebenfalls geprüft.
- Die Schlüssel stehen in `hop-auth.recheck` (`enabled`, `max_age`, `cooldown`, `retry_after`, `probe`, `probe_timeout`). `HOP_RECHECK=false` schaltet die ganze Prüfung ab.

## Verketteter Logout und vertrauenswürdige Hosts

`/auth/sso/logout` meldet die Person hier ab, macht die Session ungültig und erneuert das CSRF-Token. Wenn `?next=` eine vertrauenswürdige URL enthält, leitet es danach dorthin weiter. So meldet der Hub eine Person in jeder App ab. Alles andere geht zu `/`.

Eine `next`-URL ist vertrauenswürdig, wenn all das gilt:

- Ihr Host ist genau einer der vertrauenswürdigen Hosts. Es gibt kein Wildcard-Matching: `evil.hel.ink` ist nicht vertrauenswürdig, nur weil `hel.ink` es ist.
- Sie enthält keine Benutzerinfo, keine Leer- oder Steuerzeichen und keinen Backslash.
- Sie nutzt `https`. Einfaches `http` wird nur für `localhost`, `127.0.0.1` und `[::1]` akzeptiert, und nur, wenn dieser Host in der Liste der vertrauenswürdigen Hosts steht.

Die vertrauenswürdigen Hosts sind `hop-auth.trusted_hosts` plus der Hub (`HOP_HUB_URL`), diese App (`APP_URL`) und die URLs in der gemeinsamen `config/hop.php` (`hop.public_url`, `hop.hub_url` und die `url` jeder App).

Die Standardliste ist die Produktionsfamilie: `hel.ink`, `www.hel.ink`, `hub.hel.ink`, `link.hel.ink`, `bio.hel.ink`, `note.hel.ink`, `pixel.hel.ink`, `file.hel.ink` und `pay.hel.ink`. `HOP_TRUSTED_HOSTS` ersetzt diese Liste. Wenn du die Familie auf deiner eigenen Domain betreibst, setze die Hosts deiner Apps:

```ini
HOP_TRUSTED_HOSTS=example.org,hub.example.org,link.example.org,bio.example.org
```

## Kontoereignisse vom Hub

Der Hub informiert jede App, die die Fähigkeit `events` angibt, wenn mit einem Konto etwas passiert, und zwar mit `POST {app}/hop/events` und einem Body `{id, type, at, data}`. Er signiert die Anfrage mit dem Secret der App (HOP App Protocol v1: Header `X-HOP-App: helink`, `X-HOP-Timestamp`, `X-HOP-Nonce`, `X-HOP-Signature`) und wiederholt sie mit Backoff etwa sechzehn Stunden lang, wenn die Antwort kein 2xx ist. Die Typen:

| Typ | Gesendet, wenn | Hier eingebaut |
| --- | --- | --- |
| `session.revoked` | die Person sich am Hub abmeldet oder ein Admin ihre Sessions beendet | Beendet hier jede Session dieses Kontos und rotiert sein `remember_token`. |
| `account.deleted` | das Konto gelöscht wird | Beendet hier jede Session dieses Kontos. Lösche die eigenen Daten der App mit einem eigenen Handler. |
| `account.status` | sich der Status ändert (`data.status`: `active`, `suspended`) | Nichts. LinkHop und BioHop melden ein gesperrtes Konto bereits bei seiner nächsten Anfrage ab; füge einen Handler hinzu, wenn deine App mehr braucht. |
| `profile.updated` | sich das Profil ändert | Nichts. |

`data.sub` ist die Konto-ID in der gemeinsamen Tabelle `users`. Jeder andere Typ bekommt eine 200 `{handled: false}`, damit der Hub nie wiederholt, was diese App nicht kennt.

Warum das Remember-Token wichtig ist: Die Anmeldung dieses Pakets nutzt `remember: true`, daher reicht es nicht, die Session-Zeilen einer Person zu löschen. Das Remember-Cookie (standardmäßig 400 Tage) würde sie direkt wieder anmelden. `SessionRevoker` löscht die Sessions (Treiber `database` und `file`; andere Treiber lassen sich nicht nach Konto durchsuchen, und `complete` ist im Ergebnis `false`) und rotiert immer das Token.

Zwei Wege, sie zu empfangen:

1. Setze `routes.events` auf `/hop/events`. Das Paket registriert die Route hinter `hop.event`. Das prüft die Signatur mit denselben `HOP_SSO_SECRET` und `HOP_SSO_SECRET_PREV` wie bei den Anmeldetickets, akzeptiert einen Zeitstempel innerhalb von `events.max_age` Sekunden und verbraucht jede Nonce nur einmal. Jede Ablehnung ist dieselbe schlichte `401 {"ok":false,"error":"unauthorized"}`.
2. Wenn die App schon eine Route mit eigener Signaturprüfung hat, lass sie auf `Hop\Auth\Http\Controllers\EventController` zeigen (ein einfacher invokable Controller). Er prüft den Umschlag, speichert jede Ereignis-ID im Cache (eine Wiederholung wird mit `{duplicate: true}` beantwortet und nicht doppelt verarbeitet) und führt die Handler aus. Wirft ein Handler eine Exception, gibt es eine 500 ohne Details, und die ID wird vergessen, sodass die Wiederholung des Hubs ihn erneut ausführt.

Füge deine eigenen Handler in einem Service Provider hinzu:

```php
use Hop\Auth\HopEvents;

HopEvents::on('account.deleted', fn (array $data, array $event) => $this->purge((int) $data['sub']));
HopEvents::on('account.status', function (array $data) {
    // return false when the event is not about anyone you know
});
HopEvents::forget('session.revoked'); // drop the built-in handler
```

Die Ereignis-IDs brauchen einen Cache-Store, der zwischen Anfragen erhalten bleibt (`database`, `file`, `redis`), nicht `array`. Der Hub braucht eine `apps`-Zeile für die App mit der Fähigkeit `events`, sonst sendet er nichts.

## Tests

Die Tests in `tests/Feature` sind Pest-Tests, die eine Laravel-App mit der gemeinsamen Tabelle `users` brauchen. Sie laufen daher in einer App, die das Paket nutzt (HEL.ink enthält eine Kopie in `packages/auth`). Das eigene CI dieses Repositorys prüft, ob `composer.json` gültig ist, ob jede PHP-Datei unter PHP 8.3 den Lint besteht und ob die Familientabelle unten der kanonischen entspricht.

## Sicherheit

Melde Schwachstellen wie in [SECURITY.md](SECURITY.md) beschrieben, nicht in öffentlichen Issues.

## Änderungsprotokoll

Siehe [CHANGELOG.md](CHANGELOG.md).

## Selbst hosten

Du installierst dieses Paket nicht von Hand, um HOP zu betreiben. LinkHop und BioHop führen dieses öffentliche Repository in ihrer `composer.json`, daher holt `composer install` es ohne Token, und der Hub ([HEL.ink](https://github.com/navi-crwn/HEL.ink)) enthält eine Kopie in `packages/auth`. Eine App, die es nutzt, braucht einen laufenden Hub auf derselben Datenbank und diese Einstellungen: `HOP_HUB_URL`, `HOP_SSO_SECRET` (das des Hubs), `HOP_APP` und auf deiner eigenen Domain `HOP_TRUSTED_HOSTS`, weil die Standardliste nur die hel.ink-Hosts enthält. Imperium, das Admin-Panel, ist optional. Den Rest beschreibt die Anleitung der Familie: [docs/self-hosting.md](https://github.com/navi-crwn/HEL.ink/blob/main/docs/self-hosting.md).

## HOP-Familie

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

## Lizenz

Die gemeinsamen HOP-Pakete sind Open-Source-Software unter der [MIT-Lizenz](LICENSE).
