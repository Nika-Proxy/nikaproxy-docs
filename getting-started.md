# Quick start

From "I don't have an account" to "my first request is succeeding" in about 5 minutes.

## 1. Sign up

[nikaproxy.com](https://nikaproxy.com) — email + password. 100 MB of free credit is added on signup, no card needed. You're not subscribed to anything; the credit lets you test before paying.

## 2. Grab credentials

Open the dashboard → **API & Credentials** ([direct link](https://nikaproxy.com/dashboard/api)).

Two strings to copy:

| Credential | Format | Used as |
|---|---|---|
| **API key** | `nka_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx` | Proxy username AND `X-Api-Key` header on the REST API |
| **Proxy password** | `nkp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx` | Proxy password only (REST API doesn't use it) |

Both rotate independently — resetting one doesn't invalidate the other.

## 3. Pick a path: HTTP proxy or REST API

Two ways to use NikaProxy. They share the same backend, the same exits, and the same TLS profiles — you choose the integration shape that fits your stack.

### Path A: HTTP/SOCKS5 proxy

Drop-in for any HTTP client that supports proxies. Pros: works with everything (`curl`, `requests`, browsers, Selenium, your existing scraping codebase). Cons: TLS Pro / Ultra modes require installing our root CA in your trust store.

```bash
curl -x "http://nka_YOUR_KEY:nkp_YOUR_PASSWORD@proxy.nikaproxy.com:8080" \
     "https://httpbin.org/ip"
```

Plain residential mode (above) passes the CONNECT through transparently. Your client terminates TLS directly with the destination, no CA install needed.

For TLS Pro / Ultra mode (with fingerprint signing), the engine MITMs your traffic — download `NikaRootCA.crt` from [/dashboard/certificate](https://nikaproxy.com/dashboard/certificate) and add `--cacert NikaRootCA.crt` to your curl command, or trust it system-wide.

### Path B: REST scrape API

Send a URL to our endpoint, get the destination's response back. No proxy plumbing, no CA install.

```bash
curl -X POST "https://nikaproxy.com/api/scrape/fetch" \
     -H "X-Api-Key: nka_YOUR_KEY" \
     -H "Content-Type: application/json" \
     -d '{"url":"https://httpbin.org/ip","method":"GET"}'
```

Use when:
- You can't easily plumb a proxy into your runtime (edge functions, serverless, browser-side)
- You want NikaProxy to handle TLS fingerprinting without local CA setup
- One-shot scrapes where proxy URL setup is more friction than the request itself

## 4. Run a runnable example

[github.com/Nika-Proxy/nikaproxy-examples](https://github.com/Nika-Proxy/nikaproxy-examples) has tested code in Python, Node, Go, PHP, and curl. Copy your credentials in, run, done.

## Next steps

- [Proxy URL format](proxy-url-format.md) — sticky sessions, country selection, TLS profile pinning
- [Proxy modes](proxy-modes.md) — Residential vs TLS Pro vs TLS Ultra; pick by use case
- [Scrape API reference](scrape-api.md) — full request schema + response envelope
- [Error codes](errors.md) — what each HTTP status means and how to fix it
