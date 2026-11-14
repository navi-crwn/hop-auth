# hop/auth

[English](README.md) · **简体中文** · [Deutsch](README.de.md) · [Bahasa Indonesia](README.id.md) · [日本語](README.ja.md) · [Português (Brasil)](README.pt-BR.md) · [Español](README.es.md) · [Français](README.fr.md) · [Русский](README.ru.md)

HOP ID 的 Laravel 客户端。HOP ID 是 HOP 家族（[hel.ink](https://hel.ink)）的账号服务。家族中的应用可以用它通过 `hub.hel.ink` 上的 hub 让用户登录、校验 hub 返回的登录票据、接受带作用域的 API 令牌，以及让用户退出登录。

hub 本身只使用 `ApiToken` 模型和中间件别名。SSO 路由供使用 HOP ID 的应用使用，例如 LinkHop 和 BioHop。

## 提供的功能

- 三条用于登录往返的路由：start、callback 和 logout。
- `Hop\Auth\Ticket`，用于校验以 HMAC-SHA256 签名的票据。它检查签名、受众（即本应用）、过期时间以及应用发出的 `state`。密钥轮换期间也接受上一个密钥。
- `hop.auth` 中间件：与 Laravel 的 `auth` 类似，但访客会被引导到 hub，而不是本地登录页。
- `hop.token` 中间件：来自共享 `api_tokens` 表的 Bearer API 令牌，支持作用域、过期时间以及每个令牌的每小时速率限制。
- 链式退出登录，只跟随指向受信任主机的链接。
- hub 账号事件的接收端（`POST /hop/events`）：签名校验、只能使用一次的事件 id，以及供应用使用的钩子。在 hub 退出登录、管理员结束会话、账号被停用或删除，都会通过这条路径送达应用，即使事件发生时应用处于停机状态。

## 要求

- PHP 8.3 和 Laravel 12（`illuminate/support` ^12.0）。
- 家族共享的 `users` 表。本包会登录 id 等于票据中 `sub` 的用户。
- 使用 API 令牌时，需要共享的 `api_tokens` 表。本包不附带该表的迁移；令牌在 hub 上创建和吊销。
- 一个会话存储和一个缓存存储。缓存保存每张票据的一次性标记和速率限制计数器。

## 安装

本包不在 Packagist 上。把仓库添加到应用的 `composer.json`：

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

然后运行：

```bash
composer update hop/auth
```

Laravel 的包自动发现会注册 `Hop\Auth\AuthServiceProvider`。当 `config('hop.current')` 为 `hub` 时，该 provider 不加载 SSO 路由：hub 只签发票据，从不消费票据。

## 配置

在应用的 `.env` 中设置以下变量。通常只需要 `HOP_APP` 和 `HOP_SSO_SECRET`：

```ini
HOP_APP=linkhop
HOP_HUB_URL=https://hub.hel.ink
HOP_SSO_SECRET=at-least-32-characters-shared-with-the-hub
# HOP_SSO_SECRET_PREV=only-while-rotating
# HOP_USER_MODEL=App\Models\User
# HOP_TRUSTED_HOSTS=only-for-a-self-hosted-family
```

| 变量 | 默认值 | 含义 |
| --- | --- | --- |
| `HOP_APP` | `linkhop` | 本应用在 hub 的 `config/hop.php` 中的 slug，也是票据的受众。 |
| `HOP_HUB_URL` | `https://hub.hel.ink` | 签发票据的 hub。 |
| `HOP_SSO_SECRET` | 无 | 共享的 HMAC 密钥。短于 32 个字符的密钥会被忽略；没有可用密钥时，所有票据都会以 `not_configured` 被拒绝。 |
| `HOP_SSO_SECRET_PREV` | 无 | 上一个密钥，仅在轮换期间接受。 |
| `HOP_USER_MODEL` | `App\Models\User` | 登录所用的模型，也是 API 令牌的所有者。 |
| `HOP_TRUSTED_HOSTS` | 生产环境的家族主机 | 链式退出登录可以重定向到的主机。见下文。 |
| `HOP_RECHECK` | `true` | 页面加载时的静默账号复核(`false` 关闭)。见下文。 |
| `HOP_RECHECK_MAX_AGE` | `300` | hub 最近一次确认后,会话被视为“新鲜”的秒数。`0` 关闭复核。 |

其余选项没有对应的环境变量。如需修改，请发布配置文件：

```bash
php artisan vendor:publish --tag=hop-auth-config
```

| 键 | 默认值 | 含义 |
| --- | --- | --- |
| `leeway` | `30` | 检查票据的 `exp` 和 `iat` 时容许的时钟偏差（秒）。 |
| `home` | `/dashboard` | 应用未指定页面时，登录后跳转的位置。 |
| `api_rate_limit` | `100` | 没有自身限制的 API 令牌每小时可发出的请求数。 |
| `events.hub_app`, `events.max_age`, `events.seen_ttl` | `helink`, `120`, `172800` | 账号事件：hub 签名时使用的应用 slug、容许的时钟差（秒；每个 nonce 在该值两倍的时间内只能使用一次），以及事件 id 被记住的时长。 |
| `routes.events` | `null` | 账号事件接收端的路径。默认关闭；见下文。 |
| `routes.start`, `routes.callback`, `routes.logout` | `/auth/sso`, `/auth/sso/callback`, `/auth/sso/logout` | 路由路径。把 `routes.start` 设为 `null` 可跳过本包的路由，改为注册你自己的路由。 |

## 登录流程

1. 访客打开受 `hop.auth` 保护的页面。中间件把访客送到 `/auth/sso?next=<page>`。JSON 请求则会收到 `401`。
2. `start` 在会话中保存一个随机的 `state` 和 `next` 页面，然后重定向到 `HOP_HUB_URL/sso/{HOP_APP}?state=…&return_to=<callback url>`。只有同站路径才会作为 `next` 保留。
3. hub 让用户登录，然后重定向到 `/auth/sso/callback?ticket=…`。
4. `callback` 校验票据和 state。它会拒绝已经用过的票据（`jti` 会被记住 5 分钟），也会拒绝已停用的账号（`status` 为 `suspended`）。
5. 它登录 id 为 `sub` 的用户，重新生成会话，把票据内容以 `hop_profile` 保存在会话中，然后重定向到 `next` 或 `home`。

票据由 `base64url(json)` 和 `base64url(HMAC-SHA256)` 用一个点连接而成。JSON 必须包含 `sub`、`exp` 和 `state`。`aud`、`iat`、`jti` 和 `status` 是可选的。`Ticket::verify()` 会抛出 `Hop\Auth\TicketException`，原因为以下之一：`not_configured`, `malformed`, `bad_signature`, `expired`, `wrong_audience`, `state_mismatch`。

### 密钥轮换

把新密钥放进 `HOP_SSO_SECRET`，旧密钥放进 `HOP_SSO_SECRET_PREV`。用其中任意一个签名的票据都会被接受。等 hub 只用新密钥签名后，删除旧值。

## 保护路由

```php
Route::middleware('hop.auth')->group(function () {
    Route::get('/dashboard', DashboardController::class);
});

Route::middleware('hop.token:link:create')->post('/api/links', StoreLinkController::class);
```

### API 令牌

客户端发送 `Authorization: Bearer hlk_…`。本包只存储令牌的 SHA-256 哈希。`hop.token` 以 `{"success": false, "error": "…", "message": "…"}` 形式的 JSON 和以下状态码响应：

| 状态码 | 原因 |
| --- | --- |
| 401 | 缺少请求头或格式错误、令牌未知或令牌已过期。 |
| 403 | 账号已停用，或令牌缺少所需的作用域。 |
| 429 | 令牌的每小时限额已用完。 |

成功时，请求带有 `api_user` 和 `api_token` 属性，`$request->user()` 返回令牌的所有者。

不带作用域的 `hop.token` 只检查令牌是否有效。`hop.token:link:create` 还要求 `link:create` 作用域。没有作用域的令牌（null 或空列表）可以做任何事。否则，`can()` 接受精确匹配、`link:*` 这样的前缀通配符，或 `*`。家族定义的作用域在 `ApiToken::SCOPES` 中：`link:read`、`link:create`、`link:write`、`bio:read`、`bio:write`、`pixel:read` 和 `pixel:write`。HOP 中枢在创建新令牌时只提供 `link:*` 作用域（中枢令牌用于 LinkHop），并拒绝带有 `pixel:*` 作用域的新令牌，因为 PixelHop 自己签发密钥。`bio:*` 作用域在列表中，但表单不提供。以前带 `pixel:*` 作用域创建的令牌仍然有效，但该作用域已不起任何作用：PixelHop 只接受自己的 `ph_` 密钥，不接受中枢令牌。

### API 令牌速率限制

每个令牌每小时可以发出 `rate_limit` 个请求。`rate_limit` 为空或不大于零的令牌使用 `hop-auth.api_rate_limit`（默认 100）。时间窗口从令牌的第一个请求开始，一小时后结束。计数器保存在速率限制器的缓存存储中，并且先递增再与限额比较，因此并行请求无法同时全部通过。

被拒绝的请求会收到 `429`，响应体中带有 `retry_after`，并带有对应的 `Retry-After` 请求头，两者都是距窗口结束的秒数。

## 静默账号复核

你的应用里的会话可能比此人在 hub 上的账号活得更久:他在 hub 上换了账号登录或退出登录,而你的应用仍然保留旧账号。`hop.auth` 无需迁移即可堵上这个缺口:

- 会话在 hub 最近一次确认(通过一次登录或一次静默应答)之后的 `recheck.max_age` 秒内(默认 300)是**新鲜**的。时间保存在会话里。
- 不新鲜时,加载页面(不是 XHR、不是 JSON、也不是预取的 `GET`)会向 hub 做一次 `prompt=none` 往返([静默 SSO](https://github.com/navi-crwn/HEL.ink/blob/main/docs/silent-sso.md))。账号相同:页面照常加载,毫无察觉。账号不同:会话切换到该账号。`login_required`:此人在这里退出登录,以访客身份看到该页面。受保护的页面随后开始正常登录;公开页面则完全不显示登录。
- 保护措施:POST 或 XHR 请求绝不重定向。hub 的健康检查地址(`/up`)没有返回 2xx 时(例如 502 或超时),不重定向也不让任何人退出登录:会话仍被信任,缓存的判定过期(60 秒)后的下一次页面加载会再试。每 `recheck.cooldown` 秒(30)最多尝试一次,因此不会死循环。hub 的其他错误,例如 `temporarily_unavailable`,会保留会话并等待 `recheck.retry_after` 秒(300)。`state` 必须是该会话发起的那个。
- 不需要登录但会显示账号的页面(带“控制台”按钮的首页),请在其中间件里加上 `hop.recheck`,这样它们也会被复核。
- 配置项在 `hop-auth.recheck`(`enabled`、`max_age`、`cooldown`、`retry_after`、`probe`、`probe_path`、`probe_timeout`、`probe_ttl`)。`HOP_RECHECK=false` 关闭全部复核。

## 链式退出登录与受信任主机

`/auth/sso/logout` 让用户在本应用退出登录，使会话失效，并重新生成 CSRF 令牌。如果 `?next=` 中是受信任的 URL，随后会重定向过去，hub 正是这样让用户从每个应用退出登录的。其他情况都会跳转到 `/`。

满足以下全部条件时，`next` URL 才是受信任的：

- 它的主机与某个受信任主机完全一致。不做通配符匹配：`hel.ink` 受信任，并不代表 `evil.hel.ink` 也受信任。
- 它不包含用户信息、空白字符、控制字符或反斜杠。
- 它使用 `https`。普通的 `http` 只对 `localhost`、`127.0.0.1` 和 `[::1]` 接受，而且仅当该主机在受信任列表中时。

受信任主机包括 `hop-auth.trusted_hosts`，加上 hub（`HOP_HUB_URL`）、本应用（`APP_URL`），以及共享的 `config/hop.php` 中的 URL（`hop.public_url`、`hop.hub_url` 和每个应用的 `url`）。

默认列表是生产环境的家族：`hel.ink`、`www.hel.ink`、`hub.hel.ink`、`link.hel.ink`、`bio.hel.ink`、`note.hel.ink`、`pixel.hel.ink`、`file.hel.ink` 和 `pay.hel.ink`。`HOP_TRUSTED_HOSTS` 会替换该列表。如果你在自己的域名上运行这个家族，请设置你的应用的主机：

```ini
HOP_TRUSTED_HOSTS=example.org,hub.example.org,link.example.org,bio.example.org
```

## 来自 hub 的账号事件

当某个账号发生变化时，hub 会通知每个声明了 `events` 能力的应用：发送 `POST {app}/hop/events`，请求体为 `{id, type, at, data}`。它用应用的密钥签名请求（HOP App Protocol v1：请求头 `X-HOP-App: helink`、`X-HOP-Timestamp`、`X-HOP-Nonce`、`X-HOP-Signature`），如果响应不是 2xx，会以退避方式重试约十六个小时。事件类型：

| 类型 | 发送时机 | 本包内置行为 |
| --- | --- | --- |
| `session.revoked` | 用户在 hub 退出登录，或管理员结束其会话 | 结束该账号在本应用的所有会话，并轮换其 `remember_token`。 |
| `account.deleted` | 账号被删除 | 结束该账号在本应用的所有会话。应用自己的数据请用你自己的处理器删除。 |
| `account.status` | 状态变化（`data.status`：`active`、`suspended`） | 无。LinkHop 和 BioHop 已会在已停用账号的下一个请求时让其退出登录；如果你的应用需要更多处理，请添加处理器。 |
| `profile.updated` | 资料变化 | 无。 |

`data.sub` 是共享 `users` 表中的账号 id。其他任何类型都会得到 200 `{handled: false}`，这样 hub 就不会重试本应用不认识的事件。

为什么 remember 令牌很重要：本包的登录使用 `remember: true`，所以只删除用户的会话记录是不够的。remember Cookie（默认 400 天）会让用户直接重新登录。`SessionRevoker` 会删除会话（`database` 和 `file` 驱动；其他驱动无法按账号查找，结果中的 `complete` 为 `false`），并且总会轮换令牌。

两种接收方式：

1. 把 `routes.events` 设为 `/hop/events`。本包会注册该路由，并由 `hop.event` 保护。`hop.event` 用与登录票据相同的 `HOP_SSO_SECRET` 和 `HOP_SSO_SECRET_PREV` 校验签名，只接受 `events.max_age` 秒以内的时间戳，每个 nonce 只能用一次。所有拒绝都返回同样简单的 `401 {"ok":false,"error":"unauthorized"}`。
2. 如果应用已有一条自带签名校验的路由，就把它指向 `Hop\Auth\Http\Controllers\EventController`（一个普通的可调用控制器）。它检查信封，在缓存中记住每个事件 id（重试会得到 `{duplicate: true}`，不会被处理两次），然后运行处理器。处理器抛出异常时返回不带细节的 500，并忘掉该 id，这样 hub 重试时会再次运行它。

在 service provider 中添加你自己的处理器：

```php
use Hop\Auth\HopEvents;

HopEvents::on('account.deleted', fn (array $data, array $event) => $this->purge((int) $data['sub']));
HopEvents::on('account.status', function (array $data) {
    // return false when the event is not about anyone you know
});
HopEvents::forget('session.revoked'); // drop the built-in handler
```

事件 id 需要能在请求之间保留的缓存存储（`database`、`file`、`redis`），不能用 `array`。hub 中必须有该应用的 `apps` 记录并带有 `events` 能力，否则 hub 不会发送任何事件。

## 测试

`tests/Feature` 中的测试是 Pest 测试，需要一个带有共享 `users` 表的 Laravel 应用，所以它们在使用本包的应用中运行（HEL.ink 在 `packages/auth` 中带有一份副本）。本仓库自己的 CI 会检查 `composer.json` 是否有效、每个 PHP 文件能否在 PHP 8.3 下通过语法检查，以及下面的家族表是否与标准版本一致。

## 安全

请按照 [SECURITY.md](SECURITY.md) 中的说明报告漏洞，不要提交公开 issue。

## 更新日志

见 [CHANGELOG.md](CHANGELOG.md)。

## 自行托管

运行 HOP 时不需要手动安装本包。LinkHop 和 BioHop 在各自的 `composer.json` 中列出了这个公开仓库，所以 `composer install` 无需令牌即可获取它，而 hub（[HEL.ink](https://github.com/navi-crwn/HEL.ink)）在 `packages/auth` 中带有一份副本。使用本包的应用需要一个在同一数据库上运行的 hub，以及以下设置：`HOP_HUB_URL`、`HOP_SSO_SECRET`（hub 的密钥）、`HOP_APP`，在你自己的域名上还需要 `HOP_TRUSTED_HOSTS`，因为默认列表只包含 hel.ink 的主机。管理面板 Imperium 是可选的。其余内容见家族指南：[docs/self-hosting.md](https://github.com/navi-crwn/HEL.ink/blob/main/docs/self-hosting.md)。

## HOP 家族

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

## 许可证

HOP 共享包是采用 [MIT 许可证](LICENSE)的开源软件。
