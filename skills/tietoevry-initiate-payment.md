---
name: tietoevry-initiate-payment
description: Initiate a SEPA credit transfer through the Tietoevry PSD2 payment initiation service, get it
  authorised by the payer, follow it to settlement, and know exactly which reversal path is still open.
api: tietoevry-openbanking-xs2a
generated: '2026-09-02'
method: generated
source: Grounded in openapi/tietoevry-tieto-xs2a-payments.v1_3.yaml,
  openapi/tietoevry-tieto-xs2a-recall-payments.v1_3.yaml and the provider's worked example in
  openapi/tietoevry-how-to-instruction.yaml. Every operationId below is copied from the published contract.
operations:
  - initiateJSONPayment
  - getJSONPaymentStatus
  - getJSONPayment
  - paymentsStartAuthorization
  - paymentsGetAuthorizationStatus
  - deletePayment
  - initiateJSONPaymentsRecall
  - getJSONPaymentRecallStatus
---

# Initiate a payment through Tietoevry Open Banking

**This skill moves money. Read the reversibility section before you call anything.**

## Step 1 — initiate (`initiateJSONPayment`)

`POST /{sandbox|live}/xs2a/v1.3/payments/{payment-product}`

`{payment-product}` is a scheme name; `sepa-credit-transfers` is the one the provider's own examples use.
A 404 means this bank does not support the product you named.

Headers: `X-API-Key`, a fresh UUID `X-Request-ID`, `TPP-Redirect-URI`, `TPP-Nok-Redirect-URI` (where to send
the payer on failure) and `PSU-IP-Address`.

Body: `debtorAccount.iban`, `creditorAccount.iban`, `creditorName`, `creditorAddress.country`,
`instructedAmount` (`currency` plus `amount` **as a string**), and
`remittanceInformationUnstructured`.

You get back `paymentId`, `transactionStatus: RCVD`, `authorisationId` and `_links.scaRedirect`.

> There is no idempotency key on this endpoint. If the call times out, **do not resend it.** You may create a
> second payment. Poll `getJSONPaymentStatus` with the `paymentId` if you have one; if you never received a
> response body, treat the outcome as unknown and reconcile out of band.

## Step 2 — authorisation

Send the payer to `_links.scaRedirect`. Track it with `paymentsGetAuthorizationStatus`; `scaStatus` walks
`received` → `psuIdentified` → `psuAuthenticated` → `scaMethodSelected` → `finalised`, or ends at `failed`.

## Step 3 — follow the payment (`getJSONPaymentStatus`)

`GET /payments/{payment-product}/{payment-id}/status`

`transactionStatus` is an ISO 20022 external code. The ones that decide your next move:

| Code | Meaning | Reversal still open |
|---|---|---|
| `RCVD` | Received, not yet authorised | Cancel |
| `ACTC` / `ACCP` / `ACFC` | Validated, accepted | Cancel, usually |
| `ACSP` | Settlement in process | Cancel is likely refused |
| `ACSC` / `ACCC` | Settled | Recall only |
| `PART` | Bulk partially accepted | Inspect members |
| `RJCT` | Rejected — terminal | Nothing; read `tppMessages` |
| `CANC` | Cancelled | Terminal |

## Step 4 — reversal

**Before settlement — cancel (`deletePayment`)**

`DELETE /payments/{payment-product}/{payment-id}`

A `405` means *"The addressed payment is not cancellable e.g. due to cut off time passed or legal
constraints."* Tietoevry does not publish the cut-off time; it belongs to the bank. Never promise a customer
a cancellation window this API does not state.

**After settlement — recall (`initiateJSONPaymentsRecall`, premium)**

`POST /{sandbox|live}/xs2a-premium/v1.3/recall-payments/{payment-product}`

A recall is a fresh payment-shaped request that needs its own SCA from the payer, and it is a *request*, not
a guarantee. Follow it with `getJSONPaymentRecallStatus`.

There is no partial cancellation and no partial recall on this API. Partial reversal exists only on the SEPA
Direct Debits product, as a refund.

## Errors

Same envelope as every other XS2A call: `{ transactionStatus, tppMessages[], psuMessage }`. `400` is a bad
header or body, `401`/`403` are authorisation problems, `404 invalid payment product` means the scheme name
is wrong for this bank.
