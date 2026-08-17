---
name: Send a cross-border wire transfer
description: >-
  Initiate a SWIFT/RTGS wire transfer with Memo Bank, satisfy the compliance attachment hold that can block
  it, and interpret the wire-specific failure codes and UETR tracking reference.
api: openapi/memo-bank-premium-bank-api-openapi.yml
operations:
  - listAccounts
  - listIbans
  - createWireTransfer
  - getWireTransfer
  - createWireTransferAttachment
  - getProofOfWireTransfer
  - createWebhook
generated: '2026-08-17'
method: generated
source:
  - openapi/memo-bank-premium-bank-api-openapi.yml
---

# Send a cross-border wire transfer

Wire transfers go outside SEPA, through correspondent banks, in currencies that may not match the account.
Two things make them different from a SEPA transfer and both need handling: **the beneficiary may not have an
IBAN**, and **the bank may hold the payment pending documentation**.

Authenticate as described in `memo-bank-authenticate-a-request.md`.

## 1. Source account

`listAccounts` / `listIbans`. Use a `current_account`; paying an external beneficiary from a
`booster_account` fails with `transfer_from_saving_account_to_external_beneficiary`, and a virtual IBAN
cannot be used for transfers between your own accounts.

Memo Bank also offers a USD account product, which matters here because a currency mismatch between the
instructed currency and the account is a documented failure mode
(`invalid_currency_for_account`).

## 2. Identify the beneficiary correctly

`createWireTransfer` takes `beneficiary_account_identifier`, a **polymorphic** field with three variants —
this is the part people get wrong:

| Variant | Use for |
|---|---|
| `iban_and_bic` | IBAN geographies (Europe and beyond) |
| `account_number_and_bic` | Account number plus SWIFT BIC |
| `account_number_and_routing_code` | US-style routing codes (ABA) and similar |

Pick the variant that matches the destination country. Mismatches produce
`country_and_account_identifier_inconsistency`, `invalid_account_identifier_for_country`,
`invalid_routing_code_for_country` or `iban_and_bic_inconsistency`.

Some corridors additionally require a **beneficiary LEI** (`missing_beneficiary_lei`).

## 3. Initiate

`createWireTransfer` — `POST /v2/wire_transfers`

Required: `instructed_amount` (integer minor units), `instructed_currency` (ISO 4217), `local_iban`,
`beneficiary_account_identifier`, `message`, `reference`.

Note the field naming: **`instructed_amount` / `instructed_currency`**, not `amount` / `currency`. The
"instructed" prefix is deliberate — it is the currency you are instructing, which may differ from what
settles.

Send an **`Idempotency-Key`** (V4 UUID). Retry with the same key on a network error, `5XX`, `409` or `429`;
never on other `4XX`.

`message` is **visible to all involved parties** and on a wire it often carries the payment purpose the
correspondent bank needs. `internal_note`, `custom_id` and `custom_metadata` are workspace-only and not
transmitted.

Store the returned `id`, `reference` and — once available — the **`uetr`**, the SWIFT Unique End-to-end
Transaction Reference. That is the identifier a beneficiary's bank will ask for when tracing a payment.

## 4. Handle the compliance hold

`WireTransfer.status` has a state SEPA transfers do not:

`pending` → **`pending_attachment_required`** → `authorized` → `confirmed`, or `returned` / `failed`.

`pending_attachment_required` means **the bank is blocking the payment until you supply supporting
documentation** — typically an invoice or contract evidencing the payment's purpose. This is a compliance
control encoded directly in the state machine, and cross-border payments hit it routinely.

Handle the **`wire_transfer_attachment_required`** webhook event, then call
`createWireTransferAttachment` (`POST /v2/wire_transfers/{id}/attachments`) to upload the document.

> Build this path before you go live. A wire sitting in `pending_attachment_required` with nobody watching
> the event is a payment that silently never arrives, and it is the single most common operational surprise
> on this endpoint.

Note this is distinct from transaction attachments (`createAttachment`): these attach to the **wire transfer
itself**, for compliance, not to a settled transaction for bookkeeping.

## 5. Track to completion

Subscribe with `createWebhook` and handle:

- `wire_transfer_attachment_required`
- `wire_transfer_confirmed`
- `wire_transfer_returned`
- `wire_transfer_canceled`
- `wire_transfer_failed`

Deduplicate on `Event.id`, then call `getWireTransfer`.

There is **no cancel operation** for a wire transfer — unlike `cancelTransfer` and `cancelCollection`. Once
initiated you can only wait for a terminal state. Validate hard before sending.

## 6. Failure codes — read this caveat

`WireTransfer.failure_code` has **28 values and Memo Bank documents none of them**. Its `TransferV2` and
`Collection` siblings document every code inline; the wire enum ships with a one-line description and no
per-code meanings.

The codes are catalogued verbatim in `errors/memo-bank-decline-codes.yml` with explicit `meaning: null`
rather than guesses. Practically, this means:

- Log the raw `failure_code` and surface it to a human. Do not build automated retry logic that depends on
  interpreting these codes, because the semantics are unpublished.
- Ask your relationship manager for the meanings of the codes you actually encounter, and record them.
- Codes that also appear on `TransferV2` (`insufficient_funds`, `beneficiary_bank_refusal`,
  `intermediary_system_error`, `memo_error`, `memo_refusal`, `execution_failure`) most likely carry the same
  meaning documented there, but Memo Bank does not state this — treat it as a working assumption, not fact.

Wire-specific codes worth anticipating: `country_unavailable`, `currency_unavailable`, `amount_too_low`,
`amount_too_high`, `wire_transfer_not_authorized_for_beneficiary`.

## 7. Prove it

`getProofOfWireTransfer` generates a proof-of-transfer document — routinely requested on cross-border
payments where the beneficiary's bank has not yet credited the funds.
