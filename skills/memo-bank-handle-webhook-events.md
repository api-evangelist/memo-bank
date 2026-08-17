---
name: Handle Memo Bank webhook events
description: >-
  Register a webhook receiver with Memo Bank, authenticate deliveries, deduplicate on the event id, and route
  all 34 event types across the 11 resource types to the right handler.
api: openapi/memo-bank-premium-bank-api-openapi.yml
operations:
  - createWebhook
  - listWebhooks
  - getWebhook
  - deleteWebhook
  - getTransaction
  - getTransfer
  - getWireTransfer
  - getCollection
  - getAccount
  - getIban
  - getAttachment
  - getMandateSignatureRequest
  - getAccountAssessment
  - getTransfersBulk
  - getCollectionsBulk
generated: '2026-08-17'
method: generated
source:
  - openapi/memo-bank-premium-bank-api-openapi.yml
  - https://docs.api.memo.bank/group/webhook-webhook
  - https://docs-marketplace.api.memo.bank/topic/topic-webhooks
---

# Handle Memo Bank webhook events

Every asynchronous outcome in the Memo Bank API arrives as a webhook. Payments confirm, return and fail
hours or days after you initiate them, so **webhooks are not an optimisation over polling here — they are the
only sound way to learn a terminal state.**

Authenticate your API calls as described in `memo-bank-authenticate-a-request.md`.

## 1. Register a receiver

`createWebhook` — `POST /v2/webhooks`

Supply `name` and `url`. **The URL must use HTTPS.**

The response contains a **`bearer_token`**. Treat the entire response as a secret and store the token in your
secret manager.

`listWebhooks`, `getWebhook` and `deleteWebhook` manage them. There is **no update operation** — changing a
delivery URL means delete then re-create, which issues a *new* bearer token. Plan that cutover: run both
endpoints briefly, or accept a gap.

Webhook management endpoints were only added on 2026-03-18; before that they were configured by support.

## 2. Authenticate every delivery

Memo Bank sends the `bearer_token` in the **`Authorization`** header of each delivery. Compare it against
your stored value using a **constant-time comparison**, and reject anything that does not match.

> **Know the limitation.** There is **no HMAC signature over the body** and **no timestamp header**. A shared
> bearer token is all you get, which means:
> - You cannot detect replay of a captured delivery. Idempotent processing (step 3) is your only defence.
> - The token is a long-lived secret readable by anyone who can call `getWebhook`.
>
> Mitigate by terminating TLS properly, rejecting non-HTTPS, and optionally restricting your receiver to
> Memo Bank's egress if you can obtain the ranges from your relationship manager.

## 3. Deduplicate on `Event.id` — mandatory

The payload is the `Event` schema:

| Field | Meaning |
|---|---|
| `id` | UUID of the event — **the deduplication key** |
| `date` | Event creation time, ISO 8601 |
| `event_type` | One of 34 values |
| `resource_type` | One of 11 values |
| `resource_id` | UUID of the affected resource |

Memo Bank states plainly that an event with the same `id` **can be sent multiple times** and must only be
processed once. Persist processed `id`s and drop repeats before any business logic runs. Since deliveries
carry no replay protection, this check is doing real security work, not just tidiness.

Accept the delivery with a `2xx` **fast**, then process asynchronously. Retry behaviour exists but Memo Bank
publishes no backoff schedule or attempt cap, so do not rely on redelivery to cover slow handlers.

Content type is `application/json` or the versioned `application/vnd.memo-bank.v1+json`. Accept both.

## 4. Fetch the resource — the payload is thin

Events carry **only identifiers, never the changed resource**. This is the safer design for banking data, but
it means every handler makes a follow-up call. Route on `resource_type`:

