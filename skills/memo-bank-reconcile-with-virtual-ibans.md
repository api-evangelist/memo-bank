---
name: Reconcile incoming payments with virtual IBANs
description: >-
  Use Memo Bank virtual IBANs to identify who paid you without parsing payment references - provision one per
  customer, match incoming transactions on local_iban, and attach supporting documents.
api: openapi/memo-bank-premium-bank-api-openapi.yml
operations:
  - listAccounts
  - createIban
  - listIbans
  - getIban
  - updateIban
  - deleteIban
  - listTransactions
  - getTransaction
  - createWebhook
  - listAttachments
  - createAttachment
  - getAttachment
generated: '2026-08-17'
method: generated
source:
  - openapi/memo-bank-premium-bank-api-openapi.yml
  - https://memo.bank/en/product/api/
---

# Reconcile incoming payments with virtual IBANs

The reconciliation problem: a payment lands in your account and you have to work out which customer or
invoice it belongs to. The usual answer is fuzzy-matching a reference field that humans mistype or omit.

Virtual IBANs replace that guesswork with an exact join. You issue **a dedicated IBAN per customer, per
transaction or per department**, and the IBAN the money arrived through *is* the identifier.

Authenticate as described in `memo-bank-authenticate-a-request.md`.

## 1. Pick the parent account

Call `listAccounts`. Virtual IBANs belong to an account (`Iban.account_id`). Use a `current_account`; note
that a `booster_account` (savings) cannot receive collections and has payout restrictions.

## 2. Provision a virtual IBAN

`createIban` — `POST /v2/ibans`

Set a `name` you can trace back to your own record — the customer id, tenant name or invoice number. This is
the field you will read when reconciling, so make it meaningful rather than sequential.

Set `allow_collections` according to whether this IBAN should also be **debitable by SEPA Direct Debit**.
Memo Bank's virtual IBANs are **bi-directional**: the same IBAN can receive transfers and be used as the
creditor IBAN on collections. If you intend to collect through it, it must be `allow_collections: true`.

Send an `Idempotency-Key` (V4 UUID) so a retry does not create a second IBAN for the same customer.

Persist the returned `id` **and** the `iban` string. You need the string for matching (step 4) and the id for
management calls.

Handle the `iban_created` webhook event to confirm provisioning.

## 3. Choose a granularity

| Granularity | Use when | Trade-off |
|---|---|---|
| Per customer | Recurring billing, subscriptions | Fewest IBANs, still an exact join |
| Per invoice/transaction | One-off or high-value payments needing certainty | Many IBANs to manage |
| Per department or entity | Internal cost allocation, group treasury | Coarse but simple |

Per customer is the right default. Go per invoice only when a customer may have several open payments at
once and you cannot tolerate ambiguity between them.

## 4. Match incoming transactions

Call `listTransactions`, or better, subscribe to webhooks.

The join key is **`Transaction.local_iban`** — "the IBAN through which this transaction got in or out of the
account". Match it against the `iban` string you stored in step 2.

Useful filters on `listTransactions`:

- `local_iban` — fetch everything that arrived on one virtual IBAN
- `account_id`, `start_date`, `end_date`
- `reference`, `batch_id`, `custom_id`
- `order_by`

Read `direction` to distinguish inbound from outbound, and `status`
(`scheduled` → `authorized` → `confirmed`, or `rejected` / `canceled`). **Reconcile on `confirmed`**, not
on `authorized`.

Money fields are integers in **minor units** with an ISO 4217 `currency`. Never parse them as decimals.

Three dates are exposed and they mean different things:

- `request_date` — when it was requested
- `execution_date` — when processing started
- `accounting_date` — when it was **confirmed**; this is your value date for accounting

`Transaction.source` is a polymorphic discriminator with 44 variants. For inbound reconciliation you mostly
care about `transfer_incoming`, `collection_incoming`, `wire_transfer_incoming` and
`rtgs_transfer_incoming`, plus the `*_return` variants which are reversals, not new money.

> **Pagination.** Use `page_token` + `size`. The `page` parameter was deprecated on 2026-05-13 — it still
> works and has no published sunset date, but do not build on it.

## 5. Subscribe to events instead of polling

`createWebhook` — `POST /v2/webhooks`

The response contains a `bearer_token`. **Treat the whole response as a secret** — Memo Bank sends that token
in the `Authorization` header of every delivery to your endpoint, and verifying it is how you authenticate
the sender. There is no HMAC body signature, so this token is your only check.

Relevant events: `transaction_confirmed`, `transaction_scheduled`, `transaction_authorized`,
`transaction_rejected`, `transaction_canceled`, and `iban_created` / `iban_updated` / `iban_deleted`.

Deduplicate on `Event.id` — Memo Bank documents that the same event may arrive more than once. Events carry
only `resource_type` + `resource_id`, so call `getTransaction` for the detail.

There is **no update operation for a webhook**: changing your delivery URL means `deleteWebhook` then
`createWebhook`, which rotates the bearer token. Plan the cutover.

## 6. Attach supporting documents

Once matched, attach the invoice or receipt to the transaction with `createAttachment`
(`transaction_id`, `filename`, `mime_type`). `listAttachments` and `getAttachment` retrieve them —
`getAttachment` returns a pre-signed download URL. `Transaction.attachment_count` is a denormalised counter
you can read without a second call.

This is the one write operation Memo Bank also exposes through its MCP connector, so an AI assistant can file
a receipt against a transaction (see `mcp/memo-bank-mcp.yml`).

## 7. Lifecycle

`updateIban` renames or re-flags an IBAN. `deleteIban` **soft-deletes** — `Iban.is_deleted` becomes true and
the record remains queryable with `include_deleted`. Historical transactions keep referencing the IBAN
string, so deleting an IBAN never orphans past reconciliation.

Also note `Iban.status` (`active` / `inactive`) is distinct from deletion: status controls whether the IBAN
accepts incoming payments.

## Caveats

- A **virtual IBAN cannot be used to transfer between your own accounts**
  (`transfer_to_owned_account_with_virtual_iban`). Use the account's `main` IBAN for internal movements.
- `Transaction.local_iban` joins on the IBAN **string**, not on `Iban.id`, so keep the string indexed.
- Put your own keys in `custom_id` / `custom_metadata` — they are **not transmitted** on the payment.
  `internal_note` is likewise workspace-only, whereas `message` is visible to all parties.

## Test it

`createIncomingTransfer` and `createIncomingCollection` in the sandbox simulate inbound money on your virtual
IBANs, which is exactly what you need to exercise this whole path end to end without a real counterparty.
