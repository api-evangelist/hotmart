---
name: Receive and validate Hotmart webhook events
description: Stand up an endpoint that receives Hotmart purchase, subscription, members-area and logistics events, validate the hottok, dedupe by event id, and respond correctly so Hotmart does not mark the delivery failed.
api: https://developers.hotmart.com/docs/en/1.0.0/webhook/about-webhook/
operations:
  - WEBHOOK PURCHASE_APPROVED
  - WEBHOOK PURCHASE_REFUNDED
  - WEBHOOK PURCHASE_CHARGEBACK
  - WEBHOOK PURCHASE_OUT_OF_SHOPPING_CART
  - WEBHOOK SUBSCRIPTION_CANCELLATION
  - WEBHOOK SWITCH_PLAN
  - WEBHOOK UPDATE_SUBSCRIPTION_CHARGE_DATE
  - WEBHOOK CLUB_FIRST_ACCESS
  - WEBHOOK CLUB_MODULE_COMPLETED
  - WEBHOOK ORDER_FULFILLMENT
generated: '2026-08-04'
method: generated
source: >-
  https://developers.hotmart.com/docs/en/1.0.0/webhook/using-webhook/,
  https://developers.hotmart.com/docs/en/2.0.0/webhook/purchase-webhook/,
  https://developers.hotmart.com/docs/en/1.0.0/webhook/http-response-codes-webhook/
---

# Receive and validate Hotmart webhook events

Hotmart's event surface is a webhook (Hotmart calls it "postback"). There is no
AsyncAPI document and no subscription API — the configuration is created by a human
in the Hotmart platform, per product, choosing the events and the receiving URL.

**Choose version 2.0.0.** Every field added since 2022 (order bump, buyer details,
currency, offer metadata, product variants, shipping) landed on 2.0.0 only. 1.0.0 is
still selectable but frozen in practice.

## Step 1 — expose an endpoint

Accept `POST` with a JSON body. The envelope is:

| Field | Meaning |
|---|---|
| `id` | Unique id of this event delivery — **your dedupe key** |
| `creation_date` | Event creation time, epoch milliseconds |
| `event` | Event name (see below) |
| `version` | `2.0.0` |
| `data` | The payload |
| `hottok` | The account shared secret (also sent as a header) |

## Step 2 — validate the hottok before doing anything

```
X-HOTMART-HOTTOK: <your account's hottok>
```

Compare it against your stored value with a constant-time comparison and reject
anything else. Rotating it requires contacting Hotmart support.

**Understand what this is not.** It is a *static shared secret*, not an HMAC
signature over the body and not timestamped. It proves nothing about payload
integrity and offers no replay protection. Because of that:

- always dedupe on the envelope `id`;
- treat the webhook as a *trigger*, and re-read authoritative state from the REST API
  (`GET /payments/api/v1/sales/history`,
  `GET /payments/api/v1/subscriptions`) before doing anything with money or access.

## Step 3 — handle the events you subscribed to

| Event | Meaning |
|---|---|
| `PURCHASE_APPROVED` | Payment approved — the usual "grant access" trigger |
| `PURCHASE_COMPLETE` | Warranty window elapsed |
| `PURCHASE_BILLET_PRINTED` | Boleto generated, not yet paid |
| `PURCHASE_DELAYED` | Payment delayed |
| `PURCHASE_EXPIRED` | Expired unpaid |
| `PURCHASE_CANCELED` | Purchase cancelled |
| `PURCHASE_REFUNDED` | Refunded — revoke access |
| `PURCHASE_CHARGEBACK` | Charged back — revoke access |
| `PURCHASE_PROTEST` | Entered dispute |
| `PURCHASE_OUT_OF_SHOPPING_CART` | Cart abandonment (lead, not a buyer) |
| `SUBSCRIPTION_CANCELLATION` | Subscription cancelled |
| `SWITCH_PLAN` | Subscriber changed plan |
| `UPDATE_SUBSCRIPTION_CHARGE_DATE` | Billing date changed |
| `CLUB_FIRST_ACCESS` | Student's first members-area access |
| `CLUB_MODULE_COMPLETED` | Student completed a module |
| `ORDER_FULFILLMENT` | Physical-product logistics data |

Key ids on the purchase payload: `product.ucode` (store this, not `product.id`),
`purchase.transaction` (`HP…`), `purchase.status`, `purchase.payment_type`,
`commissions[].source`, and `buyer.email`.

## Step 4 — respond correctly

Return `2XX` **fast**. Hotmart records the delivery outcome for 60 days and treats
these as failures:

| Status | What Hotmart concluded |
|---|---|
| `400` | Your service says a required parameter was missing or invalid |
| `401` | Your service demanded a key — check you are validating `hottok`, not something else |
| `404` | Your URL does not exist |
| `408` / `5XX` | You did not respond in time |
| `-1` | You closed the connection without a reason (Hotmart's own sentinel, not HTTP) |

So: acknowledge first, process asynchronously. Never do the downstream work inline.

## Rules

- **Hotmart documents no retry or backoff schedule** for failed deliveries — do not
  assume a missed event will come back. Reconcile periodically with
  `GET /payments/api/v1/sales/history` and `GET /payments/api/v1/subscriptions`.
- Event history is retained for **60 days** and is visible only in the platform UI.
- Payloads contain buyer personal data (email, phone, national ID document, full
  address). Store only what you need.
- Test events sent from the platform carry the creator's default hottok, so your
  validation path is exercised the same way in test.
