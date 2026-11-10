# Changelog

All notable changes to `hop/auth` are listed here. The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and versions are git tags.

## [Unreleased]

### Added

- `pay.hel.ink` (PayHop) in the default `trusted_hosts`, so the chained logout passes on to PayHop. A test checks that the default list holds every host of the family manifest (`tests/fixtures/hop-apps.json`).
- Silent account re-check. A session the hub has not vouched for in the last `hop-auth.recheck.max_age` seconds (default 300) is confirmed with one `prompt=none` round trip on the next page load (`GET`, not XHR or JSON, not a prefetch). Same account: nothing visible. Another account: the session switches to it. `login_required`: signed out here, landing on the page as a guest (no sign-in page for a public page). Safeguards: no redirect and no sign-out unless the hub health URL (`/up`) answers 2xx within `recheck.probe_timeout` (the verdict is cached `recheck.probe_ttl` seconds; a 502 or a timeout leaves the session trusted), at most one attempt per `recheck.cooldown` seconds, any other hub error keeps the session and waits `recheck.retry_after` seconds, and the `state` is checked. No migration: the timestamps live in the session. New `hop.recheck` middleware for public pages that show the account; `hop.auth` does the check itself. New config `recheck.*` and env `HOP_RECHECK`, `HOP_RECHECK_MAX_AGE`.

### Fixed

- A session in an app kept the old account after the person switched accounts at the hub, because `hop.auth` let every signed-in user through without asking the hub (the gap the re-check closes).

## [0.1.5] - 2026-11-10

### Added

- `file.hel.ink` (FileHop) in the default `trusted_hosts`, so the chained logout passes on to FileHop. A family on its own domain sets `HOP_TRUSTED_HOSTS` as before.
- Receiver for the hub's account events (`POST /hop/events`): the `hop.event` middleware (signature of the hub with the app's secret or previous secret, timestamp window, each nonce once, one bare 401 for every refusal), `EventController` (event id once only, 200 for a type the app does not know, 500 and a retry when a handler fails), the `HopEvents::on()` / `forget()` hooks, and `SessionRevoker`, which deletes the sessions of an account (database and file drivers) and rotates its `remember_token`. Built in: `session.revoked` and `account.deleted` end the sessions of that account. The route is off until `hop-auth.routes.events` is set; an app with its own route can use the controller directly. New config keys `events.hub_app`, `events.max_age`, `events.seen_ttl`, `routes.events`.

### Fixed

- The sign-in start, callback and chained-logout routes no longer answer 500 when `next` or `ticket` arrives as an array (`/auth/sso?next[]=x`, `/auth/sso/callback?ticket[]=x`, `/auth/sso/logout?next[]=x`). A non-string value counts as absent: start goes on without a `next`, the callback answers 400, the logout goes home.

### Changed

- Rewrote the README: removed the `hop.signed` middleware, which this package never contained, corrected the redirect target of `hop.auth`, and documented every config key, the sign-in round trip, the API token errors and scopes. Corrected the product page the same way.
- Added this changelog.

## [0.1.2] - 2026-11-02

### Security

- The chained logout (`/auth/sso/logout?next=`) no longer follows every host under the hub's parent domain. It follows only an exact, trusted host: `hop-auth.trusted_hosts` (`HOP_TRUSTED_HOSTS`), the hub, this app and the app URLs in `config/hop.php`. Wildcards are gone, so `evil.hel.ink` is not trusted because `hel.ink` is. URLs with whitespace or control characters are refused too, and `http` is accepted for localhost only.

### Fixed

- The API token rate limit counts atomically: the counter is incremented before it is compared, so parallel requests cannot slip under the limit.
- A token with no usable `rate_limit` (empty or not above zero) gets the configured default (`hop-auth.api_rate_limit`, 100) instead of being locked out.
- `retry_after` and the `Retry-After` header report the real seconds left in the window.

### Added

- `HOP_TRUSTED_HOSTS` and the `trusted_hosts` and `api_rate_limit` config keys.

## [0.1.1] - 2026-11-02

### Security

- The chained logout accepts only `https` URLs without user info or backslashes, and `safePath()` refuses control characters and backslashes in the `next` page of a sign-in.

Between 0.1.1 and 0.1.2 the repository also gained dual-secret rotation for ticket verification (`HOP_SSO_SECRET_PREV`), `SECURITY.md`, issue templates, a static checks workflow, the canonical HOP family table with a CI check, and the product page.

## [0.1.0] - 2026-10-28

### Added

- First version of the shared package: SSO ticket verification, the `hop.auth` and `hop.token` middleware aliases, the `ApiToken` model and the chained logout. This version was not tagged.
