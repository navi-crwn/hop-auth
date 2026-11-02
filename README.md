# hop/auth

Single Sign-On (SSO) client, ticket verification, and request signing package for the **HOP** family of applications behind [hel.ink](https://hel.ink).

`hop/auth` enables any Laravel application in the ecosystem to authenticate members against the centralized HOP ID hub (`hub.hel.ink`), verify short-lived HMAC tickets, validate scoped API tokens, handle chained logouts, and verify inter-service server-to-server HMAC signatures.

---

## Features

- **SSO Ticket Verification**: Validates short-lived tickets issued by HOP ID (`/auth/sso/callback`).
- **Dual-Secret Rotation**: Seamlessly transitions secrets (`HOP_SSO_SECRET` and `HOP_SSO_SECRET_PREV`) with zero user-facing downtime during secret rotations.
- **Scoped API Tokens**: Middleware `hop.token:{scope}` verifying tokens against the shared `api_tokens` table.
- **Chained Logout**: Cleanses local app session and forwards logout request to the hub logout endpoint.
- **Signed Inter-Service Calls**: Middleware `hop.signed` verifying `X-HOP-Signature`, `X-HOP-Nonce`, and anti-replay timestamp bounds.

---

## Installation

Within a HOP ecosystem app, add the repository reference in `composer.json`:

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

Then run:

```bash
composer update hop/auth
```

The `Hop\Auth\AuthServiceProvider` will auto-register with Laravel package discovery.

---

## Configuration

Add the following environment variables to your application's `.env`:

```ini
HOP_APP=linkhop
HOP_HUB_URL=https://hub.hel.ink
HOP_SSO_SECRET=current_secret_key_minimum_32_characters
HOP_SSO_SECRET_PREV=previous_secret_key_during_rotation_window
```

### Registered Middleware Aliases

- `hop.auth`: Ensures the user is authenticated via HOP SSO; redirects unauthenticated visitors to `HOP_HUB_URL/auth/sso`.
- `hop.token:{scope}`: Verifies bearer API tokens for specified scopes.
- `hop.signed`: Verifies server-to-server signed requests (HMAC-SHA256).

### API token rate limit

`hop.token` allows each token `rate_limit` requests per hour (counted atomically in the default cache store). A token with no usable `rate_limit` (empty or not above zero) gets `hop-auth.api_rate_limit` (default 100). A refused request gets a `429` with `retry_after` (seconds until the window ends) and a matching `Retry-After` header.

### Chained logout and trusted hosts

`/auth/sso/logout?next=` only follows an `https` URL whose host is listed in `hop-auth.trusted_hosts`, plus the hub (`HOP_HUB_URL`), this app (`APP_URL`) and the app URLs in `config/hop.php`. There is no wildcard matching: `evil.hel.ink` is not trusted just because it is under `hel.ink`. Plain `http` is accepted for `localhost`, `127.0.0.1` and `[::1]` only. Anything else falls back to `/`.

The default list is the production family: `hel.ink`, `www.hel.ink`, `hub.hel.ink`, `link.hel.ink`, `bio.hel.ink`, `note.hel.ink`, `pixel.hel.ink`. If you self-host the family on your own domain, set the hosts of your apps:

```ini
HOP_TRUSTED_HOSTS=example.org,hub.example.org,link.example.org,bio.example.org
```

---

## HOP Family Ecosystem

<!-- hop-family:start -->
<!-- Generated from HEL.ink resources/hop-assets/apps.source.json; edit there, then run scripts/hop_assets.py build. -->
| App | Address | Repository | What it does |
| --- | --- | --- | --- |
| **HOP ID** | [hel.ink](https://hel.ink) · accounts at `hub.hel.ink` | [navi-crwn/HEL.ink](https://github.com/navi-crwn/HEL.ink) | One account for every HOP app. |
| **LinkHop** | [link.hel.ink](https://link.hel.ink) · `hel.ink/l/…` | [navi-crwn/LinkHop](https://github.com/navi-crwn/LinkHop) | Short links, QR codes and click analytics. |
| **BioHop** | [bio.hel.ink](https://bio.hel.ink) · `hel.ink/b/…` | [navi-crwn/BioHop](https://github.com/navi-crwn/BioHop) | One link-in-bio page for everything you do. |
| **PixelHop** | [pixel.hel.ink](https://pixel.hel.ink) · `p.hel.ink/…` | [navi-crwn/PixelHop](https://github.com/navi-crwn/PixelHop) | Image hosting with albums and direct links. |
| **NoteHop** | [note.hel.ink](https://note.hel.ink) · `n.hel.ink/…` | [navi-crwn/NoteHop](https://github.com/navi-crwn/NoteHop) | Encrypted notes you share with a link. |
| **Imperium** | private (owner only) | private | The HOP control panel. |
<!-- hop-family:end -->

---

## License

The HOP shared packages are open-source software licensed under the [MIT license](LICENSE).
