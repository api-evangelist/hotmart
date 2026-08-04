---
name: Reconcile Hotmart sales, commissions and refunds
description: Pull the sales history, per-transaction price breakdown and commission split for a Hotmart product over a period, and issue a refund or re-generate a bank payment slip for a single transaction.
api: https://developers.hotmart.com/docs/en/v1/sales/about-sales/
operations:
  - GET /payments/api/v1/sales/history
  - GET /payments/api/v1/sales/summary
  - GET /payments/api/v1/sales/price/details
  - GET /payments/api/v1/sales/commissions
  - GET /payments/api/v1/sales/users
  - PUT /payments/api/v1/sales/:transaction_code/refund
  - PUT /payments/api/v1/sales/:transaction/billet
generated: '2026-08-04'
method: generated
source: https://developers.hotmart.com/docs/en/v1/sales/about-sales/
---

# Reconcile Hotmart sales, commissions and refunds

Authenticate first — see `hotmart-authenticate.md`. Base host
`https://developers.hotmart.com`; sandbox is `https://sandbox.hotmart.com` with the
same paths. Dates are epoch milliseconds.

## Step 1 — the transaction ledger

```
GET /payments/api/v1/sales/history?transaction_status=APPROVED
```

Filter by `product_id`, transaction status and date window. Statuses seen on the
sales/webhook surface: `APPROVED`, `BLOCKED`, `CANCELLED`, `CHARGEBACK`, `COMPLETE`,
`EXPIRED`, `NO_FUNDS`, `OVERDUE`, `PARTIALLY_REFUNDED`, `PRE_ORDER`,
`PRINTED_BILLET`, `PROCESSING_TRANSACTION`, `DISPUTE`, `REFUNDED`, `STARTED`,
`UNDER_ANALISYS`, `WAITING_PAYMENT`.

Paginate with `page_token` / `page_info.next_page_token`; `max_results` up to 500.

## Step 2 — aggregate and break down

```
GET /payments/api/v1/sales/summary?product_id=<id>
GET /payments/api/v1/sales/price/details?transaction_status=CANCELLED&payment_type=CREDIT_CARD
GET /payments/api/v1/sales/commissions?product_id=<id>
```

`commissions[].source` is one of `PRODUCER`, `COPRODUCER`, `AFFILIATE`, `ADDON`.
`currency_value` is an ISO 4217 code (`BRL`, `USD`, …) — always reconcile in the
transaction's own currency, never assume BRL.

`purchase.business_model` (`R`, `A`, `I`) tells you who issues the invoice to the
buyer; it changes the tax treatment of the row.

## Step 3 — who bought it

```
GET /payments/api/v1/sales/users?product_id=<id>
GET /payments/api/v1/sales/users?transaction=<HP…>
```

## Step 4 — act on one transaction

Refund:

```
PUT /payments/api/v1/sales/:transaction_code/refund
```

Re-generate a bank payment slip (boleto):

```
PUT /payments/api/v1/sales/:transaction/billet
```

## Rules

- **A refund is money movement and Hotmart publishes no idempotency contract.** There
  is no `Idempotency-Key` header and no documented replay behaviour. Do not retry a
  refund automatically on a timeout or a 5xx — re-read the transaction with
  `GET /payments/api/v1/sales/history` and check its status before deciding.
- Require explicit human confirmation before calling either `PUT`.
- On `502 internal_server_error`, the query exceeded Hotmart's 30-second database
  budget. Narrow the date range or add filters; do not simply retry.
- Use `select=` (Custom Response) to trim payloads — buyer records carry personal
  data including national ID documents and addresses.
- 500 requests/minute, reads and writes combined; back off on `429`.
