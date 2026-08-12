---
name: Pull player registrations from a Playable campaign
description: Authenticate against the Playable API and page through the zero-party data a gamification campaign has collected, including the email and SMS the campaign sent.
api: openapi/playable-api-openapi.yml
operations:
  - POST /oauth/token
  - GET /v1/campaigns
  - GET /v1/campaign/{campaign}/registrations
  - GET /v1/campaign/{campaign}/registration/{registration}
  - GET /v1/campaign/{campaign}/email-log
  - GET /v1/campaign/{campaign}/sms-log
scopes:
  - campaigns.list
  - campaigns.registrations.list
  - campaigns.registrations.view
  - campaigns.email-log.list
  - campaigns.sms-log.list
generated: '2026-08-12'
method: generated
source: openapi/playable-api-openapi.yml
---

# Pull player registrations from a Playable campaign

The Playable OpenAPI declares **no operationIds**, so every step below is keyed by METHOD + path,
which is the only stable identifier the provider publishes. Do not invent operation names.

## 1. Get a token

`POST /oauth/token` on `https://api.playable.com` using the OAuth 2.0 **client_credentials** grant
with a developer app created in the platform under *Global settings / Developer apps*. Request only
the scopes this flow needs — the scopes are listed in the frontmatter. API access is a **Premium**
plan entitlement; on a lower plan the credential will not exist.

Send on every subsequent call:

- `Authorization: Bearer {{ACCESS_TOKEN}}`
- `Accept: application/json` — the only supported response type.

A 401 with `{"message": "Unauthenticated."}` means the token expired. Re-issue; there is no refresh
token in the client-credentials grant.

## 2. Find the campaign

`GET /v1/campaigns` (scope `campaigns.list`). Narrow with `filter_name`, `filter_type`,
`filter_display=active|inactive`, `filter_template`; order with `sort=name,created_on` plus
`asc`/`desc`.

Read the integer `id` off the row. Note `live_url` and `demo_url` — the demo URL is how you exercise
a campaign without touching live activity.

## 3. Page through registrations

`GET /v1/campaign/{campaign}/registrations` (scope `campaigns.registrations.list`).

Pagination is **page-number**: pass `?page=N`. The body is `{data, links, meta}` —
`links.next` is an absolute URL, and `meta` carries `current_page`, `from`, `to`, `per_page`, `path`.
**There is no total count**, so terminate on `links.next` being absent, never on an arithmetic page
count. Page size is not settable.

For a single record: `GET /v1/campaign/{campaign}/registration/{registration}`
(scope `campaigns.registrations.view`).

## 4. Reconcile messaging

`GET /v1/campaign/{campaign}/email-log` and `GET /v1/campaign/{campaign}/sms-log`
(scopes `campaigns.email-log.list`, `campaigns.sms-log.list`) — same pagination shape — tell you what
the campaign actually sent to those players.

## Rules

- **Rate limit: 3,600 requests/hour per developer app.** Hard; it cannot be raised, and splitting
  traffic across multiple apps to evade it is explicitly prohibited. Watch `X-RateLimit-Remaining`
  on every response and back off on `Retry-After`. The status code returned on exhaustion is *not*
  published, so treat *any* response carrying `Retry-After` as throttled.
- **Cache.** Playable states it expects clients to cache and to fetch only when needed, and that
  clients that do not "may be subject to rate limiting or have your access revoked." Persist a
  cursor and pull incrementally rather than re-walking every page.
- **No idempotency key exists** on this API. Reads are safe to retry; see the sibling skills before
  retrying anything that writes.
- Errors are `{"message": "..."}` — free text, no error code, not RFC 9457. Branch on HTTP status
  only: 400 invalid request, 401 expired/absent token, 403 (used for a missing campaign type, not
  only for permission), 404 not found.
- **This is personal data.** Registrations are player-submitted zero-party data under GDPR;
  `DELETE /v1/campaign/{campaign}/registration/{registration}` (scope
  `campaigns.registrations.delete`) is the erasure path. Do not copy this data anywhere the
  customer's DPA does not cover.
