# Error codes

What each HTTP status means and how to fix it.

## At the proxy layer (HTTP/SOCKS5 proxy)

| Status | Meaning | Fix |
|---|---|---|
| `407 Proxy Authentication Required` | Bad API key / proxy password, OR malformed username tokens, OR plan doesn't include the requested mode (e.g. `coord_*` on a free plan) | Verify credentials at [/dashboard/api](https://nikaproxy.com/dashboard/api); check token spelling; verify your plan includes the feature |
| `502 Bad Gateway` | Upstream residential exit failed (dead IP, target rejected the TLS handshake, etc.) | Retry. If persistent on the same `sessionId`, the cached exit is dead — change session ID to rotate |
| `503 Service Unavailable` | Maintenance mode, OR coordinator is overloaded (TLS Ultra only) | Check [nikaproxy.com](https://nikaproxy.com) for maintenance banner; retry with backoff |
| `504 Gateway Timeout` | Upstream took longer than our 30s timeout | Retry. If consistent, the destination may be rate-limiting your exit IP — rotate |
| Connection refused (no HTTP response) | Wrong port, IP blocked, or DNS-level filter (Proton NetShield, NextDNS, AdGuard) eating `proxy.nikaproxy.com` | `nslookup proxy.nikaproxy.com` should return `85.209.231.225`. If NXDOMAIN, disable your DNS blocker |

## At the scrape API layer (`/api/scrape/fetch`)

The scrape API returns its OWN status code as the HTTP response. The destination's status code is in the `X-Nika-Status` header (or `statusCode` field of the JSON envelope when `envelope: true`).

| API status | Meaning | Fix |
|---|---|---|
| `200` | NikaProxy successfully reached the destination. The destination's actual response (any 2xx-5xx) is in the body / envelope | Inspect `X-Nika-Status` header or `statusCode` field |
| `400` | Malformed request — bad `url`, bad `profile` name, invalid `lifetimeMinutes`, etc. | Read the `error` field in the response body |
| `401` | Bad / missing `X-Api-Key`, banned account, or zero balance | Verify key at [/dashboard/api](https://nikaproxy.com/dashboard/api); check balance |
| `402` | Plan doesn't include the requested feature (e.g. `coord_*` profile on Free tier; `families` filter without TLS Ultra) | Upgrade at [/dashboard/plans](https://nikaproxy.com/dashboard/plans) |
| `403` | Account banned for policy violation | [Contact support](https://t.me/Nikaproxy_support) |
| `429` | You're sending requests faster than our rate limit for your plan | Slow down or upgrade |
| `502` | Upstream residential exit failed OR coord couldn't find a matching exit within the retry budget | Retry. For coord no-match: try a broader bucket (e.g. `coord` instead of `coord_win11_browser`) or remove the `families` filter |
| `503` | Maintenance mode OR coordinator briefly unavailable | Retry with exponential backoff. Check [nikaproxy.com](https://nikaproxy.com) for maintenance banner |
| `504` | Destination took longer than our 30s timeout | The destination is slow; not a NikaProxy issue. Retry, or set up the destination request to be more efficient |

## Common scrape-API error bodies

```json
{ "error": "invalid_profile", "detail": "profile not found: chrome_macos_v999" }
{ "error": "invalid_lifetime", "detail": "lifetimeMinutes must be 0, 5, 10, 15, 20, or 60" }
{ "error": "balance_exhausted", "detail": "account has 0 MB remaining" }
{ "error": "feature_locked", "detail": "coord_* requires TLS Ultra plan" }
{ "error": "uniqueness_exhausted", "detail": "all matching exits already held by this customer's other sticky sessions" }
{ "error": "coord_no_match", "detail": "no compatible exit after 10 attempts (families seen: android_stock, linux_generic)" }
```

## `coord_no_match` specifically

If you're requesting a rare device family (e.g. Win11 desktop) and the coordinator can't find one within the retry budget, you'll see this. Two common causes:

1. **Pool exhaustion** — at peak times the Win11 subset of the residential pool can be fully claimed by your own concurrent sticky sessions (see [sticky-sessions.md](sticky-sessions.md) "Per-customer uniqueness"). Reduce concurrency or wait for stickies to expire.
2. **Country misalignment** — some countries have very few Win11 exits. Try without `country` (worldwide) or pick a country with more residential desktop traffic (US, GB, DE, CA).

The error body's `detail` field tells you which family the coordinator actually saw — useful for diagnosing whether the issue is "no Win11 anywhere" vs "all Win11 already taken."

## When the destination returns a challenge page

If the destination is serving a Cloudflare / DataDome / Akamai challenge page (HTML with JS challenges, often HTTP 200 with body containing `cf-mitigated` or `__cf_chl_jschl_tk__`), that's NOT a NikaProxy error — your fingerprint passed enough to get past their pre-screen, but their post-screen flagged you anyway.

Things to try, in order:
1. Move to a higher tier (Residential → TLS Pro → TLS Ultra)
2. Try `coord_*` instead of a fixed profile — better cross-layer coherence
3. Add browser-realistic headers (Referer, Accept-Language, real Sec-Fetch-*)
4. Slow down your request rate (the destination may be rate-throttling)
5. For Cloudflare specifically, try `coord_win11_browser` with the country matching your User-Agent's geo

If consistently challenged on a specific target, [tell us](https://t.me/Nikaproxy_support) which target — we tune profiles against known anti-bot deployments.

## Connection-level errors (no HTTP response at all)

| Error from your client | Likely cause |
|---|---|
| `getaddrinfo failed` / NXDOMAIN | DNS blocker (Proton NetShield, NextDNS, AdGuard) is filtering `proxy.nikaproxy.com` or `nikaproxy.com`. Disable or whitelist |
| `connection refused` | Wrong port (HTTP is 8080, SOCKS5 is 1080), or your firewall is blocking outbound to those ports |
| `connection reset` | Cloudflare-level block on the website (`nikaproxy.com`), not the proxy. Try connecting via a different network |
| `Proxy CONNECT aborted` | Usually credentials issue at the proxy layer — see `407` row above |
| `TLS handshake failed` | If you're using TLS Pro / Ultra without installing NikaRootCA.crt, your client doesn't trust our MITM cert. See [certificate.md](certificate.md) |
