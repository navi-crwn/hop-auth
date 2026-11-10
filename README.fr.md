# hop/auth

[English](README.md) · [简体中文](README.zh-CN.md) · [Deutsch](README.de.md) · [Bahasa Indonesia](README.id.md) · [日本語](README.ja.md) · [Português (Brasil)](README.pt-BR.md) · [Español](README.es.md) · **Français** · [Русский](README.ru.md)

Le client Laravel pour HOP ID, le service de comptes de la famille HOP ([hel.ink](https://hel.ink)). Il permet à une app de la famille de connecter les personnes via le hub à `hub.hel.ink`, de vérifier le ticket de connexion que le hub renvoie, d'accepter des jetons d'API avec des scopes et de déconnecter à nouveau les personnes.

Le hub lui-même n'utilise que le modèle `ApiToken` et les alias de middleware. Les routes SSO sont destinées aux apps qui consomment HOP ID, comme LinkHop et BioHop.

## Ce que fournit le paquet

- Trois routes pour l'aller-retour de connexion : start, callback et logout.
- `Hop\Auth\Ticket`, qui vérifie un ticket signé avec HMAC-SHA256. Il contrôle la signature, l'audience (cette app), l'expiration et le `state` envoyé par l'app. Pendant une rotation, l'ancien secret est aussi accepté.
- Le middleware `hop.auth` : comme `auth` de Laravel, mais un invité passe par le hub au lieu d'aller vers une page de connexion locale.
- Le middleware `hop.token` : des jetons d'API bearer issus de la table partagée `api_tokens`, avec scopes, expiration et une limite de requêtes par heure pour chaque jeton.
- Une déconnexion en chaîne qui ne suit que les liens vers des hôtes de confiance.
- Un récepteur pour les événements de compte du hub (`POST /hop/events`) : contrôle de signature, ids d'événement à usage unique et hooks pour l'app. Une déconnexion au hub, un admin qui met fin aux sessions, une suspension ou une suppression parviennent ainsi à l'app, même si l'app était arrêtée au moment où cela s'est produit.

## Prérequis

- PHP 8.3 et Laravel 12 (`illuminate/support` ^12.0).
- La table partagée `users` de la famille. Le paquet connecte l'utilisateur dont l'id est le `sub` du ticket.
- Pour les jetons d'API, la table partagée `api_tokens`. Le paquet ne fournit pas de migration pour cette table ; les jetons sont créés et révoqués au hub.
- Un stockage de session et un stockage de cache. Le cache conserve le marqueur à usage unique de chaque ticket et les compteurs de limite de requêtes.

## Installation

Le paquet n'est pas sur Packagist. Ajoutez le dépôt au `composer.json` de l'app :

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

Puis exécutez :

```bash
composer update hop/auth
```

La découverte de paquets de Laravel enregistre `Hop\Auth\AuthServiceProvider`. Le provider ne charge pas les routes SSO quand `config('hop.current')` vaut `hub` : le hub émet des tickets et n'en consomme jamais.

## Configuration

Définissez ces valeurs dans le `.env` de l'app. En général, seuls `HOP_APP` et `HOP_SSO_SECRET` sont nécessaires :

```ini
HOP_APP=linkhop
HOP_HUB_URL=https://hub.hel.ink
HOP_SSO_SECRET=at-least-32-characters-shared-with-the-hub
# HOP_SSO_SECRET_PREV=only-while-rotating
# HOP_USER_MODEL=App\Models\User
# HOP_TRUSTED_HOSTS=only-for-a-self-hosted-family
```

| Variable | Défaut | Signification |
| --- | --- | --- |
| `HOP_APP` | `linkhop` | Le slug de cette app dans le `config/hop.php` du hub. C'est aussi l'audience du ticket. |
| `HOP_HUB_URL` | `https://hub.hel.ink` | Le hub qui émet les tickets. |
| `HOP_SSO_SECRET` | aucun | Le secret HMAC partagé. Les secrets de moins de 32 caractères sont ignorés, et sans secret utilisable, chaque ticket est refusé avec `not_configured`. |
| `HOP_SSO_SECRET_PREV` | aucun | L'ancien secret, accepté uniquement pendant la rotation. |
| `HOP_USER_MODEL` | `App\Models\User` | Le modèle connecté, et le propriétaire d'un jeton d'API. |
| `HOP_TRUSTED_HOSTS` | les hôtes de la famille en production | Les hôtes vers lesquels la déconnexion en chaîne peut rediriger. Voir plus bas. |
| `HOP_RECHECK` | `true` | Vérification silencieuse du compte au chargement des pages (`false` la désactive). Voir plus bas. |
| `HOP_RECHECK_MAX_AGE` | `300` | Secondes pendant lesquelles une session reste récente après la dernière validation par le hub. `0` désactive la vérification. |

Les autres options n'ont pas de variable d'environnement. Publiez le fichier de configuration pour les modifier :

```bash
php artisan vendor:publish --tag=hop-auth-config
```

| Clé | Défaut | Signification |
| --- | --- | --- |
| `leeway` | `30` | Secondes de décalage d'horloge tolérées lors du contrôle de `exp` et `iat` d'un ticket. |
| `home` | `/dashboard` | Où arriver après la connexion quand l'app n'a pas demandé de page. |
| `api_rate_limit` | `100` | Requêtes par heure pour un jeton d'API qui n'a pas sa propre limite. |
| `events.hub_app`, `events.max_age`, `events.seen_ttl` | `helink`, `120`, `172800` | Événements de compte : le slug d'app avec lequel le hub signe, les secondes d'écart d'horloge tolérées (chaque nonce sert une fois dans le double de cette durée) et la durée pendant laquelle un id d'événement est mémorisé. |
| `routes.events` | `null` | Chemin du récepteur d'événements de compte. Désactivé par défaut ; voir plus bas. |
| `routes.start`, `routes.callback`, `routes.logout` | `/auth/sso`, `/auth/sso/callback`, `/auth/sso/logout` | Les chemins des routes. Mettez `routes.start` à `null` pour ignorer les routes du paquet et enregistrer les vôtres. |

## Déroulement d'une connexion

1. Un invité ouvre une page protégée par `hop.auth`. Le middleware l'envoie vers `/auth/sso?next=<page>`. Une requête JSON reçoit un `401` à la place.
2. `start` stocke un `state` aléatoire et la page `next` dans la session, puis redirige vers `HOP_HUB_URL/sso/{HOP_APP}?state=…&return_to=<callback url>`. Seul un chemin du même site est conservé comme `next`.
3. Le hub connecte la personne et redirige vers `/auth/sso/callback?ticket=…`.
4. `callback` vérifie le ticket et le state. Il refuse un ticket déjà utilisé (le `jti` est mémorisé pendant 5 minutes) et un compte suspendu (`status` égal à `suspended`).
5. Il connecte l'utilisateur dont l'id est `sub`, régénère la session, garde le contenu du ticket dans la session sous `hop_profile` et redirige vers `next` ou vers `home`.

Un ticket est formé de `base64url(json)` et `base64url(HMAC-SHA256)` joints par un point. Le JSON doit contenir `sub`, `exp` et `state`. `aud`, `iat`, `jti` et `status` sont facultatifs. `Ticket::verify()` lève `Hop\Auth\TicketException` avec l'une de ces raisons : `not_configured`, `malformed`, `bad_signature`, `expired`, `wrong_audience`, `state_mismatch`.

### Rotation du secret

Mettez le nouveau secret dans `HOP_SSO_SECRET` et l'ancien dans `HOP_SSO_SECRET_PREV`. Un ticket signé avec l'un ou l'autre est accepté. Retirez l'ancienne valeur dès que le hub ne signe plus qu'avec le nouveau.

## Protéger des routes

```php
Route::middleware('hop.auth')->group(function () {
    Route::get('/dashboard', DashboardController::class);
});

Route::middleware('hop.token:link:create')->post('/api/links', StoreLinkController::class);
```

### Jetons d'API

Un client envoie `Authorization: Bearer hlk_…`. Le paquet ne stocke que le hash SHA-256 d'un jeton. `hop.token` répond avec un JSON de la forme `{"success": false, "error": "…", "message": "…"}` et ces statuts :

| Statut | Cause |
| --- | --- |
| 401 | En-tête absent ou mal formé, jeton inconnu ou jeton expiré. |
| 403 | Le compte est suspendu, ou il manque au jeton le scope requis. |
| 429 | La limite horaire du jeton est épuisée. |

En cas de succès, la requête porte les attributs `api_user` et `api_token`, et `$request->user()` renvoie le propriétaire du jeton.

`hop.token` sans scope vérifie seulement que le jeton est valide. `hop.token:link:create` exige en plus le scope `link:create`. Un jeton sans scopes (null ou liste vide) peut tout faire. Sinon, `can()` accepte une correspondance exacte, un joker de préfixe comme `link:*`, ou `*`. Les scopes définis par la famille sont dans `ApiToken::SCOPES` : `link:read`, `link:create`, `link:write`, `bio:read`, `bio:write`, `pixel:read` et `pixel:write`.

### Limite de requêtes des jetons d'API

Chaque jeton peut faire `rate_limit` requêtes par heure. Un jeton dont le `rate_limit` est vide ou n'est pas supérieur à zéro reçoit `hop-auth.api_rate_limit` (100 par défaut). La fenêtre commence à la première requête du jeton et se termine une heure plus tard. Le compteur vit dans le stockage de cache du rate limiter et est incrémenté avant d'être comparé à la limite, donc des requêtes parallèles ne peuvent pas toutes passer en même temps.

Une requête refusée reçoit un `429` avec `retry_after` dans le corps et un en-tête `Retry-After` correspondant, tous deux en secondes jusqu'à la fin de la fenêtre.

## Vérification silencieuse du compte

Une session de votre app peut survivre au compte que la personne a sur le hub : si elle se connecte avec un autre compte, ou se déconnecte, sur le hub, votre app garde l'ancien compte. `hop.auth` comble cette faille sans migration :

- La session est **récente** pendant `recheck.max_age` secondes (300 par défaut) après la dernière validation par le hub, par une connexion ou par une réponse silencieuse. L'heure est conservée dans la session.
- Si elle n'est plus récente, le chargement d'une page (un `GET` qui n'est ni XHR, ni JSON, ni un préchargement) fait un seul aller-retour `prompt=none` vers le hub ([SSO silencieux](https://github.com/navi-crwn/HEL.ink/blob/main/docs/silent-sso.md)). Même compte : la page se charge comme si de rien n'était. Autre compte : la session bascule dessus. `login_required` : la personne est déconnectée ici et voit la page en invitée. Une page protégée lance alors la connexion normale ; une page publique n'affiche aucune connexion.
- Garde-fous : un POST ou une requête XHR n'est jamais redirigé. Rien n'est redirigé tant que le hub n'accepte pas de connexion (une courte sonde, mise en cache 30 secondes). Au plus une tentative toutes les `recheck.cooldown` secondes (30), donc pas de boucle. Toute autre erreur du hub, comme `temporarily_unavailable`, conserve la session et attend `recheck.retry_after` secondes (300). Le `state` doit être celui que la session a lancé.
- Les pages sans connexion requise qui affichent le compte (une page d'accueil avec un bouton « Tableau de bord ») reçoivent `hop.recheck` dans leur middleware, pour être vérifiées aussi.
- Les clés sont dans `hop-auth.recheck` (`enabled`, `max_age`, `cooldown`, `retry_after`, `probe`, `probe_timeout`). `HOP_RECHECK=false` désactive toute la vérification.

## Déconnexion en chaîne et hôtes de confiance

`/auth/sso/logout` déconnecte la personne ici, invalide la session et régénère le jeton CSRF. Si `?next=` contient une URL de confiance, il redirige ensuite vers elle. C'est ainsi que le hub déconnecte une personne de toutes les apps. Tout le reste va vers `/`.

Une URL `next` est de confiance quand toutes ces conditions sont remplies :

- Son hôte est exactement l'un des hôtes de confiance. Il n'y a pas de correspondance par joker : `evil.hel.ink` n'est pas de confiance du seul fait que `hel.ink` l'est.
- Elle ne contient ni informations utilisateur, ni espaces ou caractères de contrôle, ni barre oblique inverse.
- Elle utilise `https`. Le `http` simple n'est accepté que pour `localhost`, `127.0.0.1` et `[::1]`, et seulement si cet hôte figure dans la liste de confiance.

Les hôtes de confiance sont `hop-auth.trusted_hosts`, plus le hub (`HOP_HUB_URL`), cette app (`APP_URL`) et les URL du `config/hop.php` partagé (`hop.public_url`, `hop.hub_url` et l'`url` de chaque app).

La liste par défaut est la famille en production : `hel.ink`, `www.hel.ink`, `hub.hel.ink`, `link.hel.ink`, `bio.hel.ink`, `note.hel.ink`, `pixel.hel.ink`, `file.hel.ink` et `pay.hel.ink`. `HOP_TRUSTED_HOSTS` remplace cette liste. Si vous faites tourner la famille sur votre propre domaine, indiquez les hôtes de vos apps :

```ini
HOP_TRUSTED_HOSTS=example.org,hub.example.org,link.example.org,bio.example.org
```

## Événements de compte venant du hub

Le hub prévient chaque app qui déclare la capacité `events` quand quelque chose arrive à un compte, avec `POST {app}/hop/events` et un corps `{id, type, at, data}`. Il signe la requête avec le secret de l'app (HOP App Protocol v1 : en-têtes `X-HOP-App: helink`, `X-HOP-Timestamp`, `X-HOP-Nonce`, `X-HOP-Signature`) et réessaie avec backoff pendant environ seize heures quand la réponse n'est pas un 2xx. Les types :

| Type | Envoyé quand | Intégré ici |
| --- | --- | --- |
| `session.revoked` | la personne se déconnecte au hub, ou un admin met fin à ses sessions | Met fin ici à toutes les sessions de ce compte et fait tourner son `remember_token`. |
| `account.deleted` | le compte est supprimé | Met fin ici à toutes les sessions de ce compte. Supprimez les données propres à l'app avec votre propre handler. |
| `account.status` | le statut change (`data.status` : `active`, `suspended`) | Rien. LinkHop et BioHop déconnectent déjà un compte suspendu à sa requête suivante ; ajoutez un handler si votre app a besoin de plus. |
| `profile.updated` | le profil change | Rien. |

`data.sub` est l'id du compte dans la table partagée `users`. Tout autre type reçoit un 200 `{handled: false}`, pour que le hub ne réessaie jamais ce que cette app ne connaît pas.

Pourquoi le remember token compte : la connexion de ce paquet utilise `remember: true`, donc supprimer les lignes de session d'une personne ne suffit pas. Le cookie remember (400 jours par défaut) la reconnecterait aussitôt. `SessionRevoker` supprime les sessions (drivers `database` et `file` ; les autres drivers ne permettent pas de chercher par compte, et `complete` vaut `false` dans le résultat) et fait toujours tourner le jeton.

Deux façons de les recevoir :

1. Mettez `routes.events` à `/hop/events`. Le paquet enregistre la route derrière `hop.event`, qui vérifie la signature avec les mêmes `HOP_SSO_SECRET` et `HOP_SSO_SECRET_PREV` que les tickets de connexion, accepte un horodatage dans les `events.max_age` secondes et consomme chaque nonce une seule fois. Chaque refus est le même `401 {"ok":false,"error":"unauthorized"}`, sans autre détail.
2. Si l'app a déjà une route avec son propre contrôle de signature, faites-la pointer vers `Hop\Auth\Http\Controllers\EventController` (un simple contrôleur invocable). Il vérifie l'enveloppe, mémorise chaque id d'événement dans le cache (une nouvelle tentative reçoit `{duplicate: true}` et n'est pas traitée deux fois) et exécute les handlers. Un handler qui lève une exception donne un 500 sans détail et l'id est oublié, si bien que la nouvelle tentative du hub l'exécute à nouveau.

Ajoutez vos propres handlers dans un service provider :

```php
use Hop\Auth\HopEvents;

HopEvents::on('account.deleted', fn (array $data, array $event) => $this->purge((int) $data['sub']));
HopEvents::on('account.status', function (array $data) {
    // return false when the event is not about anyone you know
});
HopEvents::forget('session.revoked'); // drop the built-in handler
```

Les ids d'événement nécessitent un stockage de cache qui persiste entre les requêtes (`database`, `file`, `redis`), pas `array`. Le hub doit avoir une ligne `apps` pour l'app avec la capacité `events`, sinon il n'envoie rien.

## Tests

Les tests de `tests/Feature` sont des tests Pest qui ont besoin d'une app Laravel avec la table partagée `users` ; ils tournent donc dans une app qui utilise le paquet (HEL.ink en inclut une copie dans `packages/auth`). La CI propre à ce dépôt vérifie que `composer.json` est valide, que chaque fichier PHP passe le lint sous PHP 8.3 et que la table de la famille ci-dessous correspond à la table de référence.

## Sécurité

Signalez les vulnérabilités comme indiqué dans [SECURITY.md](SECURITY.md), pas dans des issues publiques.

## Journal des modifications

Voir [CHANGELOG.md](CHANGELOG.md).

## Auto-hébergement

Vous n'installez pas ce paquet à la main pour faire tourner HOP. LinkHop et BioHop référencent ce dépôt public dans leur `composer.json`, donc `composer install` le récupère sans jeton, et le hub ([HEL.ink](https://github.com/navi-crwn/HEL.ink)) en contient une copie dans `packages/auth`. Une app qui l'utilise a besoin d'un hub en marche sur la même base de données et de ces réglages : `HOP_HUB_URL`, `HOP_SSO_SECRET` (celui du hub), `HOP_APP` et, sur votre propre domaine, `HOP_TRUSTED_HOSTS`, car la liste par défaut ne contient que les hôtes hel.ink. Imperium, le panneau d'administration, est facultatif. Le guide de la famille couvre le reste : [docs/self-hosting.md](https://github.com/navi-crwn/HEL.ink/blob/main/docs/self-hosting.md).

## Famille HOP

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

## Licence

Les paquets partagés HOP sont des logiciels open source sous [licence MIT](LICENSE).
