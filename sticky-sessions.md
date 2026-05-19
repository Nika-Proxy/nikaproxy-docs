# Sticky sessions

A **sticky session** pins a sequence of requests to the same residential exit IP for a fixed window. Use the same `sessionId` token across requests to keep the same IP; change it (or omit it) to rotate.

## Why you'd want one

The destination ties state to your IP. Common cases:

- **Login + scrape protected page** — the session cookie is IP-scoped; rotating mid-session looks like a session hijack to the destination
- **Multi-step checkout flows** — payment processors that fingerprint by IP across the funnel
- **Anti-bot challenges** — Cloudflare / DataDome challenge → solve → verified cookie. If the IP changes after verification, the cookie's worthless
- **Anything with rate-limited per-IP windows** — staying on one IP for the duration of your sliding window is cheaper than burning fresh IPs

## Activation

### Proxy URL

Add `-NAME` to the username (where `NAME` is any alphanumeric ≤ 32 chars):

```
nka_KEY-us-myflow42-60:nkp_PWD
       └┬┘ └──┬───┘ └┬┘
   country  session  lifetime
```

### Scrape API

```json
{
  "url":             "https://example.com",
  "sessionId":       "myflow42",
  "lifetimeMinutes": 60
}
```

## Lifetime

How long the binding survives. The engine accepts these values only:

| Value | Window |
|---|---|
| `0` | "Keep IP" — up to ~24 hours (subject to the residential SDK's own churn — see below) |
| `5` | 5 minutes |
| `10` | 10 minutes (default if omitted) |
| `15` | 15 minutes |
| `20` | 20 minutes |
| `60` | 60 minutes |

Other values are rejected at request time with `invalid_lifetime`. These specific values reflect the residential SDK's native rotation buckets — picking a custom interval doesn't make the IP last longer than the SDK guarantees.

## Session ID rules

- **Any alphanumeric**, ≤ 32 characters
- **Case-sensitive** — `MyFlow` and `myflow` are different sessions
- **Unique per customer** — your `myflow` doesn't collide with another customer's `myflow`; they're independent address spaces
- **Stable across plans** — sticky sessions work the same in Residential, TLS Pro, and TLS Ultra (TLS Ultra adds extra: see "Per-customer uniqueness" below)

## Per-customer uniqueness (TLS Ultra)

For TLS Ultra customers running multiple concurrent sticky sessions (e.g. 500 parallel scraping workers), the engine guarantees **per-customer IP uniqueness**: no two of your active sticky sessions will be bound to the same exit IP at the same time.

Other customers can still land on the same IP — the uniqueness is scoped per-customer, not global. In practice:
- Your `myflow-1` and `myflow-2` get different exits, guaranteed
- A different customer's `their-flow-1` is unaffected — they can independently use the same IP

After your sticky window closes there's a brief settling period before the IP is eligible for your next sticky.

## Expected churn

Residential SDKs that power exit IPs come and go — the device the IP belongs to may go offline, lose its app session, or get a new IP from its ISP. Even **within** a sticky window, you may see some IP churn:

- **30–50% rotation per hour** is typical for residential pools (varies by country, time of day, device family)
- **First 30 seconds** are most stable; long-tail mid-session rotation is the same whether you asked for 5 min or 60 min stickiness

**Don't design your client to require infinite stickiness.** Add a retry layer:

```python
import time, requests

def get_with_retry(url, session_id, max_attempts=3):
    proxy = f"http://nka_KEY-us-{session_id}-60:nkp_PWD@proxy.nikaproxy.com:8080"
    for attempt in range(max_attempts):
        try:
            r = requests.get(url, proxies={"http": proxy, "https": proxy}, timeout=30)
            r.raise_for_status()
            return r
        except (requests.RequestException, requests.HTTPError) as e:
            if attempt == max_attempts - 1: raise
            time.sleep(2 ** attempt)   # 1s, 2s, 4s backoff
```

When the underlying SDK exit rotates, the next request on the same `sessionId` will land on a fresh IP automatically — the engine re-binds the sticky cache entry. Your session cookie at the destination is now scoped to a new IP, so you may need to re-login or re-solve the challenge. This is residential-proxy reality, not a NikaProxy quirk; every provider has the same shape.

## Cost of a fresh sticky session (TLS Ultra)

For TLS Ultra, the **first request on a new sticky session ID is slow** (~5–10 seconds) — the engine does extra matching work to align your request with an exit of the requested family (Win11 desktop, mobile browser, etc.). Subsequent requests on the same session ID are sub-second.

For Residential and TLS Pro, first-request latency is normal — no extra matching step.

**Implication:** don't generate a fresh `sessionId` for every request on TLS Ultra. Reuse session IDs aggressively across your worker pool; the engine handles per-customer uniqueness automatically.
