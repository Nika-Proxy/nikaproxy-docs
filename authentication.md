# Authentication

Two credentials, both at [/dashboard/api](https://nikaproxy.com/dashboard/api).

## The credentials

| Credential | Prefix | Used as |
|---|---|---|
| **API key** | `nka_` | Proxy username **and** `X-Api-Key` header on the REST API |
| **Proxy password** | `nkp_` | Proxy password only — the REST API doesn't use it |

Both are 32-character random strings. Both are stored hashed server-side (we can verify them but we can't tell you what they are if you lose them — you'd reset).

## Resetting

- Both rotate **independently**. Resetting the proxy password doesn't invalidate REST API sessions, and vice versa.
- Reset takes effect immediately for new requests. In-flight requests at the moment of reset complete normally — they're already authenticated.
- A reset **kills all sticky sessions** bound to the old key. Your customers (or your workers) will all land on fresh exits after a reset, even if they keep using the same sessionId.

## How each path uses them

### HTTP/SOCKS5 proxy

```
http://nka_KEY[-tokens]:nkp_PWD@proxy.nikaproxy.com:8080
```

The proxy authenticates via standard Basic auth. Username = your API key (plus any `-token` modifiers), password = your proxy password literal.

### REST scrape API

```
POST https://nikaproxy.com/api/scrape/fetch
X-Api-Key: nka_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

The REST API authenticates via the `X-Api-Key` header. It does NOT accept `Authorization: Bearer` — that header is used elsewhere (dashboard sessions) and we keep them distinct.

The proxy password isn't checked by the REST API.

## Why two separate credentials

Different threat models. The API key is your account identifier — it appears in proxy URLs that may end up in command histories, log files, shared scripts. The proxy password is a secondary factor that's needed to actually USE the proxy. An attacker who scrapes your API key from a log file but doesn't have the proxy password can't make billable requests; they can only make REST API calls (which is a smaller surface).

If your team works with proxy URLs (which are necessarily share-prone), keep the API key separate from secrets like database passwords in your config management.

## Telegram-only accounts

If you signed up via the [Telegram bot](https://t.me/NikaProxyBot) and never added an email, your account still has both credentials — the bot gives them to you on first generation. You can add an email later from the dashboard's Account page for password recovery; until you do, the bot session is your only login path.

## What we don't authenticate

- The destination URL. NikaProxy doesn't validate that you're authorised to scrape the target site — that's your responsibility under the [Terms of Service](https://nikaproxy.com/legal).
- The Profile picked. Free-plan customers attempting to use `coord_*` get a clean `402 feature_locked` error at request time, but we don't pre-validate every profile against every plan tier — the engine does it inline.

## Auth events

Every login, every credential reset, every billable API call shows up in [/dashboard/auth-log](https://nikaproxy.com/dashboard/auth-log) and is retained for 90 days. If you suspect compromised credentials, the auth log shows the IP each request came from. Resetting both credentials kills any active session and forces re-auth.
