# hop/auth

[English](README.md) · [简体中文](README.zh-CN.md) · [Deutsch](README.de.md) · [Bahasa Indonesia](README.id.md) · [日本語](README.ja.md) · [Português (Brasil)](README.pt-BR.md) · [Español](README.es.md) · [Français](README.fr.md) · **Русский**

Клиент Laravel для HOP ID — сервиса учётных записей семейства HOP ([hel.ink](https://hel.ink)). Он позволяет приложению семейства выполнять вход пользователей через хаб на `hub.hel.ink`, проверять тикет входа, который возвращает хаб, принимать API-токены со скоупами и выполнять выход пользователей.

Сам хаб использует только модель `ApiToken` и алиасы middleware. SSO-маршруты нужны приложениям, которые используют HOP ID, например LinkHop и BioHop.

## Что входит в пакет

- Три маршрута для цикла входа: start, callback и logout.
- `Hop\Auth\Ticket`, который проверяет тикет, подписанный HMAC-SHA256. Он проверяет подпись, аудиторию (это приложение), срок действия и `state`, отправленный приложением. Во время ротации принимается и предыдущий секрет.
- Middleware `hop.auth`: как `auth` в Laravel, но гостя отправляют через хаб, а не на локальную страницу входа.
- Middleware `hop.token`: bearer API-токены из общей таблицы `api_tokens`, со скоупами, сроком действия и почасовым лимитом запросов для каждого токена.
- Цепочка выхода, которая переходит только по ссылкам на доверенные хосты.
- Приёмник событий учётных записей от хаба (`POST /hop/events`): проверка подписи, одноразовые id событий и хуки для приложения. Выход в хабе, завершение сессий администратором, блокировка или удаление доходят до приложения этим путём, даже если приложение в тот момент не работало.

## Требования

- PHP 8.3 и Laravel 12 (`illuminate/support` ^12.0).
- Общая таблица `users` семейства. Пакет выполняет вход пользователя, чей id равен `sub` в тикете.
- Для API-токенов — общая таблица `api_tokens`. Пакет не содержит для неё миграцию; токены создаются и отзываются в хабе.
- Хранилище сессий и хранилище кэша. В кэше хранятся одноразовые метки тикетов и счётчики лимита запросов.

## Установка

Пакета нет на Packagist. Добавьте репозиторий в `composer.json` приложения:

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

Затем выполните:

```bash
composer update hop/auth
```

Механизм package discovery в Laravel регистрирует `Hop\Auth\AuthServiceProvider`. Провайдер не загружает SSO-маршруты, если `config('hop.current')` равен `hub`: хаб выдаёт тикеты и никогда их не принимает.

## Настройка

Задайте эти значения в `.env` приложения. Обычно нужны только `HOP_APP` и `HOP_SSO_SECRET`:

```ini
HOP_APP=linkhop
HOP_HUB_URL=https://hub.hel.ink
HOP_SSO_SECRET=at-least-32-characters-shared-with-the-hub
# HOP_SSO_SECRET_PREV=only-while-rotating
# HOP_USER_MODEL=App\Models\User
# HOP_TRUSTED_HOSTS=only-for-a-self-hosted-family
```

| Переменная | По умолчанию | Значение |
| --- | --- | --- |
| `HOP_APP` | `linkhop` | Slug этого приложения в `config/hop.php` хаба. Это также аудитория тикета. |
| `HOP_HUB_URL` | `https://hub.hel.ink` | Хаб, который выдаёт тикеты. |
| `HOP_SSO_SECRET` | нет | Общий секрет HMAC. Секреты короче 32 символов игнорируются, а без пригодного секрета каждый тикет отклоняется как `not_configured`. |
| `HOP_SSO_SECRET_PREV` | нет | Предыдущий секрет; принимается только на время ротации. |
| `HOP_USER_MODEL` | `App\Models\User` | Модель, которая входит в систему, и владелец API-токена. |
| `HOP_TRUSTED_HOSTS` | хосты семейства в продакшене | Хосты, на которые может перенаправлять цепочка выхода. См. ниже. |
| `HOP_RECHECK` | `true` | Тихая перепроверка учётной записи при загрузке страниц (`false` отключает её). См. ниже. |
| `HOP_RECHECK_MAX_AGE` | `300` | Сколько секунд сессия считается свежей после последнего подтверждения хабом. `0` отключает перепроверку. |

У остальных параметров нет переменной окружения. Чтобы их изменить, опубликуйте файл конфигурации:

```bash
php artisan vendor:publish --tag=hop-auth-config
```

| Ключ | По умолчанию | Значение |
| --- | --- | --- |
| `leeway` | `30` | Допустимое расхождение часов в секундах при проверке `exp` и `iat` тикета. |
| `home` | `/dashboard` | Куда попасть после входа, если приложение не запросило страницу. |
| `api_rate_limit` | `100` | Запросов в час для API-токена без собственного лимита. |
| `events.hub_app`, `events.max_age`, `events.seen_ttl` | `helink`, `120`, `172800` | События учётных записей: slug приложения, которым подписывает хаб, допустимое расхождение часов в секундах (каждый nonce действует один раз в пределах удвоенного значения) и как долго запоминается id события. |
| `routes.events` | `null` | Путь приёмника событий учётных записей. По умолчанию выключен; см. ниже. |
| `routes.start`, `routes.callback`, `routes.logout` | `/auth/sso`, `/auth/sso/callback`, `/auth/sso/logout` | Пути маршрутов. Задайте `routes.start` равным `null`, чтобы пропустить маршруты пакета и зарегистрировать свои. |

## Как проходит вход

1. Гость открывает страницу, защищённую `hop.auth`. Middleware отправляет его на `/auth/sso?next=<page>`. JSON-запрос вместо этого получает `401`.
2. `start` сохраняет случайный `state` и страницу `next` в сессии, затем перенаправляет на `HOP_HUB_URL/sso/{HOP_APP}?state=…&return_to=<callback url>`. В качестве `next` сохраняется только путь на том же сайте.
3. Хаб выполняет вход пользователя и перенаправляет на `/auth/sso/callback?ticket=…`.
4. `callback` проверяет тикет и state. Он отклоняет уже использованный тикет (`jti` запоминается на 5 минут) и заблокированную учётную запись (`status` со значением `suspended`).
5. Он выполняет вход пользователя с id `sub`, пересоздаёт сессию, сохраняет содержимое тикета в сессии как `hop_profile` и перенаправляет на `next` или на `home`.

Тикет — это `base64url(json)` и `base64url(HMAC-SHA256)`, соединённые точкой. В JSON обязательны `sub`, `exp` и `state`. `aud`, `iat`, `jti` и `status` необязательны. `Ticket::verify()` выбрасывает `Hop\Auth\TicketException` с одной из причин: `not_configured`, `malformed`, `bad_signature`, `expired`, `wrong_audience`, `state_mismatch`.

### Ротация секрета

Поместите новый секрет в `HOP_SSO_SECRET`, а старый — в `HOP_SSO_SECRET_PREV`. Принимается тикет, подписанный любым из них. Удалите старое значение, когда хаб будет подписывать только новым.

## Защита маршрутов

```php
Route::middleware('hop.auth')->group(function () {
    Route::get('/dashboard', DashboardController::class);
});

Route::middleware('hop.token:link:create')->post('/api/links', StoreLinkController::class);
```

### API-токены

Клиент отправляет `Authorization: Bearer hlk_…`. Пакет хранит только хеш SHA-256 токена. `hop.token` отвечает JSON вида `{"success": false, "error": "…", "message": "…"}` и такими статусами:

| Статус | Причина |
| --- | --- |
| 401 | Заголовок отсутствует или искажён, токен неизвестен или истёк. |
| 403 | Учётная запись заблокирована, или у токена нет нужного скоупа. |
| 429 | Почасовой лимит токена исчерпан. |

При успехе запрос несёт атрибуты `api_user` и `api_token`, а `$request->user()` возвращает владельца токена.

`hop.token` без скоупа только проверяет, что токен действителен. `hop.token:link:create` дополнительно требует скоуп `link:create`. Токену без скоупов (null или пустой список) разрешено всё. В остальных случаях `can()` принимает точное совпадение, префиксный шаблон вроде `link:*` или `*`. Скоупы, которые определяет семейство, перечислены в `ApiToken::SCOPES`: `link:read`, `link:create`, `link:write`, `bio:read`, `bio:write`, `pixel:read` и `pixel:write`. Хаб HOP при создании нового токена предлагает только скоупы `link:*` (токены хаба предназначены для LinkHop) и отклоняет новый токен со скоупом `pixel:*`, потому что PixelHop выдаёт собственные ключи. Скоупы `bio:*` есть в списке, но форма их не предлагает. Токен, созданный раньше со скоупом `pixel:*`, остаётся действительным, но этот скоуп больше ничего не даёт: PixelHop принимает только собственные ключи `ph_`, но не токен хаба.

### Лимит запросов для API-токенов

Каждый токен может выполнить `rate_limit` запросов в час. Токен, у которого `rate_limit` пуст или не больше нуля, получает `hop-auth.api_rate_limit` (по умолчанию 100). Окно начинается с первого запроса токена и заканчивается через час. Счётчик хранится в хранилище кэша rate limiter и увеличивается до сравнения с лимитом, поэтому параллельные запросы не могут пройти все сразу.

Отклонённый запрос получает `429` с `retry_after` в теле и соответствующим заголовком `Retry-After`; оба значения — в секундах до конца окна.

## Тихая перепроверка учётной записи

Сессия в вашем приложении может пережить учётную запись, которая у человека на хабе: он входит на хабе под другой записью или выходит, а приложение остаётся со старой. `hop.auth` закрывает эту брешь без миграции:

- Сессия **свежая** `recheck.max_age` секунд (по умолчанию 300) после последнего подтверждения хабом: входом или тихим ответом. Время хранится в сессии.
- Если сессия не свежая, загрузка страницы (`GET`, который не XHR, не JSON и не предзагрузка) делает один обход к хабу с `prompt=none` ([тихий SSO](https://github.com/navi-crwn/HEL.ink/blob/main/docs/silent-sso.md)). Та же запись: страница загружается как ни в чём не бывало. Другая запись: сессия переключается на неё. `login_required`: человек выходит здесь и видит страницу как гость. Защищённая страница затем запускает обычный вход; публичная страница не показывает вход вообще.
- Предохранители: POST и XHR-запросы никогда не перенаправляются. Пока хаб не отвечает 2xx на свой health-URL (`/up`), например 502 или таймаутом, ничего не перенаправляется и никто не разлогинивается: сессия остаётся доверенной, а следующая загрузка страницы после истечения кэшированного вердикта (60 секунд) пробует снова. Не более одной попытки за `recheck.cooldown` секунд (30), поэтому цикла не будет. Любая другая ошибка хаба, например `temporarily_unavailable`, сохраняет сессию и ждёт `recheck.retry_after` секунд (300). `state` должен быть тем, что начала сессия.
- Страницы без обязательного входа, показывающие учётную запись (главная с кнопкой «Панель»), получают `hop.recheck` в middleware и тоже проверяются.
- Ключи лежат в `hop-auth.recheck` (`enabled`, `max_age`, `cooldown`, `retry_after`, `probe`, `probe_path`, `probe_timeout`, `probe_ttl`). `HOP_RECHECK=false` отключает всю проверку.

## Цепочка выхода и доверенные хосты

`/auth/sso/logout` выполняет выход пользователя здесь, аннулирует сессию и пересоздаёт CSRF-токен. Если `?next=` содержит доверенный URL, затем выполняется перенаправление туда. Так хаб выполняет выход пользователя из всех приложений. Всё остальное ведёт на `/`.

URL в `next` считается доверенным, если выполнены все условия:

- Его хост в точности совпадает с одним из доверенных хостов. Шаблонов нет: `evil.hel.ink` не становится доверенным оттого, что доверенным является `hel.ink`.
- В нём нет данных пользователя, пробелов, управляющих символов и обратной косой черты.
- Он использует `https`. Обычный `http` принимается только для `localhost`, `127.0.0.1` и `[::1]` и только если этот хост есть в списке доверенных.

Доверенные хосты — это `hop-auth.trusted_hosts`, а также хаб (`HOP_HUB_URL`), это приложение (`APP_URL`) и URL из общего `config/hop.php` (`hop.public_url`, `hop.hub_url` и `url` каждого приложения).

Список по умолчанию — семейство в продакшене: `hel.ink`, `www.hel.ink`, `hub.hel.ink`, `link.hel.ink`, `bio.hel.ink`, `note.hel.ink`, `pixel.hel.ink`, `file.hel.ink` и `pay.hel.ink`. `HOP_TRUSTED_HOSTS` заменяет этот список. Если вы запускаете семейство на своём домене, укажите хосты своих приложений:

```ini
HOP_TRUSTED_HOSTS=example.org,hub.example.org,link.example.org,bio.example.org
```

## События учётных записей от хаба

Хаб сообщает каждому приложению, которое объявляет возможность `events`, когда с учётной записью что-то происходит: отправляет `POST {app}/hop/events` с телом `{id, type, at, data}`. Он подписывает запрос секретом приложения (HOP App Protocol v1: заголовки `X-HOP-App: helink`, `X-HOP-Timestamp`, `X-HOP-Nonce`, `X-HOP-Signature`) и повторяет попытки с backoff около шестнадцати часов, если ответ не 2xx. Типы:

| Тип | Когда отправляется | Что встроено здесь |
| --- | --- | --- |
| `session.revoked` | пользователь выходит в хабе, или администратор завершает его сессии | Завершает здесь все сессии этой учётной записи и меняет её `remember_token`. |
| `account.deleted` | учётная запись удалена | Завершает здесь все сессии этой учётной записи. Собственные данные приложения удалите своим обработчиком. |
| `account.status` | меняется статус (`data.status`: `active`, `suspended`) | Ничего. LinkHop и BioHop и так выполняют выход заблокированной учётной записи при её следующем запросе; добавьте обработчик, если вашему приложению нужно больше. |
| `profile.updated` | меняется профиль | Ничего. |

`data.sub` — id учётной записи в общей таблице `users`. Любой другой тип получает 200 `{handled: false}`, чтобы хаб никогда не повторял то, чего это приложение не знает.

Почему важен remember token: вход в этом пакете использует `remember: true`, поэтому удалить строки сессий пользователя недостаточно. Cookie remember (по умолчанию 400 дней) сразу же снова выполнит вход. `SessionRevoker` удаляет сессии (драйверы `database` и `file`; в других драйверах нельзя искать по учётной записи, и в результате `complete` равен `false`) и всегда меняет токен.

Два способа их получать:

1. Задайте `routes.events` равным `/hop/events`. Пакет регистрирует маршрут за `hop.event`, который проверяет подпись теми же `HOP_SSO_SECRET` и `HOP_SSO_SECRET_PREV`, что и тикеты входа, принимает метку времени в пределах `events.max_age` секунд и расходует каждый nonce один раз. Любой отказ — один и тот же голый `401 {"ok":false,"error":"unauthorized"}`.
2. Если у приложения уже есть маршрут со своей проверкой подписи, направьте его на `Hop\Auth\Http\Controllers\EventController` (обычный invokable-контроллер). Он проверяет конверт, запоминает каждый id события в кэше (на повтор отвечает `{duplicate: true}` и не обрабатывает его дважды) и запускает обработчики. Если обработчик выбрасывает исключение, ответ — 500 без подробностей, а id забывается, так что повтор от хаба запустит его снова.

Добавьте свои обработчики в service provider:

```php
use Hop\Auth\HopEvents;

HopEvents::on('account.deleted', fn (array $data, array $event) => $this->purge((int) $data['sub']));
HopEvents::on('account.status', function (array $data) {
    // return false when the event is not about anyone you know
});
HopEvents::forget('session.revoked'); // drop the built-in handler
```

Для id событий нужно хранилище кэша, которое сохраняется между запросами (`database`, `file`, `redis`), а не `array`. В хабе должна быть строка `apps` для приложения с возможностью `events`, иначе он ничего не отправляет.

## Тесты

Тесты в `tests/Feature` — это тесты Pest, которым нужно приложение Laravel с общей таблицей `users`, поэтому они запускаются внутри приложения, использующего пакет (HEL.ink содержит копию в `packages/auth`). Собственный CI этого репозитория проверяет, что `composer.json` валиден, что каждый PHP-файл проходит lint под PHP 8.3 и что таблица семейства ниже совпадает с канонической.

## Безопасность

Сообщайте об уязвимостях так, как описано в [SECURITY.md](SECURITY.md), а не в публичных issues.

## Журнал изменений

См. [CHANGELOG.md](CHANGELOG.md).

## Собственный хостинг

Чтобы запустить HOP, этот пакет не нужно устанавливать вручную. LinkHop и BioHop указывают этот публичный репозиторий в своём `composer.json`, поэтому `composer install` загружает его без токена, а хаб ([HEL.ink](https://github.com/navi-crwn/HEL.ink)) содержит копию в `packages/auth`. Приложению, которое его использует, нужен работающий хаб на той же базе данных и такие настройки: `HOP_HUB_URL`, `HOP_SSO_SECRET` (секрет хаба), `HOP_APP`, а на своём домене — `HOP_TRUSTED_HOSTS`, потому что список по умолчанию содержит только хосты hel.ink. Imperium, панель администратора, необязательна. Остальное описано в руководстве семейства: [docs/self-hosting.md](https://github.com/navi-crwn/HEL.ink/blob/main/docs/self-hosting.md).

## Семейство HOP

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

## Лицензия

Общие пакеты HOP — это программное обеспечение с открытым исходным кодом под [лицензией MIT](LICENSE).
