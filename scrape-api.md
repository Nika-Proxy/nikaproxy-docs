# Scrape API

REST endpoint that wraps the proxy + TLS engine: send a URL, get the destination's response back. No proxy plumbing, no CA install. Same backend, same exits, same profiles as the HTTP proxy path.

## Endpoint

```
POST https://nikaproxy.com/api/scrape/fetch
Content-Type: application/json
X-Api-Key: nka_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

Auth header is `X-Api-Key`, not `Authorization: Bearer`. Both your API key and proxy password identify the same account, but the REST API only uses the API key — the proxy password isn't needed here.

## Request body

```json
{
  "url":             "https://example.com/path?q=hello",
  "method":          "GET",
  "headers":         { "Accept-Language": "en-US,en;q=0.9" },
  "body":            null,

  "country":         "US",
  "sessionId":       "myFlow01",
  "lifetimeMinutes": 60,

  "profile":         "chrome_windows_normal_2508",
  "autoEmulate":     true,
  "uaVersion":       148,
  "osVersion":       null,
  "families":        null
}
```

### Field reference

| Field | Type | Required | Notes |
|---|---|---|---|
| `url` | string | yes | Full URL including scheme + path + query. Only `http://` and `https://` accepted. |
| `method` | string | no (default `GET`) | Any HTTP method. POST/PUT/PATCH/DELETE/HEAD/OPTIONS supported. |
| `headers` | object | no | Custom headers passed through to the destination. Subject to Auto-Emulation rewriting when `autoEmulate: true` — Auto-Emulate replaces the customer-supplied versions of UA / Sec-CH-UA* / Accept-* headers. |
| `body` | string | no | Request body (string). For JSON, send the stringified JSON. For binary, base64-encode and set a custom `Content-Encoding` header (the engine will pass through). |
| `country` | string | no | 2-letter ISO country code for the residential exit. Omit for worldwide. |
| `sessionId` | string | no | Sticky session pin — same value across requests = same exit IP. Omit for rotating (fresh IP each call). |
| `lifetimeMinutes` | int | no (default 10) | Sticky session lifetime. Must be `0`, `5`, `10`, `15`, `20`, or `60`. |
| `profile` | string | no | TLS profile name (e.g. `chrome_windows_normal_2508`) OR a `coord_*` token. Omit for plain Residential mode. |
| `autoEmulate` | bool | no (default `false`) | When `true` + a profile is set, the engine replaces your UA / Sec-CH-UA* / Accept-* headers with the profile's defaults so the HTTP layer matches the TLS layer. |
| `uaVersion` | int | no | Pin UA major version when Auto-Emulate is on. E.g. `148` → forces Chrome 148.x. Falls back to the profile's pool if the version isn't available. |
| `osVersion` | int | no | Pin Android OS major version (Firefox Android only). E.g. `15` → Android 15. |
| `families` | string[] | no | For `coord_*` profiles only — restrict which profile families the coordinator may pick. E.g. `["chrome", "firefox"]`. |
| `envelope` | bool | no (default false) | When `true`, response is wrapped in a JSON envelope with full headers; otherwise the destination's response is forwarded transparently. |

## Response

### Default (envelope=false)

Response is the destination's response, passed through transparently:

- `Status-Line` = upstream status
- Headers = upstream headers (minus hop-by-hop)
- Body = upstream body

NikaProxy adds informational response headers:

| Header | Meaning |
|---|---|
| `X-Nika-Profile` | The TLS profile that was actually used (relevant for `coord_*` since coord picks one for you) |
| `X-Nika-Elapsed-Ms` | Upstream round-trip in milliseconds |
| `X-Nika-Bytes-Used` | Response body bytes (the customer-facing usage figure) |
| `X-Nika-Status` | Upstream HTTP status (also in the actual status line) |

### Envelope mode (envelope=true)

```json
{
  "statusCode":   200,
  "headers":      { "Content-Type": "text/html", "Set-Cookie": "..." },
  "body":         "<!doctype html>...",
  "bodyIsBase64": false,
  "elapsedMs":    1834,
  "bytesUsed":    24516,
  "profileUsed":  "chrome_windows_normal_2508"
}
```

Use envelope mode when you need to read response headers from your code (most native HTTP clients only expose the body easily).

`bodyIsBase64: true` when the response body wasn't a text MIME type — body is base64-encoded.

## Examples

### Python

```python
import requests

r = requests.post(
    "https://nikaproxy.com/api/scrape/fetch",
    headers={"X-Api-Key": "nka_YOUR_KEY"},
    json={
        "url":     "https://httpbin.org/get",
        "method":  "GET",
        "country": "US",
        "profile": "chrome_windows_normal_2508",
        "autoEmulate": True,
    },
    timeout=60,
)
r.raise_for_status()
print(r.text[:500])
```

### Node.js

```javascript
const r = await fetch("https://nikaproxy.com/api/scrape/fetch", {
  method:  "POST",
  headers: {
    "X-Api-Key":    "nka_YOUR_KEY",
    "Content-Type": "application/json",
  },
  body: JSON.stringify({
    url:         "https://httpbin.org/get",
    method:      "GET",
    country:     "US",
    profile:     "coord_mobile_browser",
    autoEmulate: true,
    families:    ["chrome", "firefox"],
  }),
})
if (!r.ok) throw new Error(`HTTP ${r.status}`)
console.log(r.headers.get("x-nika-profile"))
console.log(await r.text())
```

### POST with a JSON body (passed to the destination)

```python
import json, requests

r = requests.post(
    "https://nikaproxy.com/api/scrape/fetch",
    headers={"X-Api-Key": "nka_YOUR_KEY"},
    json={
        "url":    "https://api.target.com/v1/orders",
        "method": "POST",
        "headers": {
            "Content-Type": "application/json",
            "Authorization": "Bearer THEIR_TOKEN",
        },
        "body": json.dumps({"product_id": 42, "qty": 1}),
        "country": "US",
        "profile": "chrome_windows_normal_2508",
        "autoEmulate": True,
    },
)
print(r.json())
```

## More examples

Working code in 5 languages: [github.com/Nika-Proxy/nikaproxy-examples](https://github.com/Nika-Proxy/nikaproxy-examples)
