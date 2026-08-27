---
name: sharethis-share-counts
description: >-
  Read aggregate social share and click counts for any public URL from the ShareThis Social Share
  Count API. Requires no account, no API key and no onboarding — use this when asked how widely a
  URL has been shared across social networks.
api: ShareThis Social Share Count API
operations:
- GET /get_counts
---

# Read ShareThis social share counts

## Why this one is different

This is the only ShareThis API that needs **no authentication at all**. There is no key, no token
and no account. An agent can call it immediately.

It is also **read-only** — there is nothing here that can be changed, deleted or spent, so none of
the reversibility cautions that apply to the Platform API apply here.

## The call

```
GET https://count-server.sharethis.com/v2.0/get_counts?url=<url-encoded target URL>
```

Parameters:

- `url` (**required**) — the URL to look up. URL-encode it.
- `wd` (optional) — set `wd=true` only if the site uses the legacy `wd.sharethis.com` widget.
  Counts are tracked separately for that widget; omit this for modern installs.

No headers are required. The response is `application/json`.

## Response shape

A bare JSON object — note it does **not** use the `{code, data}` envelope that the Platform API
uses. Top-level keys observed:

- `clicks` — per-network inbound click counts, plus `clicks.all`. Formerly called "inbound".
- `shares` — per-network outbound share counts, plus `shares.all`. Formerly called "outbound".
- `reactions` — counts per reaction (`slight_smile`, `heart_eyes`, `laughing`, `astonished`,
  `rage`, `sob`), plus `reactions.all`.
- `total` — overall total.
- `ourl` — the URL the counts were resolved for. **Check this**: it is how you confirm the service
  matched the URL you actually asked about.

Network keys are numerous (facebook, twitter, linkedin, pinterest, whatsapp, reddit, email, sms,
and dozens more) and the set is not fixed — iterate the returned keys rather than assuming a list.

## Interpreting the numbers honestly

- Counts are cumulative over the life of the URL, not windowed, and carry no date range.
- A URL only accumulates counts if the site actually runs the ShareThis script. A zero or missing
  result usually means the site does not use ShareThis, **not** that the content was never shared.
  Say that when reporting a zero.
- Many network keys reflect long-dead services (stumbleupon, digg, friendster, myspace). Treat large
  values on those as historical, not current.
- Do not present these as total social sharing for a URL across the internet — they are ShareThis's
  observed counts only.

## Limits and failure handling

No rate limits are published and no rate-limit headers are returned. The service is served through
CloudFront. Batch politely, cache results, and back off on any non-200 — there is no `Retry-After`
to guide you and no status page to check.
