# hop/auth

[English](README.md) · [简体中文](README.zh-CN.md) · [Deutsch](README.de.md) · [Bahasa Indonesia](README.id.md) · [日本語](README.ja.md) · [Português (Brasil)](README.pt-BR.md) · **Español** · [Français](README.fr.md) · [Русский](README.ru.md)

El cliente de Laravel para HOP ID, el servicio de cuentas de la familia HOP ([hel.ink](https://hel.ink)). Permite que una app de la familia inicie la sesión de las personas a través del hub en `hub.hel.ink`, compruebe el ticket de inicio de sesión que devuelve el hub, acepte tokens de API con scopes y vuelva a cerrar la sesión de las personas.

El hub solo usa el modelo `ApiToken` y los alias de middleware. Las rutas SSO son para las apps que consumen HOP ID, como LinkHop y BioHop.

## Qué incluye

- Tres rutas para el ciclo de inicio de sesión: start, callback y logout.
- `Hop\Auth\Ticket`, que verifica un ticket firmado con HMAC-SHA256. Comprueba la firma, la audiencia (esta app), la caducidad y el `state` que envió la app. Durante una rotación también se acepta el secreto anterior.
- El middleware `hop.auth`: como `auth` de Laravel, pero un invitado pasa por el hub en lugar de ir a una página de login local.
- El middleware `hop.token`: tokens de API bearer de la tabla compartida `api_tokens`, con scopes, caducidad y un límite de peticiones por hora para cada token.
- Un cierre de sesión encadenado que solo sigue enlaces a hosts de confianza.
- Un receptor para los eventos de cuenta del hub (`POST /hop/events`): comprobación de firma, ids de evento de un solo uso y hooks para la app. Cerrar sesión en el hub, que un admin termine sesiones, una suspensión o un borrado llegan a la app por esta vía, aunque la app estuviera caída cuando ocurrió.

## Requisitos

- PHP 8.3 y Laravel 12 (`illuminate/support` ^12.0).
- La tabla compartida `users` de la familia. El paquete inicia la sesión del usuario cuyo id es el `sub` del ticket.
- Para los tokens de API, la tabla compartida `api_tokens`. El paquete no incluye una migración para ella; los tokens se crean y se revocan en el hub.
- Un almacén de sesión y un almacén de caché. La caché guarda la marca de un solo uso de cada ticket y los contadores del límite de peticiones.

## Instalación

El paquete no está en Packagist. Añade el repositorio al `composer.json` de la app:

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

Después ejecuta:

```bash
composer update hop/auth
```

El package discovery de Laravel registra `Hop\Auth\AuthServiceProvider`. El provider no carga las rutas SSO cuando `config('hop.current')` es `hub`: el hub emite tickets y nunca los consume.

## Configuración

Define estos valores en el `.env` de la app. Normalmente solo hacen falta `HOP_APP` y `HOP_SSO_SECRET`:

```ini
HOP_APP=linkhop
HOP_HUB_URL=https://hub.hel.ink
HOP_SSO_SECRET=at-least-32-characters-shared-with-the-hub
# HOP_SSO_SECRET_PREV=only-while-rotating
# HOP_USER_MODEL=App\Models\User
# HOP_TRUSTED_HOSTS=only-for-a-self-hosted-family
```

| Variable | Valor por defecto | Significado |
| --- | --- | --- |
| `HOP_APP` | `linkhop` | El slug de esta app en el `config/hop.php` del hub. También es la audiencia del ticket. |
| `HOP_HUB_URL` | `https://hub.hel.ink` | El hub que emite los tickets. |
| `HOP_SSO_SECRET` | ninguno | El secreto HMAC compartido. Los secretos de menos de 32 caracteres se ignoran, y sin un secreto válido todo ticket se rechaza como `not_configured`. |
| `HOP_SSO_SECRET_PREV` | ninguno | El secreto anterior, aceptado solo mientras haces la rotación. |
| `HOP_USER_MODEL` | `App\Models\User` | El modelo que inicia sesión y el propietario de un token de API. |
| `HOP_TRUSTED_HOSTS` | los hosts de la familia en producción | Hosts a los que puede redirigir el cierre de sesión encadenado. Ver más abajo. |
| `HOP_RECHECK` | `true` | Comprobación silenciosa de la cuenta al cargar páginas (`false` la desactiva). Ver más abajo. |
| `HOP_RECHECK_MAX_AGE` | `300` | Segundos que una sesión cuenta como reciente desde que el hub la avaló por última vez. `0` desactiva la comprobación. |

El resto de opciones no tiene variable de entorno. Publica el archivo de configuración para cambiarlas:

```bash
php artisan vendor:publish --tag=hop-auth-config
```

| Clave | Valor por defecto | Significado |
| --- | --- | --- |
| `leeway` | `30` | Segundos de desfase de reloj tolerados al comprobar `exp` e `iat` de un ticket. |
| `home` | `/dashboard` | Adónde ir tras iniciar sesión cuando la app no pidió una página. |
| `api_rate_limit` | `100` | Peticiones por hora para un token de API que no tiene límite propio. |
| `events.hub_app`, `events.max_age`, `events.seen_ttl` | `helink`, `120`, `172800` | Eventos de cuenta: el slug de app con el que firma el hub, los segundos de diferencia de reloj tolerados (cada nonce vale una vez dentro del doble de este valor) y cuánto tiempo se recuerda un id de evento. |
| `routes.events` | `null` | Ruta del receptor de eventos de cuenta. Desactivado por defecto; ver más abajo. |
| `routes.start`, `routes.callback`, `routes.logout` | `/auth/sso`, `/auth/sso/callback`, `/auth/sso/logout` | Las rutas. Pon `routes.start` a `null` para omitir las rutas del paquete y registrar las tuyas. |

## Cómo funciona un inicio de sesión

1. Un invitado abre una página protegida por `hop.auth`. El middleware lo envía a `/auth/sso?next=<page>`. Una petición JSON recibe un `401` en su lugar.
2. `start` guarda un `state` aleatorio y la página `next` en la sesión, y luego redirige a `HOP_HUB_URL/sso/{HOP_APP}?state=…&return_to=<callback url>`. Solo se conserva como `next` una ruta del mismo sitio.
3. El hub inicia la sesión de la persona y redirige a `/auth/sso/callback?ticket=…`.
4. `callback` verifica el ticket y el state. Rechaza un ticket ya usado (el `jti` se recuerda durante 5 minutos) y una cuenta suspendida (`status` igual a `suspended`).
5. Inicia la sesión del usuario cuyo id es `sub`, regenera la sesión, guarda el contenido del ticket en la sesión como `hop_profile` y redirige a `next` o a `home`.

Un ticket es `base64url(json)` y `base64url(HMAC-SHA256)` unidos por un punto. El JSON necesita `sub`, `exp` y `state`. `aud`, `iat`, `jti` y `status` son opcionales. `Ticket::verify()` lanza `Hop\Auth\TicketException` con uno de estos motivos: `not_configured`, `malformed`, `bad_signature`, `expired`, `wrong_audience`, `state_mismatch`.

### Rotación del secreto

Pon el secreto nuevo en `HOP_SSO_SECRET` y el antiguo en `HOP_SSO_SECRET_PREV`. Se acepta un ticket firmado con cualquiera de los dos. Quita el valor antiguo cuando el hub firme solo con el nuevo.

## Proteger rutas

```php
Route::middleware('hop.auth')->group(function () {
    Route::get('/dashboard', DashboardController::class);
});

Route::middleware('hop.token:link:create')->post('/api/links', StoreLinkController::class);
```

### Tokens de API

Un cliente envía `Authorization: Bearer hlk_…`. El paquete solo guarda el hash SHA-256 de un token. `hop.token` responde con JSON de la forma `{"success": false, "error": "…", "message": "…"}` y estos estados:

| Estado | Causa |
| --- | --- |
| 401 | Cabecera ausente o mal formada, token desconocido o token caducado. |
| 403 | La cuenta está suspendida, o al token le falta el scope necesario. |
| 429 | Se agotó el límite por hora del token. |

Si todo va bien, la petición lleva los atributos `api_user` y `api_token`, y `$request->user()` devuelve el propietario del token.

`hop.token` sin scope solo comprueba que el token sea válido. `hop.token:link:create` además necesita el scope `link:create`. Un token sin scopes (null o una lista vacía) puede hacerlo todo. En otro caso, `can()` acepta una coincidencia exacta, un comodín de prefijo como `link:*`, o `*`. Los scopes que define la familia están en `ApiToken::SCOPES`: `link:read`, `link:create`, `link:write`, `bio:read`, `bio:write`, `pixel:read` y `pixel:write`.

### Límite de peticiones de los tokens de API

Cada token puede hacer `rate_limit` peticiones por hora. Un token cuyo `rate_limit` está vacío o no es mayor que cero recibe `hop-auth.api_rate_limit` (100 por defecto). La ventana empieza con la primera petición del token y termina una hora después. El contador vive en el almacén de caché del rate limiter y se incrementa antes de compararlo con el límite, así que las peticiones paralelas no pueden pasar todas a la vez.

Una petición rechazada recibe un `429` con `retry_after` en el cuerpo y una cabecera `Retry-After` equivalente, ambos en segundos hasta que termine la ventana.

## Comprobación silenciosa de la cuenta

Una sesión de tu app puede sobrevivir a la cuenta que la persona tiene en el hub: si entra en el hub con otra cuenta, o sale, tu app conserva la cuenta anterior. `hop.auth` cierra esa brecha sin migración:

- La sesión es **reciente** durante `recheck.max_age` segundos (300 por defecto) desde que el hub la avaló por última vez, con un inicio de sesión o con una respuesta silenciosa. La hora se guarda en la sesión.
- Si no es reciente, al cargar una página (un `GET` que no sea XHR ni JSON ni una precarga) se hace un único viaje `prompt=none` al hub ([SSO silencioso](https://github.com/navi-crwn/HEL.ink/blob/main/docs/silent-sso.md)). Misma cuenta: la página carga como si nada. Otra cuenta: la sesión cambia a ella. `login_required`: se cierra la sesión aquí y la persona ve la página como invitada. Una página protegida inicia entonces el inicio de sesión normal; una página pública no muestra ningún inicio de sesión.
- Salvaguardas: un POST o una petición XHR nunca se redirige. Mientras el hub no responda 2xx en su URL de salud (`/up`), por ejemplo con un 502 o un tiempo de espera agotado, no se redirige ni se cierra la sesión de nadie: la sesión sigue siendo de confianza y la siguiente carga de página tras caducar el veredicto en caché (60 segundos) lo intenta de nuevo. Como máximo un intento cada `recheck.cooldown` segundos (30), así que no hay bucles. Cualquier otro error del hub, como `temporarily_unavailable`, conserva la sesión y espera `recheck.retry_after` segundos (300). El `state` debe ser el que inició la sesión.
- Las páginas que no exigen sesión pero muestran la cuenta (una portada con un botón «Panel») llevan `hop.recheck` en su middleware, para que también se comprueben.
- Las claves están en `hop-auth.recheck` (`enabled`, `max_age`, `cooldown`, `retry_after`, `probe`, `probe_path`, `probe_timeout`, `probe_ttl`). `HOP_RECHECK=false` desactiva toda la comprobación.

## Cierre de sesión encadenado y hosts de confianza

`/auth/sso/logout` cierra la sesión de la persona aquí, invalida la sesión y regenera el token CSRF. Si `?next=` contiene una URL de confianza, después redirige allí. Así es como el hub cierra la sesión de una persona en todas las apps. Cualquier otra cosa va a `/`.

Una URL `next` es de confianza cuando se cumple todo esto:

- Su host es exactamente uno de los hosts de confianza. No hay coincidencia con comodines: `evil.hel.ink` no es de confianza por el hecho de que `hel.ink` sí lo sea.
- No tiene información de usuario, ni espacios ni caracteres de control, ni barra invertida.
- Usa `https`. El `http` simple solo se acepta para `localhost`, `127.0.0.1` y `[::1]`, y solo cuando ese host está en la lista de confianza.

Los hosts de confianza son `hop-auth.trusted_hosts` más el hub (`HOP_HUB_URL`), esta app (`APP_URL`) y las URLs del `config/hop.php` compartido (`hop.public_url`, `hop.hub_url` y la `url` de cada app).

La lista por defecto es la familia en producción: `hel.ink`, `www.hel.ink`, `hub.hel.ink`, `link.hel.ink`, `bio.hel.ink`, `note.hel.ink`, `pixel.hel.ink`, `file.hel.ink` y `pay.hel.ink`. `HOP_TRUSTED_HOSTS` sustituye esa lista. Si ejecutas la familia en tu propio dominio, define los hosts de tus apps:

```ini
HOP_TRUSTED_HOSTS=example.org,hub.example.org,link.example.org,bio.example.org
```

## Eventos de cuenta del hub

El hub avisa a cada app que declara la capacidad `events` cuando algo le pasa a una cuenta, con `POST {app}/hop/events` y un cuerpo `{id, type, at, data}`. Firma la petición con el secreto de la app (HOP App Protocol v1: cabeceras `X-HOP-App: helink`, `X-HOP-Timestamp`, `X-HOP-Nonce`, `X-HOP-Signature`) y reintenta con backoff durante unas dieciséis horas si la respuesta no es un 2xx. Los tipos:

| Tipo | Se envía cuando | Incluido aquí |
| --- | --- | --- |
| `session.revoked` | la persona cierra sesión en el hub, o un admin termina sus sesiones | Termina aquí todas las sesiones de esa cuenta y rota su `remember_token`. |
| `account.deleted` | se borra la cuenta | Termina aquí todas las sesiones de esa cuenta. Borra los datos propios de la app con un handler tuyo. |
| `account.status` | cambia el estado (`data.status`: `active`, `suspended`) | Nada. LinkHop y BioHop ya cierran la sesión de una cuenta suspendida en su siguiente petición; añade un handler si tu app necesita más. |
| `profile.updated` | cambia el perfil | Nada. |

`data.sub` es el id de la cuenta en la tabla compartida `users`. Cualquier otro tipo recibe un 200 `{handled: false}`, para que el hub nunca reintente lo que esta app no conoce.

Por qué importa el remember token: el inicio de sesión de este paquete usa `remember: true`, así que borrar las filas de sesión de una persona no basta. La cookie remember (400 días por defecto) volvería a iniciar su sesión directamente. `SessionRevoker` borra las sesiones (drivers `database` y `file`; los demás drivers no permiten buscar por cuenta, y `complete` es `false` en el resultado) y siempre rota el token.

Dos formas de recibirlos:

1. Pon `routes.events` a `/hop/events`. El paquete registra la ruta detrás de `hop.event`, que comprueba la firma con los mismos `HOP_SSO_SECRET` y `HOP_SSO_SECRET_PREV` que los tickets de inicio de sesión, acepta una marca de tiempo dentro de `events.max_age` segundos y gasta cada nonce una sola vez. Todo rechazo es el mismo `401 {"ok":false,"error":"unauthorized"}` sin más detalle.
2. Si la app ya tiene una ruta con su propia comprobación de firma, apúntala a `Hop\Auth\Http\Controllers\EventController` (un controlador invocable simple). Comprueba el sobre, recuerda cada id de evento en la caché (un reintento recibe `{duplicate: true}` y no se procesa dos veces) y ejecuta los handlers. Si un handler lanza una excepción, la respuesta es un 500 sin detalles y el id se olvida, así que el reintento del hub lo ejecuta de nuevo.

Añade tus propios handlers en un service provider:

```php
use Hop\Auth\HopEvents;

HopEvents::on('account.deleted', fn (array $data, array $event) => $this->purge((int) $data['sub']));
HopEvents::on('account.status', function (array $data) {
    // return false when the event is not about anyone you know
});
HopEvents::forget('session.revoked'); // drop the built-in handler
```

Los ids de evento necesitan un almacén de caché que persista entre peticiones (`database`, `file`, `redis`), no `array`. El hub debe tener una fila en `apps` para la app con la capacidad `events`; si no, no envía nada.

## Tests

Los tests de `tests/Feature` son tests de Pest que necesitan una app Laravel con la tabla compartida `users`, así que se ejecutan dentro de una app que usa el paquete (HEL.ink incluye una copia en `packages/auth`). La CI propia de este repositorio comprueba que `composer.json` es válido, que todos los archivos PHP pasan el lint con PHP 8.3 y que la tabla de la familia de abajo coincide con la canónica.

## Seguridad

Informa de vulnerabilidades como se describe en [SECURITY.md](SECURITY.md), no en issues públicos.

## Registro de cambios

Consulta [CHANGELOG.md](CHANGELOG.md).

## Autoalojamiento

No necesitas instalar este paquete a mano para ejecutar HOP. LinkHop y BioHop listan este repositorio público en su `composer.json`, así que `composer install` lo descarga sin token, y el hub ([HEL.ink](https://github.com/navi-crwn/HEL.ink)) lleva una copia en `packages/auth`. Una app que lo usa necesita un hub en marcha sobre la misma base de datos y estos ajustes: `HOP_HUB_URL`, `HOP_SSO_SECRET` (el del hub), `HOP_APP` y, en tu propio dominio, `HOP_TRUSTED_HOSTS`, porque la lista por defecto solo contiene los hosts de hel.ink. Imperium, el panel de administración, es opcional. La guía de la familia cubre el resto: [docs/self-hosting.md](https://github.com/navi-crwn/HEL.ink/blob/main/docs/self-hosting.md).

## Familia HOP

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

## Licencia

Los paquetes compartidos de HOP son software de código abierto con [licencia MIT](LICENSE).
