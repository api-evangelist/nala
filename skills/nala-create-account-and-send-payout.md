---
name: Create a payment account and send a payout
description: >-
  Register a recipient payment account (mobile money, bank, or alias) with Rafiki
  by NALA, disburse a payout to it from your local-currency wallet, then confirm
  settlement. Covers Bearer auth, idempotent writes, the async 202 + poll/webhook
  pattern, and the sandbox magic identifiers for simulating failures.
api: openapi/nala-rafiki-openapi.yml
operations:
  - POST /payment-accounts
  - POST /payouts
  - GET /payouts/{id}
source: https://docs.rafiki.com/recipes/create-account-and-send-money
---

# Create a payment account and send a payout (Rafiki by NALA)

Use this skill to move money to a recipient in a supported country. All calls are
REST/JSON over HTTPS against the versioned base URL.

## Environment & auth
- Base URL: `https://rest.sandbox.rafiki-api.com/v1` (sandbox) or
  `https://rest.prod.rafiki-api.com/v1` (production). Test in sandbox first.
- Every request: `Authorization: Bearer {apikey}` (keys are minted per environment
  in the Rafiki portal). See `authentication/nala-authentication.yml`.
- `Content-Type: application/json` on any request with a body, or the API returns
  HTTP 400.

## Idempotency (required on writes)
- Send `X-Idempotency-Key: {uuid-v4}` on **POST /payment-accounts** and
  **POST /payouts**. The server caches the response per URL+key for ~24h and
  replays it on retry; concurrent same-key requests return `IDEMPOTENCY_RACE`
  (HTTP 409). Reuse the same key when retrying a request you are unsure completed.
  See `conventions/nala-conventions.yml`.

## Steps
1. **Create (or get) the recipient's payment account — `POST /payment-accounts`.**
   Provide the account holder plus one of: `mobileMoney` (phone number + operator),
   `bankAccount` (account number + bank), or an `alias` (e.g. UPI in India). This
   is get-or-create: the same details return the same account. Capture the returned
   payment account id.
2. **Create the payout — `POST /payouts`.** Reference the payment account id, the
   sender, and the `amount` (value + currency). The API accepts the payout with
   **HTTP 202** and processes it asynchronously; capture the payout `id`.
3. **Confirm settlement.** Either:
   - **Poll `GET /payouts/{id}`** until the state is terminal, or
   - Subscribe to the `payout.state.updated` webhook (see
     `asyncapi/nala-rafiki-webhooks.yml`) and react to state changes instead of
     polling.

## Errors
Failures use the Rafiki envelope `{ code, message, errors }` (machine-readable
`code`, human `message`, field-level `errors`) — not RFC 9457. See
`errors/nala-problem-types.yml`. Payout-level failures surface a decline/context
code (see `errors/nala-decline-codes.yml`).

## Testing failures in sandbox
In sandbox any valid payload succeeds. To force a failure state, use the published
magic identifiers (see `sandbox/nala-sandbox.yml`), e.g. a mobile-money number
ending `000101` → `PAYMENT_ACCOUNT_INVALID_ACCOUNT_NUMBER`, or bank account
`0390195016650` → generic failure. Never use these values in production.
