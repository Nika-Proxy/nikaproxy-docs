# Proxy modes — Residential, TLS Pro, TLS Ultra

Three product tiers, each adding fingerprint protection at a different layer. Pick by what your target detects, not by which sounds most expensive.

## Residential

**What it is:** bare residential IP rotation. Your TLS handshake passes through to the destination unchanged.

**When to use:**
- Your client already produces a believable TLS fingerprint (real headless browser you control; well-known SDK that matches the User-Agent you advertise)
- The target doesn't fingerprint at the TLS layer (smaller sites, internal APIs, mobile-app endpoints that just check IP reputation)

**Plan tier:** included in Free / Pay-as-you-go.

**No CA install needed.** The proxy is a transparent CONNECT tunnel; your client terminates TLS directly with the destination.

**Activation:**
- Proxy URL: just don't add a profile token to the username
- Scrape API: omit `profile` from the request body

## TLS Pro (TLS Residential)

**What it is:** residential IP + a TLS fingerprint **you pick** from our profile catalog. Our engine MITMs your request, replaces the ClientHello with the profile's, optionally rewrites your User-Agent / Sec-CH-UA* / Accept-Language headers to match (Auto-Emulation), then forwards.

**When to use:**
- Targets that fingerprint at the TLS layer (Cloudflare, Akamai with JA3/JA4 enabled, most modern anti-bot vendors)
- You know which browser you want to look like (e.g. "Chrome 131 on Windows")
- Your client library can't produce a real-browser TLS handshake (Python `requests`, Go `net/http`, `curl` default)

**Plan tier:** TLS Pro subscription.

**CA install required.** Download `NikaRootCA.crt` from [/dashboard/certificate](https://nikaproxy.com/dashboard/certificate) and add it to your trust store, or pass it via `--cacert` / `verify=` per request.

**Profile name format:**

```
<client>_<os>_<variant>_<version>[_<hash>]

Examples:
  chrome_windows_normal_2508
  firefox_windows_private_2604
  safari_macos_normal_1801
  okhttp_android-13_normal_2005
  curl_windows_libressl_2604
```

Browse the full catalog via the [Generate](https://nikaproxy.com/dashboard/generate) page (visual picker with previews) or the [Testing console](https://nikaproxy.com/dashboard/testing).

**Activation:**
- Proxy URL: add the profile name as a token in the username (e.g. `-chrome_windows_normal_2508`)
- Scrape API: set `"profile": "chrome_windows_normal_2508"`

## TLS Ultra (coordinated)

**What it is:** the TLS profile is **chosen by our coordinator** to match the residential exit's TCP fingerprint. A Win11 desktop exit gets an Edge / Chrome / Firefox-Windows profile; an Android exit gets a mobile browser or OkHttp profile.

**Why this exists:** modern anti-bot stacks cross-check Layer 4 (TCP) and Layer 7 (TLS). If your TLS fingerprint says "Chrome on Windows" but your TCP fingerprint (visible from the exit IP's OS) says "Android," that mismatch is itself the bot signal. See our [tls-fingerprint-101](https://github.com/Nika-Proxy/tls-fingerprint-101) for the full explanation. TLS Ultra fixes the coherence by picking the L7 profile *based on what L4 looks like*.

**When to use:**
- High-value targets with multi-layer fingerprinting (Cloudflare Bot Management, Akamai Bot Manager, DataDome, PerimeterX/HUMAN, Kasada)
- You've tried TLS Pro and the target still flags you despite a correct JA4 — the next layer up is L4 coherence

**Plan tier:** TLS Ultra subscription.

**Coord tokens** (replace the profile token):

| Token | Meaning |
|---|---|
| `coord` | Auto — any compatible browser, any platform |
| `coord_win11` | Win11 desktop, either browser or library |
| `coord_win11_browser` | Win11 desktop · Chrome / Edge / Firefox |
| `coord_win11_library` | Win11 desktop · .NET / curl / Python / etc. |
| `coord_mobile` | Android, either browser or library |
| `coord_mobile_browser` | Android · Chrome / Samsung / Firefox Mobile |
| `coord_mobile_library` | Android · OkHttp / Cronet / Volley / etc. |

**First-request latency:** ~5–10 seconds while the coordinator probes candidate exits to find one matching the requested family. Subsequent requests on the same sticky session are sub-second (cached binding).

**Family filter** (optional): restrict which profile families coord may pick. Scrape-API only:

```json
{
  "profile":  "coord_mobile_browser",
  "families": ["chrome", "firefox"]
}
```

The above tells coord to pick only `chrome_android_*` or `firefox_android_*` profiles, never Samsung Internet, UCBrowser, etc.

## How to choose

```
Does your target fingerprint at TLS?     → no  → Residential
                                          → yes → continue
Do they cross-check TCP × TLS coherence? → no  → TLS Pro
                                          → yes → TLS Ultra
```

In practice: try Residential first (cheapest, fastest). If you see 403 / challenge pages, move to TLS Pro with a profile matching the User-Agent you advertise. If still blocked, move to TLS Ultra. Don't pay for layers you don't need.
