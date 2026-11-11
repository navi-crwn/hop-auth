# hop/auth

[English](README.md) · [简体中文](README.zh-CN.md) · [Deutsch](README.de.md) · [Bahasa Indonesia](README.id.md) · [日本語](README.ja.md) · **Português (Brasil)** · [Español](README.es.md) · [Français](README.fr.md) · [Русский](README.ru.md)

O cliente Laravel para o HOP ID, o serviço de contas da família HOP ([hel.ink](https://hel.ink)). Ele permite que um app da família faça o login das pessoas pelo hub em `hub.hel.ink`, verifique o ticket de login que o hub devolve, aceite tokens de API com escopos e faça o logout das pessoas de novo.

O próprio hub usa apenas o model `ApiToken` e os aliases de middleware. As rotas de SSO são para os apps que consomem o HOP ID, como LinkHop e BioHop.

## O que o pacote oferece

- Três rotas para o ciclo de login: start, callback e logout.
- `Hop\Auth\Ticket`, que verifica um ticket assinado com HMAC-SHA256. Ele confere a assinatura, o público (este app), a expiração e o `state` que o app enviou. Durante uma rotação, o segredo anterior também é aceito.
- O middleware `hop.auth`: como o `auth` do Laravel, mas um visitante é enviado pelo hub em vez de ir para uma página de login local.
- O middleware `hop.token`: tokens de API bearer da tabela compartilhada `api_tokens`, com escopos, expiração e um limite de requisições por hora para cada token.
- Um logout encadeado que só segue links para hosts confiáveis.
- Um receptor para os eventos de conta do hub (`POST /hop/events`): verificação de assinatura, ids de evento de uso único e hooks para o app. Um logout no hub, um admin encerrando sessões, uma suspensão ou uma exclusão chegam ao app por esse caminho, mesmo que o app estivesse fora do ar quando aconteceu.

## Requisitos

- PHP 8.3 e Laravel 12 (`illuminate/support` ^12.0).
- A tabela compartilhada `users` da família. O pacote faz o login do usuário cujo id é o `sub` do ticket.
- Para tokens de API, a tabela compartilhada `api_tokens`. O pacote não traz uma migration para ela; os tokens são criados e revogados no hub.
- Um armazenamento de sessão e um de cache. O cache guarda a marca de uso único de cada ticket e os contadores do limite de requisições.

## Instalação

O pacote não está no Packagist. Adicione o repositório ao `composer.json` do app:

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

Depois rode:

```bash
composer update hop/auth
```

O package discovery do Laravel registra `Hop\Auth\AuthServiceProvider`. O provider não carrega as rotas de SSO quando `config('hop.current')` é `hub`: o hub emite tickets e nunca os consome.

## Configuração

Defina estes valores no `.env` do app. Normalmente só `HOP_APP` e `HOP_SSO_SECRET` são necessários:

```ini
HOP_APP=linkhop
HOP_HUB_URL=https://hub.hel.ink
HOP_SSO_SECRET=at-least-32-characters-shared-with-the-hub
# HOP_SSO_SECRET_PREV=only-while-rotating
# HOP_USER_MODEL=App\Models\User
# HOP_TRUSTED_HOSTS=only-for-a-self-hosted-family
```

| Variável | Padrão | Significado |
| --- | --- | --- |
| `HOP_APP` | `linkhop` | O slug deste app no `config/hop.php` do hub. Também é o público do ticket. |
| `HOP_HUB_URL` | `https://hub.hel.ink` | O hub que emite os tickets. |
| `HOP_SSO_SECRET` | nenhum | O segredo HMAC compartilhado. Segredos com menos de 32 caracteres são ignorados e, sem um segredo utilizável, todo ticket é recusado como `not_configured`. |
| `HOP_SSO_SECRET_PREV` | nenhum | O segredo anterior, aceito apenas enquanto você faz a rotação. |
| `HOP_USER_MODEL` | `App\Models\User` | O model que faz login e o dono de um token de API. |
| `HOP_TRUSTED_HOSTS` | os hosts da família em produção | Hosts para os quais o logout encadeado pode redirecionar. Veja abaixo. |
| `HOP_RECHECK` | `true` | Verificação silenciosa da conta ao carregar páginas (`false` desliga). Veja abaixo. |
| `HOP_RECHECK_MAX_AGE` | `300` | Segundos em que uma sessão conta como recente desde a última vez que o hub a validou. `0` desliga a verificação. |

As demais opções não têm variável de ambiente. Publique o arquivo de configuração para alterá-las:

```bash
php artisan vendor:publish --tag=hop-auth-config
```

| Chave | Padrão | Significado |
| --- | --- | --- |
| `leeway` | `30` | Segundos de diferença de relógio tolerados ao verificar `exp` e `iat` de um ticket. |
| `home` | `/dashboard` | Para onde ir após o login quando o app não pediu uma página. |
| `api_rate_limit` | `100` | Requisições por hora para um token de API que não tem limite próprio. |
| `events.hub_app`, `events.max_age`, `events.seen_ttl` | `helink`, `120`, `172800` | Eventos de conta: o slug de app com que o hub assina, os segundos de diferença de relógio tolerados (cada nonce vale uma vez dentro do dobro desse valor) e por quanto tempo um id de evento é lembrado. |
| `routes.events` | `null` | Caminho do receptor de eventos de conta. Desligado por padrão; veja abaixo. |
| `routes.start`, `routes.callback`, `routes.logout` | `/auth/sso`, `/auth/sso/callback`, `/auth/sso/logout` | Os caminhos das rotas. Defina `routes.start` como `null` para pular as rotas do pacote e registrar as suas. |

## Como funciona um login

1. Um visitante abre uma página protegida por `hop.auth`. O middleware o envia para `/auth/sso?next=<page>`. Uma requisição JSON recebe um `401` em vez disso.
2. `start` guarda um `state` aleatório e a página `next` na sessão e redireciona para `HOP_HUB_URL/sso/{HOP_APP}?state=…&return_to=<callback url>`. Apenas um caminho do mesmo site é mantido como `next`.
3. O hub faz o login da pessoa e redireciona para `/auth/sso/callback?ticket=…`.
4. `callback` verifica o ticket e o state. Ele recusa um ticket já usado (o `jti` é lembrado por 5 minutos) e uma conta suspensa (`status` igual a `suspended`).
5. Ele faz o login do usuário cujo id é `sub`, regenera a sessão, guarda o conteúdo do ticket na sessão como `hop_profile` e redireciona para `next` ou para `home`.

Um ticket é `base64url(json)` e `base64url(HMAC-SHA256)` unidos por um ponto. O JSON precisa de `sub`, `exp` e `state`. `aud`, `iat`, `jti` e `status` são opcionais. `Ticket::verify()` lança `Hop\Auth\TicketException` com um destes motivos: `not_configured`, `malformed`, `bad_signature`, `expired`, `wrong_audience`, `state_mismatch`.

### Rotação do segredo

Coloque o novo segredo em `HOP_SSO_SECRET` e o antigo em `HOP_SSO_SECRET_PREV`. Um ticket assinado com qualquer um dos dois é aceito. Remova o valor antigo quando o hub passar a assinar só com o novo.

## Protegendo rotas

```php
Route::middleware('hop.auth')->group(function () {
    Route::get('/dashboard', DashboardController::class);
});

Route::middleware('hop.token:link:create')->post('/api/links', StoreLinkController::class);
```

### Tokens de API

Um cliente envia `Authorization: Bearer hlk_…`. O pacote guarda apenas o hash SHA-256 de um token. `hop.token` responde com JSON no formato `{"success": false, "error": "…", "message": "…"}` e estes status:

| Status | Causa |
| --- | --- |
| 401 | Cabeçalho ausente ou malformado, token desconhecido ou token expirado. |
| 403 | A conta está suspensa, ou falta ao token o escopo necessário. |
| 429 | O limite por hora do token se esgotou. |

Em caso de sucesso, a requisição carrega os atributos `api_user` e `api_token`, e `$request->user()` retorna o dono do token.

`hop.token` sem escopo só verifica se o token é válido. `hop.token:link:create` também exige o escopo `link:create`. Um token sem escopos (null ou uma lista vazia) pode fazer tudo. Caso contrário, `can()` aceita uma correspondência exata, um curinga de prefixo como `link:*`, ou `*`. Os escopos que a família define estão em `ApiToken::SCOPES`: `link:read`, `link:create`, `link:write`, `bio:read`, `bio:write`, `pixel:read` e `pixel:write`.

### Limite de requisições dos tokens de API

Cada token pode fazer `rate_limit` requisições por hora. Um token cujo `rate_limit` está vazio ou não é maior que zero recebe `hop-auth.api_rate_limit` (100 por padrão). A janela começa na primeira requisição do token e termina uma hora depois. O contador fica no armazenamento de cache do rate limiter e é incrementado antes de ser comparado com o limite, então requisições paralelas não conseguem passar todas juntas.

Uma requisição recusada recebe um `429` com `retry_after` no corpo e um cabeçalho `Retry-After` correspondente, ambos em segundos até o fim da janela.

## Verificação silenciosa da conta

Uma sessão no seu app pode durar mais que a conta que a pessoa tem no hub: ela entra no hub com outra conta, ou sai, e o seu app continua com a conta antiga. O `hop.auth` fecha essa brecha sem migração:

- A sessão é **recente** por `recheck.max_age` segundos (padrão 300) desde a última validação do hub, por um login ou por uma resposta silenciosa. A hora fica guardada na sessão.
- Quando não é recente, o carregamento de uma página (um `GET` que não seja XHR, nem JSON, nem pré-carregamento) faz uma única ida ao hub com `prompt=none` ([SSO silencioso](https://github.com/navi-crwn/HEL.ink/blob/main/docs/silent-sso.md)). Mesma conta: a página carrega como se nada tivesse acontecido. Outra conta: a sessão troca para ela. `login_required`: a pessoa é desconectada aqui e vê a página como visitante. Uma página protegida então inicia o login normal; uma página pública não mostra login algum.
- Salvaguardas: um POST ou uma requisição XHR nunca é redirecionado. Enquanto o hub não responder 2xx na sua URL de saúde (`/up`), por exemplo com 502 ou tempo esgotado, nada é redirecionado e ninguém é desconectado: a sessão continua confiável e o próximo carregamento de página após expirar o veredito em cache (60 segundos) tenta de novo. No máximo uma tentativa a cada `recheck.cooldown` segundos (30), então não há laço. Qualquer outro erro do hub, como `temporarily_unavailable`, mantém a sessão e espera `recheck.retry_after` segundos (300). O `state` precisa ser o que a sessão iniciou.
- Páginas que não exigem login mas mostram a conta (uma página inicial com um botão "Painel") recebem `hop.recheck` no middleware, para serem verificadas também.
- As chaves estão em `hop-auth.recheck` (`enabled`, `max_age`, `cooldown`, `retry_after`, `probe`, `probe_path`, `probe_timeout`, `probe_ttl`). `HOP_RECHECK=false` desliga toda a verificação.

## Logout encadeado e hosts confiáveis

`/auth/sso/logout` faz o logout da pessoa aqui, invalida a sessão e regenera o token CSRF. Se `?next=` contém uma URL confiável, ele então redireciona para lá. É assim que o hub faz o logout de uma pessoa em todos os apps. Qualquer outra coisa vai para `/`.

Uma URL `next` é confiável quando tudo isto vale:

- O host dela é exatamente um dos hosts confiáveis. Não há correspondência por curinga: `evil.hel.ink` não é confiável só porque `hel.ink` é.
- Ela não tem informações de usuário, espaços em branco, caracteres de controle nem barra invertida.
- Ela usa `https`. O `http` simples só é aceito para `localhost`, `127.0.0.1` e `[::1]`, e só quando esse host está na lista de confiáveis.

Os hosts confiáveis são `hop-auth.trusted_hosts` mais o hub (`HOP_HUB_URL`), este app (`APP_URL`) e as URLs no `config/hop.php` compartilhado (`hop.public_url`, `hop.hub_url` e a `url` de cada app).

A lista padrão é a família em produção: `hel.ink`, `www.hel.ink`, `hub.hel.ink`, `link.hel.ink`, `bio.hel.ink`, `note.hel.ink`, `pixel.hel.ink`, `file.hel.ink` e `pay.hel.ink`. `HOP_TRUSTED_HOSTS` substitui essa lista. Se você roda a família no seu próprio domínio, defina os hosts dos seus apps:

```ini
HOP_TRUSTED_HOSTS=example.org,hub.example.org,link.example.org,bio.example.org
```

## Eventos de conta do hub

O hub avisa cada app que declara a capacidade `events` quando algo acontece com uma conta, com `POST {app}/hop/events` e um corpo `{id, type, at, data}`. Ele assina a requisição com o segredo do app (HOP App Protocol v1: cabeçalhos `X-HOP-App: helink`, `X-HOP-Timestamp`, `X-HOP-Nonce`, `X-HOP-Signature`) e tenta de novo com backoff por cerca de dezesseis horas quando a resposta não é um 2xx. Os tipos:

| Tipo | Enviado quando | Embutido aqui |
| --- | --- | --- |
| `session.revoked` | a pessoa faz logout no hub, ou um admin encerra as sessões dela | Encerra aqui todas as sessões dessa conta e rotaciona o `remember_token` dela. |
| `account.deleted` | a conta é excluída | Encerra aqui todas as sessões dessa conta. Exclua os dados próprios do app com um handler seu. |
| `account.status` | o status muda (`data.status`: `active`, `suspended`) | Nada. LinkHop e BioHop já fazem o logout de uma conta suspensa na próxima requisição dela; adicione um handler se o seu app precisar de mais. |
| `profile.updated` | o perfil muda | Nada. |

`data.sub` é o id da conta na tabela compartilhada `users`. Qualquer outro tipo recebe um 200 `{handled: false}`, para que o hub nunca tente de novo o que este app não conhece.

Por que o remember token importa: o login deste pacote usa `remember: true`, então apagar as linhas de sessão de uma pessoa não basta. O cookie remember (400 dias por padrão) faria o login dela de novo na hora. `SessionRevoker` apaga as sessões (drivers `database` e `file`; outros drivers não permitem busca por conta, e `complete` é `false` no resultado) e sempre rotaciona o token.

Duas formas de recebê-los:

1. Defina `routes.events` como `/hop/events`. O pacote registra a rota atrás de `hop.event`, que verifica a assinatura com os mesmos `HOP_SSO_SECRET` e `HOP_SSO_SECRET_PREV` dos tickets de login, aceita um timestamp dentro de `events.max_age` segundos e gasta cada nonce uma única vez. Toda recusa é o mesmo `401 {"ok":false,"error":"unauthorized"}`, sem mais detalhes.
2. Se o app já tem uma rota com a própria verificação de assinatura, aponte-a para `Hop\Auth\Http\Controllers\EventController` (um controller invocável simples). Ele verifica o envelope, lembra cada id de evento no cache (uma nova tentativa recebe `{duplicate: true}` e não é tratada duas vezes) e executa os handlers. Um handler que lança exceção gera um 500 sem detalhes e o id é esquecido, então a nova tentativa do hub o executa de novo.

Adicione seus próprios handlers em um service provider:

```php
use Hop\Auth\HopEvents;

HopEvents::on('account.deleted', fn (array $data, array $event) => $this->purge((int) $data['sub']));
HopEvents::on('account.status', function (array $data) {
    // return false when the event is not about anyone you know
});
HopEvents::forget('session.revoked'); // drop the built-in handler
```

Os ids de evento precisam de um armazenamento de cache que sobreviva entre requisições (`database`, `file`, `redis`), não `array`. O hub precisa ter uma linha em `apps` para o app com a capacidade `events`; caso contrário, não envia nada.

## Testes

Os testes em `tests/Feature` são testes Pest que precisam de um app Laravel com a tabela compartilhada `users`, então eles rodam dentro de um app que usa o pacote (o HEL.ink inclui uma cópia em `packages/auth`). A CI própria deste repositório verifica se `composer.json` é válido, se todo arquivo PHP passa no lint com PHP 8.3 e se a tabela da família abaixo bate com a canônica.

## Segurança

Relate vulnerabilidades como descrito em [SECURITY.md](SECURITY.md), não em issues públicas.

## Registro de alterações

Veja [CHANGELOG.md](CHANGELOG.md).

## Hospedagem própria

Você não instala este pacote manualmente para rodar o HOP. LinkHop e BioHop listam este repositório público no `composer.json` deles, então `composer install` o baixa sem token, e o hub ([HEL.ink](https://github.com/navi-crwn/HEL.ink)) traz uma cópia em `packages/auth`. Um app que o usa precisa de um hub em execução no mesmo banco de dados e destas configurações: `HOP_HUB_URL`, `HOP_SSO_SECRET` (o do hub), `HOP_APP` e, no seu próprio domínio, `HOP_TRUSTED_HOSTS`, porque a lista padrão só contém os hosts de hel.ink. O Imperium, o painel de administração, é opcional. O guia da família cobre o resto: [docs/self-hosting.md](https://github.com/navi-crwn/HEL.ink/blob/main/docs/self-hosting.md).

## Família HOP

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

## Licença

Os pacotes compartilhados do HOP são software de código aberto sob a [licença MIT](LICENSE).
