<div align="center">

# NikaProxy — Documentation

Customer-facing documentation for the NikaProxy residential MITM proxy.

[![Website](https://img.shields.io/badge/Website-nikaproxy.com-d4af37)](https://nikaproxy.com)
[![Live docs](https://img.shields.io/badge/Live%20docs-nikaproxy.com%2Fdocs-cyan)](https://nikaproxy.com/docs)
[![Examples](https://img.shields.io/badge/Code%20examples-Nika--Proxy%2Fnikaproxy--examples-green)](https://github.com/Nika-Proxy/nikaproxy-examples)

</div>

This is the source for the docs at [nikaproxy.com/docs](https://nikaproxy.com/docs) — mirrored here in Markdown for direct GitHub reading, deep-linking, and community PRs.

## Contents

### Getting started

- [**Quick start**](getting-started.md) — sign up, grab credentials, fire your first request (5 min)
- [**Authentication**](authentication.md) — API keys, proxy passwords, where they live

### Proxy modes

- [**Residential, TLS Pro, TLS Ultra**](proxy-modes.md) — the three product tiers and when to use which
- [**Proxy URL format**](proxy-url-format.md) — the username syntax (`-country`, `-session`, `-profile`, `-coord`, `-uav`, `-osv`, `-noae`)
- [**Sticky sessions**](sticky-sessions.md) — how to pin requests to the same exit IP, lifetime values, expected churn

### Scrape API

- [**REST scrape API**](scrape-api.md) — endpoint reference, request schema, response envelope, code examples

### Operations

- [**Error codes**](errors.md) — HTTP status mapping + how to fix each
- [**Pricing & bandwidth**](pricing.md) — what counts as billable, plan tiers, balance behaviour
- [**TLS certificate setup**](certificate.md) — only required for TLS Pro / Ultra modes

## On the source of truth

The website at [nikaproxy.com/docs](https://nikaproxy.com/docs) is canonical — it has live integration examples, the dashboard's actual UI, and the up-to-the-minute API surface. This GitHub mirror is updated periodically. **If they disagree on a detail, trust the website.**

## Spot a bug? File an issue

Docs PRs welcome. Please don't post API keys, real customer emails, sample destination URLs that include private data, or anything else that would be problematic in a public issue.

## Need help?

- **Discord**: [discord.gg/xeeDxZMRh](https://discord.gg/xeeDxZMRh) � community + dev questions, fastest reply
- **Telegram**: [@Nikaproxy_support](https://t.me/Nikaproxy_support) � 1-on-1 support
- **Live docs**: [nikaproxy.com/docs](https://nikaproxy.com/docs)
- **Examples**: [github.com/Nika-Proxy/nikaproxy-examples](https://github.com/Nika-Proxy/nikaproxy-examples)
