---
name: Look up a recipient account before paying
description: >-
  Validate and enrich a recipient's account with Rafiki by NALA before creating a
  payout — list the supported banks for a country and resolve an account number to
  its holder metadata so you can confirm the destination is correct.
api: openapi/nala-rafiki-openapi.yml
operations:
  - GET /banks
  - GET /lookups/{accountNumber}
source: https://docs.rafiki.com/recipes/lookup-bank-account
---

# Look up a recipient account (Rafiki by NALA)

Run this before `POST /payouts` to confirm you are paying the right destination
and to fetch the bank/holder metadata some corridors require.

## Environment & auth
- Base URL: `https://rest.sandbox.rafiki-api.com/v1` (sandbox) or
  `https://rest.prod.rafiki-api.com/v1` (production).
- Every request: `Authorization: Bearer {apikey}`.

## Steps
1. **List banks — `GET /banks`.** Retrieve the financial institutions that own
   payment accounts for the corridor you are paying into; use the returned bank
   identifier when creating a bank payment account. This is a collection endpoint,
   so it is cursor-paginated: pass `paging_after` / `paging_limit` and follow
   `meta.paging.cursors.after` / `meta.paging.next` (see
   `conventions/nala-conventions.yml`).
2. **Resolve the account — `GET /lookups/{accountNumber}`.** Pass the recipient's
   account number (plus the required query context such as bank or country). The
   response returns the resolved account holder metadata, confirming the account
   exists and belongs to whom you expect before you disburse funds.

## Errors
Lookups that fail return the standard envelope `{ code, message, errors }` (see
`errors/nala-problem-types.yml`); an unresolvable account surfaces an
account-number/invalid context code rather than throwing at payout time.

## Notes
Both operations are read-only (`connected`/`read` in
`agentic-access/nala-agentic-access.yml`) and safe to call speculatively; no
idempotency key is needed. In sandbox, the failure-simulating identifiers in
`sandbox/nala-sandbox.yml` (e.g. bank account `0390195016650`) let you exercise
the unresolved-account path.
