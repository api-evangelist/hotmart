---
name: Find overdue or cancelled subscriptions and reach their subscribers
description: List Hotmart subscriptions in a delayed, overdue or cancelled state over a date window, pull the buyer contact behind each one, and check whether they still have members-area access — the recovery/winback flow.
api: https://developers.hotmart.com/docs/en/v1/subscription/get-subscribers/
operations:
  - GET /payments/api/v1/subscriptions
  - GET /payments/api/v1/subscriptions/:subscriber_code/purchases
  - GET /payments/api/v1/sales/users
  - GET /club/api/v1/users
generated: '2026-08-04'
method: generated
source: >-
  https://developers.hotmart.com/docs/en/tutorials/overdue-subscriptions/,
  https://developers.hotmart.com/docs/en/tutorials/canceled-subscriptions/
---

# Find overdue or cancelled subscriptions and reach their subscribers

Authenticate first — see `hotmart-authenticate.md`. Every request below needs
`Authorization: Bearer <access_token>`. Base host is
`https://developers.hotmart.com` (or `https://sandbox.hotmart.com` in test).

**All dates are epoch milliseconds, UTC.** Multi-valued filters are sent by repeating
the same query key, not as a comma list.

## Step 1 — list the subscriptions in the state you care about

Overdue / delayed, filtered by next-charge window:

```
GET /payments/api/v1/subscriptions
    ?status=DELAYED
    &status=OVERDUE
    &date_next_charge=<from_ms>
    &end_date_next_charge=<to_ms>
```

Cancelled, filtered by cancellation window:

```
GET /payments/api/v1/subscriptions
    ?status=CANCELLED_BY_CUSTOMER
    &status=CANCELLED_BY_SELLER
    &status=CANCELLED_BY_ADMIN
    &cancelation_date=<from_ms>
    &end_cancelation_date=<to_ms>
```

Valid `status` values: `ACTIVE`, `INACTIVE`, `DELAYED`, `OVERDUE`, `STARTED`,
`EXPIRED`, `CANCELLED_BY_CUSTOMER`, `CANCELLED_BY_SELLER`, `CANCELLED_BY_ADMIN`.

Each item gives you `subscriber_code`, `subscription_id`, `status`, `plan`,
`date_next_charge` and the latest `transaction`.

**Paginate.** Read `page_info.next_page_token` from the response and send it back as
`page_token`; do not construct cursors yourself (an invented cursor returns
`400 invalid_token`). `max_results` maxes at 500 on this endpoint.

If the window is wide and you get a `502 internal_server_error`, the query exceeded
Hotmart's 30-second database budget — narrow the date range or add filters rather
than retrying the same call.

## Step 2 — get the purchase history for one subscriber

```
GET /payments/api/v1/subscriptions/:subscriber_code/purchases
```

Use it to see which recurrences settled and which did not.

## Step 3 — get the buyer contact behind a transaction

```
GET /payments/api/v1/sales/users?transaction=<transaction_code>
```

Transaction codes are `HP`-prefixed, e.g. `HP17715690036014`.

Trim the payload with Custom Response when you only need contact fields:

```
GET /payments/api/v1/sales/users?transaction=<code>&select=name,email
```

## Step 4 — check members-area access (optional)

If the product is hosted in Hotmart Club:

```
GET /club/api/v1/users?subdomain=<club_subdomain>&email=<buyer_email>
```

`subdomain` is required on Club v1 calls.

## Rules

- **These reads are safe; the writes on this surface are not.**
  `POST /payments/api/v1/subscriptions/:subscriber_code/cancel`,
  `POST .../reactivate` and `PATCH .../:subscriber_code` change a customer's billing.
  Hotmart documents **no idempotency key and no retry-safety contract**, so a retried
  write may charge or cancel twice. Never auto-retry them; get explicit confirmation
  before calling them.
- Buyer records contain personal data (name, email, phone, national ID document,
  address). Only request the fields you need, with `select`.
- 500 requests/minute, reads and writes combined.
