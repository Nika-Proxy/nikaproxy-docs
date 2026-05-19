# Pricing & bandwidth

How billing works, what counts as a billable byte, plan tiers, balance behaviour.

## What gets billed

**Bandwidth-based billing.** Each request bills the **actual upstream wire bytes** against your account balance.

Wire bytes include:
- Request line, request headers, request body
- TLS handshake (amortized share per request)
- HTTP/2 framing overhead (SETTINGS, HEADERS, DATA frames)
- Response status line, response headers, response body
- TCP/CONNECT framing to the residential gateway

In practice, a small GET request (~500 byte body) bills roughly 4–6 KB total because TLS + headers dominate at that size. A 50 KB response bills ~52–55 KB. Large responses are dominated by the body itself.

**Why wire bytes and not just response body:** the upstream residential provider bills US for the full wire bytes; aligning customer billing with what we pay keeps the accounting honest. The response body shown to your client is what your code uses; the wire bytes are what your card pays for.

Live `X-Nika-Bytes-Used` response header (and `bytesUsed` envelope field) report the billable count for each request.

## What doesn't get billed

- **Failed transport errors** — if the request never reached the destination's application layer (DNS fail, TCP reset before HTTP), no charge
- **Retry-on-transport-failure attempt 1** — when we retry due to a stale upstream connection, the first attempt's wasted bytes aren't billed
- **Probe traffic** — TLS Ultra prober runs are split 50/50: customer pays half, NikaProxy absorbs half. The customer half is included in your billable count above

## Plan tiers

| Plan | Includes | Use when |
|---|---|---|
| **Free / Pay-as-you-go** | Residential only | You only need IP rotation — your client already produces a believable TLS fingerprint |
| **TLS Pro** | + TLS Residential (`profile=` fixed) | Targets fingerprint the TLS layer; you pick the browser to spoof |
| **TLS Ultra** | + TLS Ultra (`coord_*` profiles) + `families` filter + per-customer sticky uniqueness | Targets cross-reference TCP × TLS; you need L4↔L7 coherence |

Subscription plans include a **monthly bandwidth allowance** that refreshes on renewal day. Top up with extra GB any time from [/dashboard/plans](https://nikaproxy.com/dashboard/plans) — pay-as-you-go GB **never expires** and rolls forward indefinitely.

## Balance behaviour

- **Live balance** is shown in the dashboard top bar at all times
- **MB-precision deduction** — the engine accumulates sub-MB charges in memory and flushes whole megabytes to the balance roughly once per MB used
- **Zero balance = traffic stops** — when your balance hits zero mid-flight, the in-flight request completes (you already paid for it) but the next request returns `401 balance_exhausted`
- **Top up before the wall** — set a balance threshold alert at [/dashboard/account/notifications](https://nikaproxy.com/dashboard/account/notifications) to get warned at 1 GB / 100 MB remaining

## Concurrent requests

No artificial concurrency cap on any plan. But there are practical limits:

- **Residential SDK exits have natural concurrency limits.** A single sticky session bound to a real Android phone can sustain maybe 5–10 parallel requests before the phone's TCP stack saturates. Heavy parallel fanout on ONE sticky session causes failures.
- **Spread load with rotating sessions.** If you need 100 parallel requests, use 100 different `sessionId` values (or omit `sessionId` entirely for rotating mode). The engine distributes across the residential pool.
- **TLS Ultra per-customer uniqueness** caps you at the size of the matching subset. 500 concurrent `coord_win11_browser` sticky sessions need 500 unique Win11 exits — see [sticky-sessions.md](sticky-sessions.md). If you exceed pool capacity, requests fail with `uniqueness_exhausted` rather than silently colliding.

## Pricing changes

We don't surprise-bill. Plan price changes, GB rates, or trial credit modifications are announced via:
- Banner on the dashboard
- Email to your account
- Telegram broadcast to subscribers

Bandwidth credit purchased at the old rate retains the old rate's GB amount.

## Refunds

Unused pay-as-you-go GB doesn't expire — you have no incentive to refund. If you genuinely don't want to use the service anymore and have substantial unused credit, [contact support](https://t.me/Nikaproxy_support) on a case-by-case basis.

## Currency

USD throughout. Crypto payment via [OxaPay](https://oxapay.com) supports BTC, ETH, USDT, USDC, LTC, TRX. Card payment via Stripe.
