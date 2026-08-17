---
name: Initiate and track a SEPA transfer
description: >-
  Send a single SEPA credit transfer with Memo Bank, retry it safely with an idempotency key, choose the
  standard/instant rail strategy, and follow it to a terminal state through webhooks and failure codes.
api: openapi/memo-bank-premium-bank-api-openapi.yml
operations:
  - listAccounts
  - listIbans
  - createAccountAssessment
  - getAccountAssessment
  - createTransferV2
  - getTransfer
  - cancelTransfer
  - getProofOfTransfer
generated: '2026-08-17'
method: generated
source:
  - https://docs.api.memo.bank/topic/topic-idempotent-requests
  - openapi/memo-bank-premium-bank-api-openapi.yml
---

# Initiate and track a SEPA transfer

This moves **real money irreversibly**. Read the idempotency section before you write the retry logic, not
after.

Authenticate every call as described in `memo-bank-authenticate-a-request.md`.

## 1. Find the source account and IBAN

Call `listAccounts` to get accounts with `id`, `iban`, `balance` (integer minor units), `currency` and
`type`. Pick a `current_account` — you **cannot** pay external beneficiaries from a `booster_account`
(savings), which fails with `transfer_from_saving_account_to_external_beneficiary`.

Call `listIbans` if you need a specific IBAN on that account. Note the constraint that a **virtual IBAN
cannot be used for transfers between your own accounts** (`transfer_to_owned_account_with_virtual_iban`).

## 2. Verify the beneficiary first (recommended)

Before sending money to a new beneficiary, call `createAccountAssessment` and poll `getAccountAssessment`.
It verifies the IBAN and the account-holder identity, matching against a `NameIdentification`,
`SirenIdentification` (French company register) or `LeiIdentification` (global legal entity identifier).

The assessment resolves to `pending`, `completed` or `failed`.

This one extra call pre-empts three failure codes that otherwise cost you a full settlement cycle:
`invalid_beneficiary_iban`, `unreachable_beneficiary_iban` and
`beneficiary_bank_invalid_bank_details`.

## 3. Choose a rail strategy

`createTransferV2` takes a `type_strategy`:

| Strategy | Behaviour |
|---|---|
| `standard_only` | Always standard SEPA |
| `instant_only` | Fail if instant is unavailable |
| `instant_if_available` | Try instant, fall back to standard automatically |
| `rtgs_only` | Use the RTGS/T2 rail |

**Prefer `instant_if_available`** unless you have a business reason not to. It lets the bank downgrade
gracefully and reports the rail actually used in the response `transfer_type`. Using `instant_only` without
checking first is what produces `instant_transfer_not_available`.

## 4. Initiate the transfer — with an idempotency key

`createTransferV2` — `POST /v2/transfers`

Required: `amount` (integer minor units), `currency` (ISO 4217), `local_iban` (your source IBAN),
`beneficiary_iban`, `reference`. Supply `beneficiary_name` for a first-time beneficiary or you get
`missing_new_beneficiary_name`.

Send an **`Idempotency-Key` header** with a V4 UUID. This is not optional in practice — without it a network
timeout leaves you unable to tell whether the money moved, and you cannot safely retry.

```
POST /v2/transfers HTTP/1.1
Host: api.memo.bank
Authorization: Bearer <jwt>
Idempotency-Key: 19b390d1-e7d4-4e27-abe2-49cac9b41ba1
Content-Type: application/json

{ "amount": 125000, "currency": "EUR", "local_iban": "...", "beneficiary_iban": "...",
  "beneficiary_name": "...", "type_strategy": "instant_if_available" }
```

Useful optional fields:

- `message` — **visible to all parties** on the payment
- `internal_note`, `custom_id`, `custom_metadata` — **not transmitted**, visible only in your workspace.
  Put your reconciliation key in `custom_id`.
- `end_to_end_id` — SEPA end-to-end identification
- `scheduled_date` — to post-date the transfer

Persist the returned `id` and `reference` before doing anything else.

## 5. Retry correctly

Reuse the **same** `Idempotency-Key` on:

- a network error with no response
- `5XX`
- `409 Conflict` — the original is still in flight; safe to retry
- `429 Too Many Requests` — wait for `RateLimit-Reset` seconds first

**Do not retry** on any other `4XX`. Memo Bank returns the same result every time, so a retry is pointless.

`422 Unprocessable Entity` means you reused a key with a **different body** — a bug in your code, not a
transient failure. Never "fix" it by generating a new key for the same logical payment; that risks paying
twice.

A replayed response carries `Idempotent-Replayed: true`. Treat it as success, not as a duplicate.

## 6. Follow it to a terminal state

`TransferV2.status` moves through:

`pending` → `scheduled` → `authorized` → `confirmed`, or terminates at `returned`, `canceled` or `failed`.

**Subscribe to webhooks rather than polling.** Register a receiver with `createWebhook` and handle:

- `transfer_confirmed`
- `transfer_returned`
- `transfer_failed`
- `transfer_canceled`

Events carry only `resource_type` + `resource_id`, so call `getTransfer` to fetch state. Deduplicate on
`Event.id` — Memo Bank may deliver the same event more than once.

You can call `cancelTransfer` while the transfer has not yet been executed.

## 7. Handle failure

On `failed`, read `failure_code`. All 22 values are documented in
`errors/memo-bank-decline-codes.yml` with a retry recommendation for each. The distinction that matters:

- **Counterparty/network codes** (`beneficiary_bank_error`, `intermediary_system_error`, `memo_error`) are
  transient — retry.
- **Beneficiary detail codes** (`beneficiary_bank_account_closed`,
  `beneficiary_bank_invalid_bank_details`, `invalid_beneficiary_iban`) need corrected data — never retry
  unchanged.
- **`memo_refusal`** — do not retry; contact your relationship manager.

If the transfer is `returned`, `return_transaction_id` points at the compensating credit transaction, so you
can link the original and its reversal without matching on amount and date.

## 8. Prove it

`getProofOfTransfer` generates a proof-of-transfer document — the artefact to attach to an invoice or hand
to a counterparty who disputes receipt.

## Test it first

The sandbox (`https://api.sandbox.memo.bank`) runs the full endpoint set, and
`createIncomingTransfer` lets you simulate inbound transfers to exercise your reconciliation and webhook
handling. See `sandbox/memo-bank-sandbox.yml`. Note there are **no published test IBANs** and no test
clocks, so you cannot deterministically test a D+1 settlement transition.
