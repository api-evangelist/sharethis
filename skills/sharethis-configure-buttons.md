---
name: sharethis-configure-buttons
description: >-
  Configure, inspect and remove ShareThis button apps (inline share, sticky share, inline follow,
  inline reaction) on a verified property, over the ShareThis MCP server or the Platform API.
  Use this after the property exists and is verified.
api: ShareThis Platform API
operations:
- POST /properties/{property_id}/apps
- listPropertyApps
- getPropertyApp
- deletePropertyApp
mcp_tools:
- sharethis_apps_upsert
- sharethis_apps_list
- sharethis_apps_get
- sharethis_apps_delete
- sharethis_apps_liveview
---

# Configure ShareThis buttons on a property

## Prerequisites

A verified property — see `sharethis-onboard-property`. Authenticate as described there.

## The four apps

`app_id` is a closed enum. There are exactly four, one config schema each:

| app_id | What it is | Config schema |
|---|---|---|
| `inline-share-buttons` | Share buttons in the page flow | `InlineShareButtonConfig` |
| `sticky-share-buttons` | Fixed/floating share rail | `StickyShareButtonsConfig` |
| `inline-follow-buttons` | Follow buttons | `InlineFollowButtonsConfig` |
| `inline-reaction-buttons` | Reaction buttons | `InlineReactionButtonsConfig` |

A property can hold at most these four. Anything else the user names — the Privacy Policy Generator
or the Consent Management Platform — is **not** API-configurable; say so rather than attempting it.

## STOP — what cannot be undone

- **`sharethis_apps_upsert` overwrites the existing configuration outright.** There is no version
  history and no restore endpoint. Always read the current config first (step 1) so you can tell
  the user what is about to change, and so you hold a copy of the old values.
- **`sharethis_apps_delete` is the one destructive operation on this API** — ShareThis itself
  annotates it `destructiveHint: true`. You can re-create the app afterwards by upserting the same
  `app_id`, but that is a fresh create: **the previous configuration is gone** and must be supplied
  again from scratch. ShareThis publishes no undo and no recovery window. Confirm with the user
  before calling it, every time.

## Steps

1. **Read what is configured now.**
   - MCP: `sharethis_apps_list` with `{"property_id": "..."}`, then `sharethis_apps_get` with
     `{"property_id": "...", "app_id": "..."}`.
   - REST: `listPropertyApps` (`GET /properties/{property_id}/apps/`), then `getPropertyApp`
     (`GET /properties/{property_id}/apps/{app_id}`).
   - Keep the returned config — it is your only copy if an upsert goes wrong.

2. **Create or update the configuration.**
   - MCP: `sharethis_apps_upsert` — requires `property_id`, `app_id` **and** `config` (all three
     are mandatory; there is no partial update, so `config` must be complete).
   - REST: `POST /properties/{property_id}/apps`.
   - One call handles both create and update, keyed on `(property_id, app_id)`, so repeating an
     identical upsert is safe in effect even though no idempotency mechanism is declared.
   - Share button configs carry a `ShareNetworks` vocabulary; follow buttons carry `FollowNetworks`;
     reaction buttons carry `Reactions` (`slight_smile`, `heart_eyes`, `laughing`, `astonished`,
     `rage`, `sob`).
   - `400 Invalid app id or configuration` means the `app_id` is outside the enum or the config does
     not match that app's schema — check which of the two before retrying.

3. **Verify what actually renders.**
   - MCP: `sharethis_apps_liveview` with `{"property_id": "...", "app_id": "..."}`.
   - This renders the app's **saved** state through the property's `sharethis.js` runtime. It is
     an after-the-fact inspection, not a preview — there is no dry-run or simulate mode on this
     API, so you cannot rehearse a change before committing it.

4. **Remove an app only on explicit instruction.**
   - MCP: `sharethis_apps_delete`. REST: `deletePropertyApp`.
   - Re-read the config first and show the user what will be lost. See the STOP section above.

## Errors

| Status | Meaning | What to do |
|---|---|---|
| 400 | Invalid app id or configuration | Check `app_id` against the enum and the config against its schema |
| 401 | Authentication required | Re-authenticate |
| 404 | Property or app not found | The app may simply not be configured yet — list first |
| 500 | Internal server error | Retry with backoff; no Retry-After is provided |

## Limits

No rate limits are published and no rate-limit headers are returned, so pace conservatively and
back off on 500s rather than assuming headroom.
