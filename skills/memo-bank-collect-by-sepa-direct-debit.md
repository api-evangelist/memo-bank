---
name: Collect by SEPA Direct Debit with a signed mandate
description: >-
  Run the full Memo Bank direct debit lifecycle - obtain a signed SEPA mandate through a signature request,
  schedule single and bulk collections against it, and handle returns including the debtor refusal path.
api: openapi/memo-bank-premium-bank-api-openapi.yml
operations:
  - listAccounts
  - listIbans
  - createMandateSignatureRequest
  - getMandateSignatureRequest
  - listMandateSignatureRequests
  - renewMandateSignatureRequest
  - deleteMandateSignatureRequest
  - createCollection
  - getCollection
  - cancelCollection
  - createCollectionsBulk
  - getCollectionsBulk
  - getCollectionsBulkItems
generated: '2026-08-17'
method: generated
source:
  - openapi/memo-bank-premium-bank-api-openapi.yml
  - https://docs.api.memo.bank/topic/topic-idempotent-requests
---

# Collect by SEPA Direct Debit with a signed mandate

Direct debit **pulls money from someone else's account**. That is only lawful under a mandate the debtor has
signed, so the mandate lifecycle is the substance of this skill and the collection call is the easy part.

Authenticate as described in `memo-bank-authenticate-a-request.md`.

## 0. One-time setup

You need a **SEPA creditor identifier (SCI)**, provisioned by your Memo Bank banker. Without it every
collection in a bulk fails with `no_sepa_creditor_identifier`.

You also need a collection-capable IBAN. Call `listIbans` and check `allow_collections: true`. A
`booster_account` cannot be a creditor (`creditor_is_saving_account`), and an IBAN without
`allow_collections` fails with `account_cannot_receive_collections`.

## 1. Get a signed mandate

Call `createMandateSignatureRequest`. It is polymorphic on `mode`:

| Mode | Behaviour |
|---|---|
| `email` | Memo Bank emails the debtor a signature link |
| `redirect` | You host the journey and redirect the debtor — added 2026-07-03 |

Choose `redirect` if you want the signing step inside your own onboarding flow.

You supply the debtor details (`MandateSignatureRequestDebtor`, including an address — required for non-EEA
SEPA countries, else `missing_debtor_address`) and the mandate `scheme`:

- **`core`** — consumers and general use. The debtor keeps a broad refund right.
- **`b2b`** — business-to-business. No refund right, but the debtor's bank must pre-register the mandate.

Pick `b2b` only when your debtors are businesses that will confirm the mandate with their bank; otherwise
`core` is the safe default. `core` collections are subject to a limit (`core_limit_exceeded`).

Track the request with `getMandateSignatureRequest` or `listMandateSignatureRequests`, and handle these
webhook events:

- `mandate_signature_request_sent`
- `mandate_signature_request_completed` ← you may now collect
- `mandate_signature_request_expired`
- `mandate_signature_request_deleted`

If a request expires, `renewMandateSignatureRequest` re-issues it rather than forcing a fresh one.
`deleteMandateSignatureRequest` withdraws an outstanding request.

> **Modelling note.** Mandates are **not** first-class resources — there is no `listMandates` or
> `getMandate`. A mandate exists only as an embedded `CollectionMandate` (`reference` + `scheme`) on a
> collection. **You must store the mandate reference and scheme yourself**; the API will not give you a list
> of your active mandates later. Plan your data model around this.

## 2. Schedule a collection

`createCollection` — `POST /v2/collections`

Required: `amount` (integer minor units), `currency`, `local_iban` (your collection-capable IBAN),
`scheduled_date`, `reference`, and the `mandate` object (`reference` + `scheme`).

Send an **`Idempotency-Key`** (V4 UUID). Retry with the same key on a network error, `5XX`, `409` or `429`;
never on other `4XX`. See `conventions/memo-bank-conventions.yml`.

Reuse the exact mandate `reference` you already used for that debtor. Reusing a reference with a **different
IBAN** fails with `mandate_iban_mismatch` — a deliberate guard against silently repointing a signed mandate
at another account.

Put your own reconciliation key in `custom_id`; it is not transmitted on the payment. `message` **is**
visible to all parties.

## 3. Or collect in bulk

For a billing run use `createCollectionsBulk`, then `getCollectionsBulk` for aggregate progress
(`collections_total`, `collections_confirmed`, `collections_canceled`, `collections_failed`, and `status` of
`pending` or `completed`) and `getCollectionsBulkItems` for per-item outcomes. Handle
`bulk_collections_completed`.

> **The bulk trap.** Validation problems that a *single* collection reports synchronously as an HTTP error —
> so the resource is never created — instead appear *asynchronously* as a `failure_code` on the individual
> collection inside a bulk. That includes `current_account_not_found`, `no_sepa_creditor_identifier`,
> `mandate_info_missing`, `mandate_iban_mismatch`, `creditor_is_saving_account`,
> `collection_to_same_account` and `account_cannot_receive_collections`. **You must implement each condition
> twice** — once as a synchronous error, once as an async failure code — or bulk failures will pass silently.

## 4. Follow to a terminal state

`Collection.status`: `pending` → `scheduled` → `confirmed`, or `returned` / `canceled` / `failed`.

Webhook events: `collection_confirmed`, `collection_returned`, `collection_canceled`,
`collection_failed`. Deduplicate on `Event.id`, then call `getCollection`.

`cancelCollection` works while the collection is still `scheduled`.

## 5. Handle returns — the part that costs money

A confirmed collection can still be **returned** days later. Read `failure_code`; all 21 values are
documented in `errors/memo-bank-decline-codes.yml`. Three groups behave very differently:

- **`debtor_bank_insufficient_funds`** — the classic soft decline. **Re-present on a schedule**; this is the
  one worth retrying and typically the largest category.
- **`debtor_refusal`** — the debtor exercised their SEPA refund right. This is the direct debit equivalent of
  a chargeback. **Do not re-present.** Resolve commercially, and treat a rising rate here as a
  product/billing problem rather than a payments one.
- **Detail codes** — `debtor_bank_account_closed`, `debtor_bank_invalid_bank_details`,
  `invalid_mandate_iban`: obtain a new mandate with corrected details. The existing mandate is dead.

On a return, `return_transaction_id` points at the compensating debit transaction, so you can link the
original collection to its reversal directly.

Note that bank refusal reasons (`debtor_bank_refusal`) are deliberately opaque — Memo Bank is not told why
either.

## 6. Test it

In the sandbox, `createIncomingCollection` simulates an inbound collection including its mandate and
creditor, letting you exercise `collection_confirmed` and `collection_returned` handling. There is no
documented way to force a specific return reason, so you cannot deterministically test each `failure_code`
branch — see `sandbox/memo-bank-sandbox.yml`.
