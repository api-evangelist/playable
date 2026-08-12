---
name: Report on Playable campaign performance
description: Assemble a campaign performance report from Playable's statistics, session, registration and game-data endpoints, and confirm which downstream integrations the campaign is feeding.
api: openapi/playable-api-openapi.yml
operations:
  - POST /oauth/token
  - GET /v1/campaigns
  - GET /v1/campaign/{campaign}
  - GET /v1/campaign/{campaign}/statistics
  - GET /v1/campaign/{campaign}/statistics/sessions
  - GET /v1/campaign/{campaign}/statistics/registrations
  - GET /v1/campaign/{campaign}/game-data-statistics
  - GET /v1/campaign/{campaign}/integrations
  - GET /v1/campaign/{campaign}/sections
  - GET /v1/campaign/{campaign}/section/{section}/form-fields
scopes:
  - campaigns.list
  - campaigns.view
  - campaigns.game-data-statistics
  - campaigns.integrations.list
  - campaigns.sections.list
  - campaigns.sections.form-fields.list
generated: '2026-08-12'
method: generated
source: openapi/playable-api-openapi.yml
---

# Report on Playable campaign performance

No operationIds exist in the spec; steps are keyed by METHOD + path. Every operation here is a read.

## 1. Authenticate

`POST /oauth/token` (client_credentials) → `Authorization: Bearer {{ACCESS_TOKEN}}`,
`Accept: application/json`. A reporting token needs only the six scopes in the frontmatter — note
that all three `statistics` endpoints are covered by plain `campaigns.view`, not by a dedicated
analytics scope.

## 2. Scope the report

`GET /v1/campaigns` (scope `campaigns.list`), filtered with `filter_display=active`, `filter_type`,
`filter_name`, and sorted with `sort=created_on,desc`.

Prefer the `with=` expansion over extra round trips — one call can carry
`registrations,sessions,is_instant_win,integrations,sections,sections.form_fields,bulk_prizes,sections.sessions`.
That is the cheapest way to stay inside the rate limit.

## 3. Pull the numbers

- `GET /v1/campaign/{campaign}/statistics` — the headline aggregate (scope `campaigns.view`).
- `GET /v1/campaign/{campaign}/statistics/sessions` — plays.
- `GET /v1/campaign/{campaign}/statistics/registrations` — conversions.
- `GET /v1/campaign/{campaign}/game-data-statistics` (scope `campaigns.game-data-statistics`) —
  per-game-type result data.

**Read `game-data-statistics` carefully:** for a game type that does not support it the API returns
an **empty array**, not an error. An empty array is "not supported for this game", not "zero
activity" — never report it as a zero.

## 4. Explain the funnel

`GET /v1/campaign/{campaign}/sections` (scope `campaigns.sections.list`) gives the flow steps, and
`GET /v1/campaign/{campaign}/section/{section}/form-fields` (scope
`campaigns.sections.form-fields.list`) gives what each step asked the player for. Together these turn
a session/registration ratio into a story about where players dropped.

## 5. Confirm the data actually left

`GET /v1/campaign/{campaign}/integrations` (scope `campaigns.integrations.list`) lists the ESP, CRM,
storage and webhook connections wired to the campaign. Delivery status per attempt is not in the API
— it is in the platform's integration log, and the status codes it records are catalogued in
`errors/playable-integration-status-codes.yml`.

## Rules

- Paginate with `?page=N` and stop on the absence of `links.next`. **`meta` carries no total and no
  last-page count**, so never compute a page count and never assert a total row count you did not
  count yourself.
- **3,600 requests/hour per developer app**, hard. A naive per-campaign fan-out across a large
  account will exhaust it — batch with `with=` expansions, and cache. Playable explicitly warns that
  clients that do not cache "may be subject to rate limiting or have your access revoked."
- Watch `X-RateLimit-Remaining` on every response and back off on `Retry-After`. The exhaustion
  status code is not published, so treat any response carrying `Retry-After` as throttled.
- Errors are `{"message": "..."}` with no error code. Declared statuses are 400/401/403/404 only;
  a 403 on a campaign-type lookup means "unknown type", not "forbidden".
- These reads are safe to retry — no idempotency key exists on this API, but nothing in this skill
  writes.
