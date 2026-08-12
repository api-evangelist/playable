---
name: Award prizes and notify Playable winners
description: Read a campaign's prize and bulk-prize pools, adjust a pool, issue the winner email for a bulk-prize item, and reconcile vouchers and delivery against the email and SMS logs.
api: openapi/playable-api-openapi.yml
operations:
  - POST /oauth/token
  - GET /v1/campaign/{campaign}/prizes
  - GET /v1/campaign/{campaign}/prize/{prize}
  - GET /v1/campaign/{campaign}/bulk-prizes
  - GET /v1/campaign/{campaign}/bulk-prize/{bulkPrize}
  - PATCH /v1/campaign/{campaign}/bulk-prize/{bulkPrize}
  - POST /v1/campaign/{campaign}/bulk-prize/{bulkPrize}/item/{bulkPrizeItem}/send/email
  - GET /v1/campaign/{campaign}/vouchers
  - GET /v1/campaign/{campaign}/voucher/{voucher}
  - DELETE /v1/campaign/voucher
  - GET /v1/campaign/{campaign}/email-log
scopes:
  - campaigns.prizes.list
  - campaigns.prizes.view
  - campaigns.bulk-prizes.list
  - campaigns.bulk-prizes.view
  - campaigns.bulk-prizes.modify
  - campaigns.bulk-prizes-items.send-email
  - campaigns.voucher.list
  - campaigns.voucher.view
  - campaigns.voucher.delete
  - campaigns.email-log.list
generated: '2026-08-12'
method: generated
source: openapi/playable-api-openapi.yml
---

# Award prizes and notify Playable winners

No operationIds exist in the spec; steps are keyed by METHOD + path.

## 1. Authenticate

`POST /oauth/token` (client_credentials), then `Authorization: Bearer {{ACCESS_TOKEN}}` and
`Accept: application/json`.

Scope this token tightly. `campaigns.bulk-prizes-items.send-email` sends real mail to real people and
`campaigns.voucher.delete` destroys redeemable codes — neither belongs in a general-purpose
read token.

## 2. Read the prize surface

- `GET /v1/campaign/{campaign}/prizes` → `GET /v1/campaign/{campaign}/prize/{prize}` for
  individually configured prizes (scopes `campaigns.prizes.list` / `.view`). **Read-only.**
- `GET /v1/campaign/{campaign}/bulk-prizes` → `GET /v1/campaign/{campaign}/bulk-prize/{bulkPrize}`
  for prize pools (scopes `campaigns.bulk-prizes.list` / `.view`).

Both paginate with `?page=N` and return `{data, links, meta}` with no total count — stop when
`links.next` is absent.

## 3. Adjust a pool

`PATCH /v1/campaign/{campaign}/bulk-prize/{bulkPrize}` (scope `campaigns.bulk-prizes.modify`) is the
only prize write in the API.

## 4. Notify a winner

`POST /v1/campaign/{campaign}/bulk-prize/{bulkPrize}/item/{bulkPrizeItem}/send/email`
(scope `campaigns.bulk-prizes-items.send-email`).

**Treat this as the highest-consequence call in the Playable API.** It has an external, irreversible
side effect — an email to a real recipient — and the API provides **no idempotency key**. A blind
retry after a timeout is a duplicate winner notification.

The safe sequence:

1. Snapshot `GET /v1/campaign/{campaign}/email-log` (scope `campaigns.email-log.list`) before sending.
2. Send once.
3. On any timeout, connection reset, or undeclared status — **do not retry.** Re-read the email log
   and check whether the message was recorded before deciding.

Bulk-prize items are not independently readable; `send/email` is the only operation on them, so the
email log is your only confirmation surface.

## 5. Reconcile vouchers

`GET /v1/campaign/{campaign}/vouchers` and `GET /v1/campaign/{campaign}/voucher/{voucher}`
(scopes `campaigns.voucher.list` / `.view`).

`DELETE /v1/campaign/voucher` (scope `campaigns.voucher.delete`) is a **spec anomaly**: unlike every
other voucher operation it is *not* nested under a campaign id, so the campaign it acts on comes from
the request body rather than the path. Confirm the target explicitly before calling it and require a
human confirmation — deleted vouchers are redeemable value.

## Rules

- **3,600 requests/hour per developer app**, hard. Honour `X-RateLimit-Remaining` and `Retry-After`;
  the exhaustion status code is not published.
- Errors are `{"message": "..."}`, no code, not RFC 9457. Only 400/401/403/404 are declared — no 5xx
  anywhere in the spec. Fail closed on anything undeclared rather than retrying a send.
- Winner records are personal data under the customer's DPA. Do not export them beyond the
  reconciliation you were asked for.
