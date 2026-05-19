# Proxy URL format

The proxy URL username carries every customization NikaProxy supports — country selection, sticky session pin, TLS profile, UA pin, OS pin. Password is your proxy password only.

## Anatomy

```
http://USERNAME:PASSWORD@proxy.nikaproxy.com:8080
       └──── tokens ────┘  └── proxy pwd ──┘  └─ HTTP gateway ─┘
                                              proxy.nikaproxy.com:1080 for SOCKS5
```

Username = your API key + zero or more dash-separated tokens.

## Token reference

```
USERNAME = nka_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
           [ -<country>    ]   2-letter ISO (us, gb, de, fr, jp, br, in, …)
           [ -<sessionId>  ]   any alphanumeric ≤ 32 chars — same id = same exit IP
           [ -<lifetime>   ]   0 | 5 | 10 | 15 | 20 | 60 minutes
           [ -<profile>    ]   TLS profile name OR a coord_* token
           [ -noae         ]   disable Auto-Emulation (don't replace UA / Sec-* headers)
           [ -uav<N>       ]   pin UA major version, e.g. -uav148 → Chrome 148
           [ -osv<N>       ]   pin Android OS major, e.g. -osv15 (Firefox Android only)
PASSWORD = nkp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

**Token rules:**
- Order doesn't matter; the engine parses by content not position.
- Tokens are dash-separated (`-`). Don't use spaces or commas.
- Country codes are case-insensitive (`us` and `US` both work).
- Session IDs are case-sensitive — `MyFlow` and `myflow` are different sessions.

## Common patterns

### Plain residential, worldwide, rotating IP every request

```
nka_KEY:nkp_PWD
```

### US residential, sticky 60-minute session

```
nka_KEY-us-myflow-60:nkp_PWD
```

### TLS Pro, Chrome 148 on Windows, US, sticky

```
nka_KEY-us-myflow-60-chrome_windows_normal_2508-uav148:nkp_PWD
```

### TLS Ultra, Win11 desktop browser, sticky

```
nka_KEY-us-myflow-60-coord_win11_browser:nkp_PWD
```

### Firefox Android with pinned OS version

```
nka_KEY-us-myflow-60-firefox_android_normal_2604-osv15:nkp_PWD
```

### Plain residential pass-through (no Auto-Emulation, no header rewriting)

```
nka_KEY-us-myflow-60-chrome_windows_normal_2508-noae:nkp_PWD
```

The `-noae` flag tells the engine to apply the TLS profile but leave your client's existing User-Agent / Sec-CH-UA* / Accept-* headers alone. Use this when your client is already producing correct headers and you only need the TLS signing layer.

## SOCKS5

Same hostname, port `1080`, same auth pattern. All tokens behave identically.

```
socks5://nka_KEY-us-myflow-60-coord_mobile_browser:nkp_PWD@proxy.nikaproxy.com:1080
```

## Generating URLs from the dashboard

The [Generate](https://nikaproxy.com/dashboard/generate) page in the dashboard has a visual builder — pick country, sticky settings, profile, UA pin, and it produces the username string for you. Save common configurations as presets to reuse.

## Why dashes instead of a real query string

The HTTP-CONNECT proxy protocol passes the username/password via Basic auth, which doesn't carry structured parameters. Dash-separated tokens in the username are how every residential proxy provider does this — same convention as Bright Data, Smartproxy, Oxylabs, etc. — so existing tooling and proxy-management libraries already understand the shape.

## Token validation errors

The engine validates the token combination before opening the upstream. Common failures:

| Error | Cause |
|---|---|
| `invalid_country: XY` | Country code isn't a recognised ISO-3166 alpha-2 |
| `invalid_lifetime: 30` | Lifetime must be 0, 5, 10, 15, 20, or 60 |
| `profile_not_found: foo_bar` | Typo or non-existent profile name |
| `feature_locked: tls_ultra` | Your plan doesn't include the coord_* family — upgrade or pick a TLS Pro profile |
| `uav_not_in_pool: 999` | UA version not in this profile's pool; pick a profile that covers it |
