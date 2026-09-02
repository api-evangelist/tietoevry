---
name: tietoevry-reverse-sepa-direct-debit
description: Reverse a SEPA Direct Debit through the Tietoevry payments gateway — cancel, refund, reject or
  charge back — picking the right operation for the role you hold and the state the collection is in.
api: tietoevry-sepa-direct-debits
generated: '2026-09-02'
method: generated
source: Grounded entirely in openapi/tietoevry-sepa-direct-debit-api-gateway.yaml (OpenAPI 3.0.1, 15
  operations). Every operationId and reason code below is copied from that contract.
operations:
  - createPayment
  - getPayment
  - cancelPayment
  - createRefund
  - getRefund
  - rejectPayment
  - chargebackPayment
  - getChargeback
---

# Reverse a SEPA Direct Debit

Four reversal paths exist. Which one you may use is decided by **your role** and **how far the collection has
got**. Picking the wrong one gets you a rejection, not a fallback.

Base URL: `https://payments.api.tieto.com/{sandbox|live}/v1/sepadd`

| You are | Collection state | Operation | Reason code |
|---|---|---|---|
| Creditor | Presented, not due | `cancelPayment` | e.g. `MD05` — Collection not due |
| Creditor | Collected | `createRefund` | — |
| Debtor | Incoming, not settled | `rejectPayment` | e.g. `MD01` — No Mandate |
| Debtor | Settled | `chargebackPayment` | e.g. `MD06` — Refund request by end customer |

## Cancel — creditor, before it is due (`cancelPayment`)

`POST /v1/creditor/payments/{payment-id}/actions/cancel`

Send a reason code. The contract's example is `MD05`.

## Refund — creditor, after collection (`createRefund`)

`POST /v1/creditor/payments/{payment-id}/refunds`

Send the original `paymentId` and the amount. The contract is explicit about the ceiling: the amount *"must
be equal to the original amount of the initial Payment, minus all previously made refunds to that initial
Payment"* — so partial refunds are allowed and they accumulate against the original.

Set `Creditor-Notification-URI` on the request. That header is how you find out what happened: it registers
the webhook that will receive `receiveRefundStatus` events for this refund specifically.

Read it back with `getRefund` at `/v1/creditor/payments/{payment-id}/refunds/{refund-id}`.

## Reject — debtor, before settlement (`rejectPayment`)

`POST /v1/debtor/payments/{payment-id}/actions/reject` with a reason code, e.g. `MD01`.

## Chargeback — debtor, after settlement (`chargebackPayment`)

`POST /v1/debtor/payments/{payment-id}/chargebacks` with a reason code, e.g. `MD06`. Read it back with
`getChargeback`.

## Two things this API does not tell you

1. **No deadlines.** Not one of these four operations states a time window, even though the SEPA rulebook
   imposes them. Get the applicable window from the scheme rulebook or your Tietoevry contract — never
   assume one from the API.
2. **Only three reason codes are enumerated.** `ReasonCode` in the published contract carries `MD05` and
   `MD06`; `MD01` appears in prose only. The full SEPA R-transaction set is not published, so build your
   handler to tolerate codes you have never seen.

## Watching for the outcome

Reversals are asynchronous. Register `Creditor-Notification-URI` or `Debtor-Notification-URI` on the request
and handle `receiveRefundStatus` / `receiveChargebackStatus` / `receivePaymentStatus`. Note that no signature
or signing secret is documented for these callbacks, so authenticate them yourself — a secret path segment,
mTLS, or an allowlist — and confirm every notification by reading the object back with `getRefund` or
`getChargeback` before you act on it.
