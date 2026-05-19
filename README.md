<div align="center">

# NikaProxy — Documentation

Customer-facing documentation for the NikaProxy residential MITM proxy.

[![Website](https://img.shields.io/badge/Website-nikaproxy.com-d4af37)](https://nikaproxy.com)
[![Live docs](https://img.shields.io/badge/Live%20docs-nikaproxy.com%2Fdocs-cyan)](https://nikaproxy.com/docs)

</div>

This is the source for the docs available at [nikaproxy.com/docs](https://nikaproxy.com/docs) — mirrored here in Markdown for direct GitHub reading, deep-linking, and PRs from the community.

## What's in here

- **[Getting started](getting-started.md)** — sign up, grab credentials, fire your first request
- **[Proxy modes](modes.md)** — Residential, TLS Pro, TLS Ultra: when to use which
- **[Proxy URL grammar](proxy-url.md)** — the full username syntax (`-country-`, `-session-`, `-profile-`, `-coord-`, `-ua-v`, `-noae`)
- **[Sticky sessions](sticky-sessions.md)** — what they do, lifetime values, when they expire, gotchas
- **[Scrape API](scrape-api.md)** — REST endpoint reference, request schema, response shapes
- **[TLS certificate setup](certificate.md)** — only required for TLS Pro / Ultra
- **[Pricing & billing](pricing.md)** — bandwidth credit, TLS Engine subscription, what counts as billable
- **[FAQ](faq.md)** — most-asked questions, by frequency

## On the source of truth

The website at [nikaproxy.com/docs](https://nikaproxy.com/docs) is canonical — it has live integration examples, the dashboard's actual UI, and the up-to-the-minute API surface. This GitHub mirror is updated weekly. If they disagree on a detail, **trust the website**.

## Spot a bug? File an issue

Docs PRs welcome. Please don't post API-key snippets, sample destination URLs that include private data, or anything that would be problematic in a public issue.

## Need help?

- Telegram (fastest): [@Nikaproxy_support](https://t.me/Nikaproxy_support)
- Customer docs (live): [nikaproxy.com/docs](https://nikaproxy.com/docs)
