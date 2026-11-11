# hop/auth

[English](README.md) · [简体中文](README.zh-CN.md) · [Deutsch](README.de.md) · [Bahasa Indonesia](README.id.md) · **日本語** · [Português (Brasil)](README.pt-BR.md) · [Español](README.es.md) · [Français](README.fr.md) · [Русский](README.ru.md)

HOP ID 用の Laravel クライアントです。HOP ID は HOP ファミリー（[hel.ink](https://hel.ink)）のアカウントサービスです。ファミリーのアプリはこのパッケージで、`hub.hel.ink` のハブを通じてユーザーをサインインさせ、ハブが返すサインインチケットを検証し、スコープ付きの API トークンを受け付け、ユーザーをサインアウトさせることができます。

ハブ自体が使うのは `ApiToken` モデルとミドルウェアのエイリアスだけです。SSO ルートは、LinkHop や BioHop など HOP ID を利用するアプリ向けです。

## 提供する機能

- サインインの往復に使う 3 つのルート: start、callback、logout。
- `Hop\Auth\Ticket`。HMAC-SHA256 で署名されたチケットを検証します。署名、オーディエンス（このアプリ）、有効期限、アプリが送った `state` を確認します。ローテーション中は以前のシークレットも受け付けます。
- `hop.auth` ミドルウェア: Laravel の `auth` と同様ですが、ゲストはローカルのログインページではなくハブへ送られます。
- `hop.token` ミドルウェア: 共有の `api_tokens` テーブルにある Bearer API トークンを扱います。スコープ、有効期限、トークンごとの 1 時間あたりのレート制限があります。
- 信頼できるホストへのリンクだけをたどる連鎖ログアウト。
- ハブのアカウントイベントの受信口（`POST /hop/events`）: 署名の確認、一度しか使えないイベント ID、アプリ用のフック。ハブでのサインアウト、管理者によるセッション終了、利用停止、削除はこの経路でアプリに届きます。発生時にアプリが停止していても届きます。

## 要件

- PHP 8.3 と Laravel 12（`illuminate/support` ^12.0）。
- ファミリー共有の `users` テーブル。パッケージは、ID がチケットの `sub` と一致するユーザーをサインインさせます。
- API トークンを使う場合は、共有の `api_tokens` テーブル。このテーブルのマイグレーションは同梱していません。トークンはハブで作成・失効させます。
- セッションストアとキャッシュストア。キャッシュには、各チケットの使い捨てマーカーとレート制限のカウンターが入ります。

## インストール

このパッケージは Packagist にはありません。アプリの `composer.json` にリポジトリを追加します。

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

次に実行します。

```bash
composer update hop/auth
```

Laravel のパッケージディスカバリーが `Hop\Auth\AuthServiceProvider` を登録します。`config('hop.current')` が `hub` のとき、このプロバイダーは SSO ルートを読み込みません。ハブはチケットを発行するだけで、受け取ることはないからです。

## 設定

アプリの `.env` に次を設定します。通常必要なのは `HOP_APP` と `HOP_SSO_SECRET` だけです。

```ini
HOP_APP=linkhop
HOP_HUB_URL=https://hub.hel.ink
HOP_SSO_SECRET=at-least-32-characters-shared-with-the-hub
# HOP_SSO_SECRET_PREV=only-while-rotating
# HOP_USER_MODEL=App\Models\User
# HOP_TRUSTED_HOSTS=only-for-a-self-hosted-family
```

| 変数 | デフォルト | 意味 |
| --- | --- | --- |
| `HOP_APP` | `linkhop` | ハブの `config/hop.php` におけるこのアプリのスラッグ。チケットのオーディエンスでもあります。 |
| `HOP_HUB_URL` | `https://hub.hel.ink` | チケットを発行するハブ。 |
| `HOP_SSO_SECRET` | なし | 共有の HMAC シークレット。32 文字未満のシークレットは無視され、使えるシークレットがない場合はすべてのチケットが `not_configured` として拒否されます。 |
| `HOP_SSO_SECRET_PREV` | なし | 以前のシークレット。ローテーション中だけ受け付けます。 |
| `HOP_USER_MODEL` | `App\Models\User` | サインインさせるモデル。API トークンの所有者でもあります。 |
| `HOP_TRUSTED_HOSTS` | 本番環境のファミリーのホスト | 連鎖ログアウトがリダイレクトしてよいホスト。下記を参照してください。 |
| `HOP_RECHECK` | `true` | ページ読み込み時のサイレントなアカウント再確認(`false` で無効)。下記を参照してください。 |
| `HOP_RECHECK_MAX_AGE` | `300` | ハブが最後に保証してから、セッションを新しいとみなす秒数。`0` で再確認を無効にします。 |

残りのオプションには環境変数がありません。変更するには設定ファイルを公開します。

```bash
php artisan vendor:publish --tag=hop-auth-config
```

| キー | デフォルト | 意味 |
| --- | --- | --- |
| `leeway` | `30` | チケットの `exp` と `iat` を確認するときに許容する時計のずれ（秒）。 |
| `home` | `/dashboard` | アプリがページを指定しなかった場合の、サインイン後の移動先。 |
| `api_rate_limit` | `100` | 独自の制限を持たない API トークンの、1 時間あたりのリクエスト数。 |
| `events.hub_app`, `events.max_age`, `events.seen_ttl` | `helink`, `120`, `172800` | アカウントイベント: ハブが署名に使うアプリのスラッグ、許容する時計の差（秒。各 nonce はこの 2 倍の時間内で一度だけ使えます）、イベント ID を記憶する期間。 |
| `routes.events` | `null` | アカウントイベント受信口のパス。デフォルトでは無効です。下記を参照してください。 |
| `routes.start`, `routes.callback`, `routes.logout` | `/auth/sso`, `/auth/sso/callback`, `/auth/sso/logout` | ルートのパス。`routes.start` を `null` にすると、パッケージのルートを使わず独自のルートを登録できます。 |

## サインインの流れ

1. ゲストが `hop.auth` で保護されたページを開きます。ミドルウェアはゲストを `/auth/sso?next=<page>` へ送ります。JSON リクエストには代わりに `401` を返します。
2. `start` はランダムな `state` と `next` のページをセッションに保存し、`HOP_HUB_URL/sso/{HOP_APP}?state=…&return_to=<callback url>` へリダイレクトします。`next` として保持されるのは同一サイトのパスだけです。
3. ハブがユーザーをサインインさせ、`/auth/sso/callback?ticket=…` へリダイレクトします。
4. `callback` はチケットと state を検証します。すでに使われたチケット（`jti` は 5 分間記憶されます）と、利用停止中のアカウント（`status` が `suspended`）は拒否します。
5. ID が `sub` のユーザーをサインインさせ、セッションを再生成し、チケットの内容を `hop_profile` としてセッションに保存して、`next` または `home` へリダイレクトします。

チケットは `base64url(json)` と `base64url(HMAC-SHA256)` をドットでつないだものです。JSON には `sub`、`exp`、`state` が必要です。`aud`、`iat`、`jti`、`status` は省略できます。`Ticket::verify()` は次のいずれかの理由で `Hop\Auth\TicketException` をスローします: `not_configured`, `malformed`, `bad_signature`, `expired`, `wrong_audience`, `state_mismatch`。

### シークレットのローテーション

新しいシークレットを `HOP_SSO_SECRET` に、古いシークレットを `HOP_SSO_SECRET_PREV` に設定します。どちらかで署名されたチケットが受け付けられます。ハブが新しいシークレットだけで署名するようになったら、古い値を削除してください。

## ルートの保護

```php
Route::middleware('hop.auth')->group(function () {
    Route::get('/dashboard', DashboardController::class);
});

Route::middleware('hop.token:link:create')->post('/api/links', StoreLinkController::class);
```

### API トークン

クライアントは `Authorization: Bearer hlk_…` を送ります。パッケージが保存するのはトークンの SHA-256 ハッシュだけです。`hop.token` は `{"success": false, "error": "…", "message": "…"}` という形の JSON と、次のステータスで応答します。

| ステータス | 原因 |
| --- | --- |
| 401 | ヘッダーがない、形式が不正、トークンが不明、またはトークンの有効期限切れ。 |
| 403 | アカウントが利用停止中、またはトークンに必要なスコープがない。 |
| 429 | トークンの 1 時間あたりの上限に達した。 |

成功すると、リクエストには `api_user` と `api_token` の属性が付き、`$request->user()` はトークンの所有者を返します。

スコープなしの `hop.token` は、トークンが有効かどうかだけを確認します。`hop.token:link:create` はさらに `link:create` スコープを必要とします。スコープのないトークン（null または空のリスト）は何でもできます。それ以外の場合、`can()` は完全一致、`link:*` のような前方一致のワイルドカード、または `*` を受け付けます。ファミリーが定義するスコープは `ApiToken::SCOPES` にあります: `link:read`、`link:create`、`link:write`、`bio:read`、`bio:write`、`pixel:read`、`pixel:write`。

### API トークンのレート制限

各トークンは 1 時間に `rate_limit` 回までリクエストできます。`rate_limit` が空、または 0 以下のトークンには `hop-auth.api_rate_limit`（デフォルトは 100）が適用されます。ウィンドウはトークンの最初のリクエストで始まり、1 時間後に終わります。カウンターはレートリミッターのキャッシュストアにあり、上限と比較する前に加算されるので、並列リクエストがまとめて通過することはありません。

拒否されたリクエストには `429` が返り、本文に `retry_after`、対応する `Retry-After` ヘッダーが付きます。どちらもウィンドウが終わるまでの秒数です。

## サイレントなアカウント再確認

アプリのセッションは、ハブでその人が使っているアカウントより長く残ることがあります。ハブで別のアカウントとしてサインインしたりサインアウトしたりしても、アプリは古いアカウントのままです。`hop.auth` はマイグレーションなしでこの穴を塞ぎます。

- セッションは、ハブが最後に保証してから `recheck.max_age` 秒(既定 300)のあいだ**新しい**とみなされます。保証はサインインまたはサイレントな応答で行われ、時刻はセッションに保存されます。
- 新しくない場合、ページの読み込み(XHR でも JSON でもプリフェッチでもない `GET`)で、ハブへ `prompt=none` の往復を 1 回だけ行います([サイレント SSO](https://github.com/navi-crwn/HEL.ink/blob/main/docs/silent-sso.md))。同じアカウント: 何事もなかったようにページが読み込まれます。別のアカウント: セッションがそのアカウントに切り替わります。`login_required`: ここでサインアウトされ、ゲストとしてそのページが表示されます。保護されたページは通常のサインインを始め、公開ページにはサインイン画面は一切表示されません。
- 安全策: POST や XHR リクエストはリダイレクトされません。ハブがヘルス URL(`/up`)に 2xx で応答しない間(502 やタイムアウトなど)は、リダイレクトもサインアウトもしません。セッションは信頼されたままで、キャッシュした判定の期限(60 秒)が切れたあとの次のページ読み込みで再試行します。試行は `recheck.cooldown` 秒(30)に最大 1 回なので、ループしません。`temporarily_unavailable` など他のハブのエラーは、セッションを保ったまま `recheck.retry_after` 秒(300)待ちます。`state` はセッションが開始したものと一致する必要があります。
- サインイン不要でもアカウントを表示するページ(「ダッシュボード」ボタンのあるトップページなど)は、ミドルウェアに `hop.recheck` を付けると同様に確認されます。
- キーは `hop-auth.recheck`(`enabled`、`max_age`、`cooldown`、`retry_after`、`probe`、`probe_path`、`probe_timeout`、`probe_ttl`)にあります。`HOP_RECHECK=false` で確認全体を無効にできます。

## 連鎖ログアウトと信頼できるホスト

`/auth/sso/logout` はこのアプリでユーザーをサインアウトさせ、セッションを無効にし、CSRF トークンを再生成します。`?next=` に信頼できる URL があれば、続けてそこへリダイレクトします。ハブはこの仕組みでユーザーをすべてのアプリからサインアウトさせます。それ以外の場合は `/` へ移動します。

`next` の URL は、次のすべてを満たすときに信頼されます。

- ホストが信頼できるホストのいずれかと完全に一致する。ワイルドカードによる一致はありません: `hel.ink` が信頼されていても、`evil.hel.ink` は信頼されません。
- ユーザー情報、空白文字、制御文字、バックスラッシュを含まない。
- `https` を使う。素の `http` は `localhost`、`127.0.0.1`、`[::1]` に対してのみ、かつそのホストが信頼リストにある場合だけ受け付けます。

信頼できるホストは、`hop-auth.trusted_hosts` に加えて、ハブ（`HOP_HUB_URL`）、このアプリ（`APP_URL`）、共有の `config/hop.php` にある URL（`hop.public_url`、`hop.hub_url`、各アプリの `url`）です。

デフォルトのリストは本番環境のファミリーです: `hel.ink`、`www.hel.ink`、`hub.hel.ink`、`link.hel.ink`、`bio.hel.ink`、`note.hel.ink`、`pixel.hel.ink`、`file.hel.ink`、`pay.hel.ink`。`HOP_TRUSTED_HOSTS` はこのリストを置き換えます。自分のドメインでファミリーを運用する場合は、自分のアプリのホストを設定してください。

```ini
HOP_TRUSTED_HOSTS=example.org,hub.example.org,link.example.org,bio.example.org
```

## ハブからのアカウントイベント

アカウントに何かが起きると、ハブは `events` ケイパビリティを宣言したすべてのアプリに、`POST {app}/hop/events` と本文 `{id, type, at, data}` で通知します。リクエストはアプリのシークレットで署名され（HOP App Protocol v1: ヘッダー `X-HOP-App: helink`、`X-HOP-Timestamp`、`X-HOP-Nonce`、`X-HOP-Signature`）、応答が 2xx でない場合はバックオフしながら約 16 時間再試行します。種類は次のとおりです。

| 種類 | 送信されるとき | このパッケージの組み込み動作 |
| --- | --- | --- |
| `session.revoked` | ユーザーがハブでサインアウトした、または管理者がそのセッションを終了した | このアプリでそのアカウントのすべてのセッションを終了し、`remember_token` をローテーションします。 |
| `account.deleted` | アカウントが削除された | このアプリでそのアカウントのすべてのセッションを終了します。アプリ独自のデータは自作のハンドラーで削除してください。 |
| `account.status` | ステータスが変わった（`data.status`: `active`、`suspended`） | なし。LinkHop と BioHop は、利用停止中のアカウントを次のリクエストの時点ですでにサインアウトさせます。アプリでそれ以上が必要ならハンドラーを追加してください。 |
| `profile.updated` | プロフィールが変わった | なし。 |

`data.sub` は共有の `users` テーブルにおけるアカウント ID です。それ以外の種類には 200 `{handled: false}` を返すので、このアプリが知らないイベントをハブが再試行することはありません。

remember トークンが重要な理由: このパッケージのサインインは `remember: true` を使うため、ユーザーのセッション行を削除するだけでは足りません。remember Cookie（デフォルトで 400 日）によって、すぐにサインインし直してしまいます。`SessionRevoker` はセッションを削除し（`database` と `file` ドライバー。ほかのドライバーはアカウント単位で検索できず、結果の `complete` は `false` になります）、トークンを必ずローテーションします。

受信方法は 2 つあります。

1. `routes.events` を `/hop/events` に設定します。パッケージはこのルートを `hop.event` の保護下に登録します。`hop.event` はサインインチケットと同じ `HOP_SSO_SECRET` と `HOP_SSO_SECRET_PREV` で署名を確認し、`events.max_age` 秒以内のタイムスタンプを受け付け、各 nonce を一度だけ消費します。拒否はすべて同じ、詳細のない `401 {"ok":false,"error":"unauthorized"}` です。
2. 独自の署名確認を持つルートがすでにある場合は、そのルートを `Hop\Auth\Http\Controllers\EventController`（シンプルな invokable コントローラー）に向けます。このコントローラーはエンベロープを確認し、各イベント ID をキャッシュに記憶し（再試行には `{duplicate: true}` を返し、二重に処理しません）、ハンドラーを実行します。ハンドラーが例外をスローすると詳細なしの 500 を返し、ID は忘れられるので、ハブの再試行で再び実行されます。

独自のハンドラーはサービスプロバイダーで追加します。

```php
use Hop\Auth\HopEvents;

HopEvents::on('account.deleted', fn (array $data, array $event) => $this->purge((int) $data['sub']));
HopEvents::on('account.status', function (array $data) {
    // return false when the event is not about anyone you know
});
HopEvents::forget('session.revoked'); // drop the built-in handler
```

イベント ID には、リクエストをまたいで保持されるキャッシュストア（`database`、`file`、`redis`）が必要です。`array` は使えません。ハブには、`events` ケイパビリティを持つそのアプリの `apps` 行が必要です。ない場合、ハブは何も送りません。

## テスト

`tests/Feature` のテストは Pest のテストで、共有の `users` テーブルを持つ Laravel アプリが必要です。そのため、このパッケージを使うアプリの中で実行します（HEL.ink は `packages/auth` にコピーを含んでいます）。このリポジトリ自体の CI は、`composer.json` が有効であること、すべての PHP ファイルが PHP 8.3 で lint を通ること、下のファミリー表が正規の表と一致することを確認します。

## セキュリティ

脆弱性は公開の issue ではなく、[SECURITY.md](SECURITY.md) に書かれた方法で報告してください。

## 変更履歴

[CHANGELOG.md](CHANGELOG.md) を参照してください。

## セルフホスティング

HOP を動かすために、このパッケージを手動でインストールする必要はありません。LinkHop と BioHop は `composer.json` にこの公開リポジトリを記載しているので、`composer install` がトークンなしで取得します。ハブ（[HEL.ink](https://github.com/navi-crwn/HEL.ink)）は `packages/auth` にコピーを持っています。このパッケージを使うアプリには、同じデータベース上で動いているハブと、次の設定が必要です: `HOP_HUB_URL`、`HOP_SSO_SECRET`（ハブのもの）、`HOP_APP`、そして自分のドメインでは `HOP_TRUSTED_HOSTS`。デフォルトのリストには hel.ink のホストしか入っていないためです。管理パネルの Imperium は任意です。残りはファミリーのガイドで説明しています: [docs/self-hosting.md](https://github.com/navi-crwn/HEL.ink/blob/main/docs/self-hosting.md)。

## HOP ファミリー

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

## ライセンス

HOP の共有パッケージは、[MIT ライセンス](LICENSE)のもとで公開されているオープンソースソフトウェアです。
