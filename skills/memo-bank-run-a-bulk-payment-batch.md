---
name: Run a bulk payment batch
description: >-
  Submit thousands of Memo Bank transfers or collections as one batch, track aggregate and per-item progress,
  and handle the validation errors that arrive asynchronously in bulk mode but synchronously for single
  payments.
api: openapi/memo-bank-premium-bank-api-openapi.yml
operations:
  - listAccounts
  - listIbans
  - createTransfersBulk
  - getTransfersBulk
  - getTransfersBulkItems
  - createCollectionsBulk
  - getCollectionsBulk
  - getCollectionsBulkItems
  - createWebhook
  - getTransfer
  - getCollection
generated: '2026-08-17'
method: generated
source:
  - openapi/memo-bank-premium-bank-api-openapi.yml
  - https://docs.api.memo.bank/topic/topic-idempotent-requests
---

# Run a bulk payment batch

Bulk is how you replace an EBICS file upload: submit a payroll run, a supplier payment run or a monthly
billing collection in one call.

The one thing that will bite you is **not** the batch mechanics. It is that bulk mode changes *where errors
appear*. Read section 4 before writing any code.

Authenticate as described in `memo-bank-authenticate-a-request.md`.

## 1. Prepare

`listAccounts` for a funded `current_account`; `listIbans` for the source IBAN. For collections, confirm
`allow_collections: true` and that you have a SEPA creditor identifier from your banker.

Check `Account.balance` (integer minor units) against the batch total. `insufficient_funds` is evaluated
per item, so an underfunded account produces a partially-succeeded batch rather than a clean rejection —
usually the worst outcome.

## 2. Submit the batch

- Transfers: `createTransfersBulk` — `POST /v2/transfers/bulks`
- Collections: `createCollectionsBulk` — `POST /v2/collections/bulks`

Send an **`Idempotency-Key`** (V4 UUID). This matters far more here than for a single payment: retrying a
batch without a key risks paying an entire payroll twice. Retry with the **same** key on a network error,
`5XX`, `409` or `429`; never on other `4XX`. A `422` means you reused the key with a different body — fix the
bug, do not mint a new key.

Give every item a `custom_id` carrying your own record key. It is not transmitted on the payment and it is
what lets you join per-item results back to your ledger.

Store the returned bulk `id` immediately.

## 3. Track progress

`getTransfersBulk` / `getCollectionsBulk` return the aggregate:

| Field | Meaning |
|---|---|
| `transfers_total` / `collections_total` | Items in the batch |
| `transfers_confirmed` / `collections_confirmed` | Processed and confirmed |
| `transfers_canceled` / `collections_canceled` | Cancelled before processing |
| `transfers_failed` / `collections_failed` | Processed and failed |
| `status` | `pending` or `completed` |

Note `status` only distinguishes in-flight from done — **`completed` does not mean successful**. A batch where
every item failed is also `completed`. Always compare the counters.

`getTransfersBulkItems` / `getCollectionsBulkItems` list per-item outcomes, paginated with `page_token` +
`size`.

Subscribe with `createWebhook` and handle `bulk_transfers_completed` / `bulk_collections_completed`, plus the
per-item events (`transfer_confirmed`, `transfer_failed`, `collection_confirmed`, `collection_returned` …).
Deduplicate on `Event.id`.

`Transaction.batch_id` links a resulting ledger transaction back to its bulk, and `listTransactions` accepts
`batch_id` as a filter — that is the cleanest way to reconcile a whole run.

## 4. The bulk-versus-single divergence

**This is the section that matters.** Memo Bank documents it explicitly and it is easy to miss.

A class of validation conditions behaves differently depending on how you initiated:

- **Single payment** → returned synchronously as an **HTTP error response**, and the resource is *never
  created*.
- **Inside a bulk** → the item *is* created, then surfaces the same condition asynchronously as a
  **`failure_code`** on that item.

For transfers, these codes are bulk-only as failure codes:

`current_account_not_found`, `instant_transfer_not_available`, `insufficient_funds`,
`invalid_beneficiary_iban`, `maximum_amount_exceeded`, `missing_new_beneficiary_name`,
`new_beneficiary_is_owned_iban`, `transfer_to_same_account`,
`transfer_to_owned_account_with_virtual_iban`, `transfer_from_saving_account_to_external_beneficiary`,
`unreachable_beneficiary_iban`

For collections:

`account_cannot_receive_collections`, `current_account_not_found`, `creditor_is_saving_account`,
`no_sepa_creditor_identifier`, `mandate_info_missing`, `mandate_iban_mismatch`,
`collection_to_same_account`

**Consequence:** you must implement every one of these conditions **twice** — once in your synchronous error
handler for single payments, once in your asynchronous per-item failure handler for bulks. Code that only
handles the synchronous path will process a batch, report `completed`, and silently drop every failed item.

Full meanings and retry guidance: `errors/memo-bank-decline-codes.yml`.

## 5. Retry the failures, not the batch

When a batch completes with `failed > 0`:

1. Pull the failed items with `getTransfersBulkItems` / `getCollectionsBulkItems`.
2. Group by `failure_code`.
3. Retry **only** the transient groups — `beneficiary_bank_error`, `intermediary_system_error`, `memo_error`,
   `debtor_bank_insufficient_funds` (re-present on a schedule).
4. Route data-correction groups (`invalid_beneficiary_iban`, `beneficiary_bank_invalid_bank_details`,
   `mandate_iban_mismatch`, `missing_new_beneficiary_name`) to an operations queue for human fixing.
5. Never blind-retry the whole batch. Build the retry as a **new, smaller batch with a new
   `Idempotency-Key`** containing only the items you decided to retry.

## 6. Pre-flight validation is cheaper than a failed run

Before a large run:

- Verify new beneficiaries with `createAccountAssessment` / `getAccountAssessment` (IBAN and holder identity
  against SIREN, LEI or name). This removes the largest failure category up front.
- Confirm the account balance covers the total.
- For collections, confirm every mandate `reference` is one you already used with that same IBAN.
- Rate limits apply — read `RateLimit-Remaining` and `RateLimit-Reset`, and back off on `429` rather than
  hammering. Memo Bank publishes the headers but no numeric limit, so treat the headers as the source of
  truth (`rate-limits/memo-bank-rate-limits.yml`).

## 7. Test it

Run the batch in the sandbox (`https://api.sandbox.memo.bank`) first, and use `createIncomingTransfer` /
`createIncomingCollection` to simulate the inbound side. Be aware there are no test clocks, so a batch's
settlement-day transition cannot be fast-forwarded (`sandbox/memo-bank-sandbox.yml`).
