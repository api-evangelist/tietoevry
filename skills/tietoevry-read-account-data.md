---
name: tietoevry-read-account-data
description: Establish a PSD2 AIS consent with a Tietoevry-powered bank, get the customer through Strong
  Customer Authentication, then read their accounts, balances and transactions.
api: tietoevry-openbanking-xs2a
generated: '2026-09-02'
method: generated
source: Grounded in openapi/tietoevry-tieto-xs2a-accounts.v1_3.yaml and the provider's own worked example in
  openapi/tietoevry-how-to-instruction.yaml. Every operationId below is copied from the published contract.
operations:
  - postConsent
  - getConsentStatus
  - getConsent
  - getAccounts
  - getAccount
  - getAccountBalances
  - getAccountTransactions
  - deleteConsent
---

# Read account data through Tietoevry Open Banking

Account data is unreachable without a consent the customer (the PSU) has personally authorised. Consent
first, always.

## Before you start

- Register an application in the portal and copy its API key. Send it as `X-API-Key` on **every** call.
- Send a fresh UUID as `X-Request-ID` on every call. It is a correlation id, **not** an idempotency key —
  resending it does not replay a stored response.
- Base URL is one host with the environment in the path:
  `https://openbanking.api.tietoevry.com/{sandbox|live}/xs2a/v1.3/...`
- Live access additionally needs an eIDAS QWAC client certificate and a PSD2 AISP licence. Sandbox needs
  neither.

## Step 1 — establish the consent (`postConsent`)

`POST /{sandbox|live}/xs2a/v1.3/consents`

Send `TPP-Redirect-URI` (where the customer lands after approving) and a body carrying:

- `access` — which of `accounts`, `balances`, `transactions` you are asking for. Ask for the least you need.
- `recurringIndicator` — `true` for ongoing access, `false` for one shot.
- `validUntil` — the consent's expiry. Recurring consents are capped at 90 days, or 180 days where the RTS
  Article 10a SCA exemption applies; set the date accordingly.
- `frequencyPerDay` — **your own daily read quota.** Exceeding it returns `429 Consent access exceeded`, and
  no `Retry-After` is sent, so pick a number that covers your polling before you create the consent.
- `combinedServiceIndicator` — `false`; combined services are not supported.

The response carries `consentId`, `consentStatus: received`, `authorisationId` and, in `_links.scaRedirect`,
the URL the customer must open.

## Step 2 — get the customer through SCA

Send them to `_links.scaRedirect`. They authenticate with their bank and approve. They are then returned to
your `TPP-Redirect-URI`.

## Step 3 — confirm before you read (`getConsentStatus`)

`GET /{sandbox|live}/xs2a/v1.3/consents/{consent-id}/status`

Do not skip this. Read until `consentStatus` is `valid`. If it comes back `rejected`, `expired` or
`revokedByPsu`, the consent is dead and you must create a new one — there is no repair path.

## Step 4 — read (`getAccounts`, `getAccountBalances`, `getAccountTransactions`)

Every read carries `Consent-ID: {consentId}` alongside `X-API-Key` and `X-Request-ID`.

- `GET /accounts?withBalance=true` — the account list, balances embedded.
- `GET /accounts/{account-id}/balances`
- `GET /accounts/{account-id}/transactions?dateFrom=...&bookingStatus=both` — returns a report split into
  `booked` and `pending`. Page by following `_links.next`, not by inventing offsets.

Use the `resourceId` from the account list as `{account-id}`; identifiers are opaque UUIDs with no type
prefix.

## Step 5 — clean up (`deleteConsent`)

`DELETE /{sandbox|live}/xs2a/v1.3/consents/{consent-id}` when you no longer need access. The consent moves to
`terminatedByTpp`. This is reversible only by asking the customer for a new consent.

## Error handling

| Status | Meaning | What to do |
|---|---|---|
| 400 | Missing or malformed header, body or query parameter | Check `X-API-Key`, `X-Request-ID` (must be a UUID) and `Consent-ID` |
| 401 | Consent expired or not valid for this resource | Create a new consent and re-run SCA |
| 403 | Consent not authorised by the customer | They have not finished the `scaRedirect` flow |
| 404 | Consent or account not found | Check the id you are sending |
| 429 | `frequencyPerDay` exhausted | Wait for the day to roll over; no reset time is published |

Errors arrive as `{ transactionStatus, tppMessages: [{ category, code, text }], psuMessage }` — this API does
**not** use RFC 9457 problem+json. `psuMessage` is the only field intended for display to the customer.