| `resource_type` | Fetch with |
|---|---|
| `account` | `getAccount` |
| `iban` | `getIban` |
| `transaction` | `getTransaction` |
| `transfer` | `getTransfer` |
| `wire_transfer` | `getWireTransfer` |
| `collection` | `getCollection` |
| `attachment` | `getAttachment` |
| `mandate_signature_request` | `getMandateSignatureRequest` |
| `account_assessment` | `getAccountAssessment` |
| `bulk_transfers` | `getTransfersBulk` |
| `bulk_collections` | `getCollectionsBulk` |

This tagging matters because Memo Bank ids are **bare UUIDs with no type prefix** — `resource_type` is the
only thing telling you which endpoint to call.

## 5. The 34 event types

**Accounts:** `account_created`, `account_updated`, `account_closed`

**IBANs:** `iban_created`, `iban_updated`, `iban_deleted`

**Attachments:** `attachment_created`, `attachment_deleted`

**Transactions (ledger):** `transaction_scheduled`, `transaction_authorized`, `transaction_confirmed`,
`transaction_rejected`, `transaction_canceled`

**SEPA transfers:** `transfer_confirmed`, `transfer_returned`, `transfer_canceled`, `transfer_failed`

**Wire transfers:** `wire_transfer_confirmed`, `wire_transfer_returned`, `wire_transfer_canceled`,
`wire_transfer_failed`, **`wire_transfer_attachment_required`**

**Collections:** `collection_confirmed`, `collection_returned`, `collection_canceled`, `collection_failed`

**Bulks:** `bulk_transfers_completed`, `bulk_collections_completed`

**Mandates:** `mandate_signature_request_sent`, `mandate_signature_request_completed`,
`mandate_signature_request_expired`, `mandate_signature_request_deleted`

**Assessments:** `account_assessment_completed`, `account_assessment_failed`

### The three you must not ignore

1. **`wire_transfer_attachment_required`** — the bank is holding a cross-border payment pending
   documentation. Unhandled, the wire silently never arrives. Respond with
   `createWireTransferAttachment`.
2. **`collection_returned`** — money you already counted is being taken back. Check `failure_code`:
   `debtor_bank_insufficient_funds` is worth re-presenting, `debtor_refusal` is not.
3. **`transfer_returned`** / **`wire_transfer_returned`** — a payment you believed complete has reversed.
   `return_transaction_id` links to the compensating transaction.

## 6. Tolerate unknown values — required, not optional

Memo Bank's backwards-compatibility policy **reserves the right to add new `event_type`, `resource_type` and
`TransactionSource` values without a version bump**, and states clients must handle additions gracefully.

So: never `switch` on `event_type` with a throwing default, and never reject an unrecognised
`resource_type`. Log and ignore unknown values. A strict enum parser here is a bug that will fire on a
Tuesday when Memo Bank ships a feature.

## 7. Multi-tenant / Marketplace integrations

Marketplace applications get an extra header, **`X-Memo-Connection-Id`**, identifying which customer
connection the event belongs to — that is how you route an event to the right tenant. The same value comes
back from the OAuth token endpoint as `connection_id`.

Webhooks created with a marketplace access token are scoped to that connection, so you can run a distinct URL
per customer; alternatively Memo Bank can configure one global webhook for the whole application.

A user-deactivated connection makes subsequent API calls fail with `inactive_connection` — stop calling and
re-run the authorization flow.

## 8. Known gaps to design around

- **No event-type filter.** You appear to receive all 34 types, so filter server-side in your handler.
- **No delivery log and no replay endpoint.** If your receiver is down beyond Memo Bank's unpublished retry
  window, those events are gone. **Build a reconciliation sweep** — periodically call `listTransactions` with
  a date window and compare against what you processed. Do not treat webhooks as your only source of truth.
- **No AsyncAPI document**, so this surface is invisible to event-catalogue tooling. The captured catalogue is
  `asyncapi/memo-bank-webhooks.yml`.

## 9. Test it

Register a sandbox webhook, then drive real events with `createIncomingTransfer` and
`createIncomingCollection`. That closed loop needs no Memo Bank involvement and exercises delivery,
authentication and deduplication together.
