---
name: sharethis-onboard-property
description: >-
  Register a website domain as a ShareThis property and verify ownership of it, using either the
  ShareThis MCP server or the Platform API. Use this before configuring any share, follow or
  reaction buttons, because every app configuration hangs off a verified property.
api: ShareThis Platform API
operations:
- POST /properties/
- POST /properties/{property_id}/validate
- GET /properties
- GET /properties/{property_id}
mcp_tools:
- sharethis_properties_create
- sharethis_properties_validate
- sharethis_properties_list
- sharethis_properties_get
---

# Onboard a ShareThis property

## Before you start

- A ShareThis account is required (https://platform.sharethis.com).
- Over MCP: connect to `https://mcp.sharethis.com` and complete OAuth (scope `mcp:tools`).
- Over REST: `POST https://platform-api.sharethis.com/v2.0/auth/login` to get a bearer JWT. This is
  the only operation that does not require authentication.

## STOP — read this before creating anything

**Property creation cannot be undone.** The ShareThis Platform API has no delete-property
operation — there is no way to remove a property through the API once it exists. Property count is
also quota-limited (`403 Property limit exceeded`), and the numeric limit is not published. A
property created by mistake permanently consumes quota you cannot reclaim.

There is also **no idempotency support** — no `Idempotency-Key` header is accepted. If a create
call times out, do **not** blindly retry: list properties first and check whether the create
actually succeeded.

## Steps

1. **Check whether the property already exists.**
   - MCP: `sharethis_properties_list` (no arguments).
   - REST: `GET /properties`.
   - The response is the full unbounded collection — there is no pagination. Match on `domain`.
   - If the domain is already present, skip to step 3 and check its `verified` flag.

2. **Create the property.**
   - MCP: `sharethis_properties_create` with `{"domain": "example.com"}` — `domain` is the only
     input and is required (minLength 1).
   - REST: `POST /properties/` with the same body.
   - Success returns a `property_id`.
   - Handle `403 Property limit exceeded` by reporting the quota to the user; you cannot free
     quota yourself, because properties cannot be deleted.

3. **Verify domain ownership.**
   - Ownership is proved by placing a file on the domain, then calling validation.
   - MCP: `sharethis_properties_validate` with `{"property_id": "..."}`.
   - REST: `POST /properties/{property_id}/validate`.
   - `400` means the verification file is missing or invalid — the user must place it before you
     retry.
   - `409 Property already verified` means the property is already done. Treat this as success,
     not as an error; verification is a one-way transition and there is no un-verify operation.

4. **Confirm final state.**
   - MCP: `sharethis_properties_get` with `{"property_id": "..."}`.
   - REST: `GET /properties/{property_id}`.
   - Check `verified: true` before moving on to app configuration.
   - Note the identifier naming mismatch: the response field is `_id`, while every request uses
     `property_id`. They are the same 24-character hex value.

## Errors

| Status | Meaning | What to do |
|---|---|---|
| 400 | Invalid input / verification file missing or invalid | Fix input or ask the user to place the file |
| 401 | Authentication required | Re-authenticate; the bearer JWT has no documented lifetime |
| 403 | Property limit exceeded | Report to the user — quota cannot be freed |
| 404 | Property not found | Re-list properties; do not assume the id |
| 409 | Property already verified | Treat as success |
| 500 | Internal server error | Retry with backoff; no Retry-After is provided |

Errors arrive in ShareThis's `{code, data}` envelope, **not** RFC 9457 problem+json. Branch on the
HTTP status together with the `code` string.

## Reporting back

There is no status page and no request-id header, so if calls fail repeatedly you cannot
distinguish a ShareThis outage from a client problem and you have no identifier to give support.
Say so plainly rather than retrying indefinitely.
