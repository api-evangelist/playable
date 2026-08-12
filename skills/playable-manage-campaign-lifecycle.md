---
name: Run a Playable campaign lifecycle
description: Clone a proven campaign, edit it, activate, pause and resume it, and force its cache to rebuild — the write surface of the Playable API.
api: openapi/playable-api-openapi.yml
operations:
  - POST /oauth/token
  - GET /v1/campaign-types
  - GET /v1/campaign-type/{campaignType}
  - GET /v1/campaigns
  - GET /v1/campaign/{campaign}
  - POST /v1/campaign/copy/{campaign}
  - POST /v1/campaign/{campaign}
  - GET /v1/campaign/{campaign}/game-settings
  - PATCH /v1/campaign/{campaign}/game-settings
  - POST /v1/campaign/{campaign}/activate
  - POST /v1/campaign/{campaign}/pause
  - POST /v1/campaign/{campaign}/resume
  - POST /v1/campaign/{campaign}/clear-cache
  - DELETE /v1/campaign/{campaign}
scopes:
  - campaigns.types.list
  - campaigns.types.view
  - campaigns.list
  - campaigns.view
  - campaigns.copy
  - campaigns.modify
  - campaigns.game-settings.view
  - campaigns.game-settings.modify
  - campaigns.activate
  - campaigns.pause
  - campaigns.resume
  - campaigns.clear-cache
  - campaigns.delete
generated: '2026-08-12'
method: generated
source: openapi/playable-api-openapi.yml
---

# Run a Playable campaign lifecycle

The spec declares **no operationIds**; steps are keyed by METHOD + path. Note that Playable uses
`POST`, not `PUT`, for the update operation.

## 1. Authenticate

`POST /oauth/token` (client_credentials) → `Authorization: Bearer {{ACCESS_TOKEN}}` plus
`Accept: application/json`. Request only the scopes for the step you are performing — the API's scope
map is genuinely fine-grained (activate, pause and resume are three separate scopes), so a
least-privilege token is cheap here.

## 2. Choose a game concept

`GET /v1/campaign-types` (scope `campaigns.types.list`), then
`GET /v1/campaign-type/{campaignType}` (scope `campaigns.types.view`) — campaign types are addressed
by **alias string**, not by integer id.

Careful: an unknown campaign type returns **403**, not 404. Do not treat that 403 as a permission
failure.

## 3. Clone rather than build

There is no create-campaign operation in this API. The supported path is
`POST /v1/campaign/copy/{campaign}` (scope `campaigns.copy`) from an existing campaign or template —
find a source with `GET /v1/campaigns?filter_template=1`.

## 4. Configure

- `POST /v1/campaign/{campaign}` (scope `campaigns.modify`) updates the campaign.
- `GET` / `PATCH /v1/campaign/{campaign}/game-settings` (scopes `campaigns.game-settings.view` /
  `.modify`) read and change the game configuration.
- Sections, form fields, prizes and vouchers are **read-only** over the API — they are edited in the
  campaign builder. Do not attempt writes there; the operations do not exist.

## 5. Verify before going live

Fetch `GET /v1/campaign/{campaign}` (scope `campaigns.view`) and play the campaign through its
`demo_url`. Confirm the run landed by checking `GET /v1/campaign/{campaign}/registrations` and, if
integrations are wired, `GET /v1/campaign/{campaign}/integrations`.

## 6. Go live and control it

`POST /v1/campaign/{campaign}/activate` → `/pause` → `/resume`, each with its own scope. The campaign
resource carries `active`, `active_from` and `active_to`, so a scheduled window may already govern it
— read the campaign before overriding.

After any content change, `POST /v1/campaign/{campaign}/clear-cache` (scope
`campaigns.clear-cache`) forces the cache to rebuild. This matters more than usual on Playable
because the provider expects clients and the platform to cache aggressively.

## Rules

- **No idempotency key exists on this API.** Every write above is a bare POST/PATCH/DELETE with no
  replay protection. On a timeout or a connection reset, do **not** blind-retry: re-read the campaign
  with `GET /v1/campaign/{campaign}` and reconcile state first. `POST /v1/campaign/copy/{campaign}` is
  the dangerous one — a blind retry produces a duplicate campaign.
- `DELETE /v1/campaign/{campaign}` (scope `campaigns.delete`) is destructive and takes the campaign's
  collected registrations with it. Require an explicit human confirmation before calling it.
- **3,600 requests/hour per developer app**, hard. `X-RateLimit-Remaining` on every response;
  `Retry-After` once exhausted. The throttled status code is not documented.
- Errors are `{"message": "..."}` with no code. Declared statuses are 400, 401, 403, 404; **no 5xx is
  declared anywhere in the spec**, so a server error will arrive in a shape the contract never
  described. Fail closed on any undeclared status rather than retrying a write.
- No `Sunset`/`Deprecation` header support and no published deprecation policy — pin nothing on
  advance notice of a breaking change.
