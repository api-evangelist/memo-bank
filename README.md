# Memo Bank

<!-- API-EVANGELIST-PROVENANCE:BEGIN -->
> ### About this repository
>
> **This is not our API.** This repository is an independent, third-party profile of a company's
> **publicly available** API surface, maintained by [API Evangelist](https://apievangelist.com).
> API Evangelist does not operate, host, resell, or support this company's APIs, and is not
> affiliated with or endorsed by the company unless stated on the profile.
>
> **Where the information came from.** Everything here is assembled from material a member of the
> public can reach with a browser and no credentials — the company's own website, developer portal
> and documentation, the specifications it publishes for public use (OpenAPI, AsyncAPI, JSON Schema,
> `apis.json`, `llms.txt` and similar), its public repositories, and its public status, pricing and
> changelog pages. **Nothing here is obtained by breaching a system, defeating an access control, or
> using credentials of any kind.**
>
> **The rating is an independent assessment.** The Kin Score and Agent Readiness rating are
> independently calculated scores of a company's *public* API artifacts, produced by API Evangelist
> against a published rubric. They are not certifications, endorsements, security assessments, or
> audits, and they score published artifacts — not the quality, safety, or security of the software.
>
> **Corrections, re-scores, and removal are free.** No partnership, contract, or purchase is
> required, and you do not need to justify the request.
>
> - **Something wrong?** Open an issue on this repository, or email
>   [info@apievangelist.com](mailto:info@apievangelist.com).
> - **Published something new?** Ask for a re-score and we will re-run the rating.
> - **Want the listing taken down?** Say so and we will honor it. The profile is reduced to your
>   company name, a factual description, and a link to your own site, and the company is recorded as
>   **unrated** — never scored zero for having asked.
>
> **Response times.** Acknowledgement within **one business day**; removal or restriction within
> **two business days**; corrections and re-scores within **five business days**.
>
> **Not from the company, and here with a question?** You are welcome here — we would rather be the
> front line and point you the right way than have a good report go nowhere. What this repository
> can answer is narrow, though, so it is worth knowing who you are actually looking for:
>
> - **A question about how the API works, an account, billing, or a bug in the service** — that is
>   the company's own support, not us. We profile this API; we do not operate it and cannot see
>   your account.
> - **A bug in an open-source project we only catalog** — file it on that project's own repository.
>   This has happened with a real and correct bug report that reached us instead of the people who
>   could fix it, which helped nobody.
> - **Anything about this listing itself** — the description, the tags, the rating, a missing or
>   wrong artifact — is ours. Open an issue here.
> - **Not sure, or something general about API Evangelist or APIs.io** — open an issue on the
>   [APIs.io Inbox](https://github.com/api-search/inbox) and we will route it.
>
> **This repository contains no software, and we will never ask you to download anything.** There is
> no build, release, installer, or binary here — only text and machine-readable API descriptions, so
> there is nothing here that can be "corrupt" or need "repairing". Any issue, comment, or email
> claiming otherwise and offering a download link is not from us and is hostile. Do not follow the
> link; it is a lure. Report it to GitHub and, if you like, tell us at
> [info@apievangelist.com](mailto:info@apievangelist.com) so we can take it down.
>
> **On a security or compliance team?** Email
> [info@apievangelist.com](mailto:info@apievangelist.com) with *security* in the subject line and
> you will get a person, not a form. We will tell you exactly which public URLs this profile was
> built from so your team can see the same surface we did, and we will take the listing down on
> request while you work through it.
>
> Full detail: **[Where this data comes from](https://apievangelist.com/about/where-our-data-comes-from)**
<!-- API-EVANGELIST-PROVENANCE:END -->

Memo Bank is an independent French commercial bank, accredited as a credit institution by the European Central Bank and supervised by the ACPR, that serves small and mid-sized companies with current accounts, loans, savings and an API-first payments platform. Its Premium Bank API is a REST/JSON contract authenticated with RS256-signed JWT/JWS request tokens, covering accounts, transactions, bi-directional virtual IBANs, SEPA credit transfers, SEPA Direct Debit collections and mandate signature requests, SWIFT/RTGS wire transfers, bulk payment batches, attachments and account assessments, with 34 webhook event types for asynchronous reconciliation. A Marketplace API adds an OAuth 2.0 layer so third-party developers can act on customer resources, a NextGenPSD2 API implements the Berlin Group AIS/PIS interface for licensed third-party providers, and a hosted MCP server exposes read-only banking data plus attachment upload to AI assistants.

**Areas:** Company · Fintech Insurtech · Banking · Business Banking · Payments · SEPA · SEPA Direct Debit · virtual-iban · Wire Transfers · Open Banking · PSD2 · Berlin Group

## Memo Bank on APIs.io

Everything profiled for Memo Bank is published on APIs.io rather than in this repository. Start at the provider page — **[apis.io/providers/memo-bank/](https://apis.io/providers/memo-bank/)** — or go straight to one of the APIs or properties below.

### APIs (15)

- [Memo Bank Account assessments API](https://apis.io/apis/memo-bank/memo-bank-account-assessments-api/) — Account assessments allow you to assess SEPA counterparty accounts before initiating transactions with them. An assessment provides: * **Risk indicators**:…
- [Memo Bank Accounts API](https://apis.io/apis/memo-bank/memo-bank-accounts-api/) — Accounts are any bank account that your company owns: current account, ring-fenced account, settlement account, specially dedicated account, trust account,…
- [Memo Bank AIS API](https://apis.io/apis/memo-bank/memo-bank-ais-api/) — Account Information Service. Consent is given using the **OAuth2 as a pre-step** authentication flow described in NextGenPSD2 specifications. This means you…
- [Memo Bank Attachments API](https://apis.io/apis/memo-bank/memo-bank-attachments-api/) — Documents attached to transactions.
- [Memo Bank Collections API](https://apis.io/apis/memo-bank/memo-bank-collections-api/) — Collections are SEPA direct debit collections, including SEPA CORE direct debits and SEPA B2B direct debits. To initiate a direct debit, a **mandate**…
- [Memo Bank Mandate signature requests API](https://apis.io/apis/memo-bank/memo-bank-mandate-signature-requests-api/) — Mandate signature requests are a way to prepare and send collection mandates for signature. The mandate debtor receives an email with a link so they can…
- [Memo Bank PIS API](https://apis.io/apis/memo-bank/memo-bank-pis-api/) — Payment Initiation Service. Payment initiation is done using the **integrated OAuth2** authentication flow described in NextGenPSD2 specifications. This…
- [Memo Bank Sandbox API](https://apis.io/apis/memo-bank/memo-bank-sandbox-api/) — Sandbox only endpoints.
- [Memo Bank Transactions API](https://apis.io/apis/memo-bank/memo-bank-transactions-api/) — Transactions are any debit and credit operations on an account. <img src="https://assets.memo.bank/memobankapi/transactions-lifecycle-api-v2.png"…
- [Memo Bank Transfers API](https://apis.io/apis/memo-bank/memo-bank-transfers-api/) — Transfers are transfers within the SEPA-zone, including SEPA standard transfers, SEPA instant transfers and Target 2 transfers. They can be initiated…
- [Memo Bank Webhook API](https://apis.io/apis/memo-bank/memo-bank-webhook-api/) — # Events When something interesting happens on your Memo Bank workspace, such as a new transaction being created, Memo Bank can reach out to your…
- [Memo Bank Webhooks API](https://apis.io/apis/memo-bank/memo-bank-webhooks-api/) — Manage webhooks for your application. Webhooks allow you to receive real-time notifications when events occur on your Memo Bank workspace. Please refer to…
- [Memo Bank Wire Transfers API](https://apis.io/apis/memo-bank/memo-bank-wire-transfers-api/) — Wire transfers are transfers going through Swift. They allow you to send funds outside the SEPA network and to transfer money in foreign currencies.…
- [Memo Bank IBA Ns API](https://apis.io/apis/memo-bank/memo-bank-ibans-api/) — IBANs are identifiers for bank accounts. There are two types of IBANs at Memo Bank: * Main IBANs, which act as primary identifiers for a bank account. There…
- [Memo Bank O Auth2 API](https://apis.io/apis/memo-bank/memo-bank-oauth2-api/) — OAuth2 token management endpoints.

### Published properties

Each property below is a page on APIs.io carrying the machine-readable artifact and what it says.

#### AsyncAPI (1)

Event-driven and webhook contracts.

- [Memo Bank Webhooks](https://apis.io/asyncapis/memo-bank/memo-bank-webhooks/)

#### MCP servers (1)

Model Context Protocol servers that expose the API to agents.

- [memo-bank-mcp.yml](https://apis.io/servers/memo-bank/memo-bank-mcp-yml/)

#### Agentic access (1)

Which operations an agent can call, which act, and which need a human in the loop.

- [Memo Bank Agentic Access](https://apis.io/agentic-access/memo-bank/memo-bank-agentic-access/)

#### JSON Schema (44)

The data models the API reads and writes.

- [Account](https://apis.io/schemas/memo-bank/memo-bank-account/)
- [AccountPage](https://apis.io/schemas/memo-bank/memo-bank-account-page/)
- [Attachment](https://apis.io/schemas/memo-bank/memo-bank-attachment/)
- [AttachmentPage](https://apis.io/schemas/memo-bank/memo-bank-attachment-page/)
- [AuthorizationCodeGrant](https://apis.io/schemas/memo-bank/memo-bank-authorization-code-grant/)
- [BulkCollections](https://apis.io/schemas/memo-bank/memo-bank-bulk-collections/)
- [BulkPaymentInitiationJson](https://apis.io/schemas/memo-bank/memo-bank-bulk-payment-initiation-json/)
- [BulkPaymentInitiationWithStatusResponse](https://apis.io/schemas/memo-bank/memo-bank-bulk-payment-initiation-with-status-response/)
- [BulkTransfers](https://apis.io/schemas/memo-bank/memo-bank-bulk-transfers/)
- [Collection](https://apis.io/schemas/memo-bank/memo-bank-collection/)
- [CollectionPage](https://apis.io/schemas/memo-bank/memo-bank-collection-page/)
- [CompletedAccountAssessment](https://apis.io/schemas/memo-bank/memo-bank-completed-account-assessment/)
- [CreateAccountAssessment](https://apis.io/schemas/memo-bank/memo-bank-create-account-assessment/)
- [CreateBulkCollections](https://apis.io/schemas/memo-bank/memo-bank-create-bulk-collections/)
- [CreateBulkTransfers](https://apis.io/schemas/memo-bank/memo-bank-create-bulk-transfers/)
- [CreateCollection](https://apis.io/schemas/memo-bank/memo-bank-create-collection/)
- [CreateEmailMandateSignatureRequest](https://apis.io/schemas/memo-bank/memo-bank-create-email-mandate-signature-request/)
- [CreateIban](https://apis.io/schemas/memo-bank/memo-bank-create-iban/)
- [CreateIncomingCollection](https://apis.io/schemas/memo-bank/memo-bank-create-incoming-collection/)
- [CreateIncomingTransfer](https://apis.io/schemas/memo-bank/memo-bank-create-incoming-transfer/)
- [CreateRedirectMandateSignatureRequest](https://apis.io/schemas/memo-bank/memo-bank-create-redirect-mandate-signature-request/)
- [CreateTransferV2](https://apis.io/schemas/memo-bank/memo-bank-create-transfer-v2/)
- [CreateWebhook](https://apis.io/schemas/memo-bank/memo-bank-create-webhook/)
- [CreateWireTransfer](https://apis.io/schemas/memo-bank/memo-bank-create-wire-transfer/)
- [EmailMandateSignatureRequest](https://apis.io/schemas/memo-bank/memo-bank-email-mandate-signature-request/)
- …and 19 more published on APIs.io

#### JSON-LD (1)

Linked-data contexts for the provider's vocabulary.

- [Memo Bank Context](https://apis.io/jsonld/memo-bank/memo-bank-context/)

#### Overlays (3)

OpenAPI overlays, including conversational phrasing for each operation.

- [Memo Bank Marketplace Api Overlay](https://apis.io/overlays/memo-bank/memo-bank-marketplace-api-overlay/)
- [Memo Bank Nextgenpsd2 Api Overlay](https://apis.io/overlays/memo-bank/memo-bank-nextgenpsd2-api-overlay/)
- [Memo Bank Premium Bank Api Overlay](https://apis.io/overlays/memo-bank/memo-bank-premium-bank-api-overlay/)

#### Governance rules (1)

Spectral rulesets for the provider's contracts.

- [Memo Bank API Rules](https://apis.io/rules/memo-bank/memo-bank-rules/)

#### Security (3)

Authentication, trust centre, and vulnerability disclosure.

- [Memo Bank Authentication](https://apis.io/security/memo-bank/memo-bank-authentication/)
- [Memo Bank Domain Security](https://apis.io/security/memo-bank/memo-bank-domain-security/)
- [Memo Bank Vulnerability Disclosure](https://apis.io/security/memo-bank/memo-bank-vulnerability-disclosure/)

#### OAuth scopes (1)

The permissions the API asks for.

- [Memo Bank Scopes](https://apis.io/scopes/memo-bank/memo-bank-scopes/)

#### Plans and pricing (1)

Published plans, tiers and prices.

- [Memo Bank Plans Pricing](https://apis.io/plans/memo-bank/memo-bank-plans-pricing/)

#### Rate limits (1)

Published request limits.

- [Memo Bank Rate Limits](https://apis.io/rate-limits/memo-bank/memo-bank-rate-limits/)

## Where to find this API

Links to the company's own pages. This repository publishes no copy of their specifications — the canonical document is theirs and lives at the URLs below.

- [Website](https://memo.bank/)
- [Documentation](https://docs.api.memo.bank/)
- [Getting started](https://docs.api.memo.bank/topic/topic-getting-started)
- [API reference](https://docs.api.memo.bank/version-2/)
- [Pricing](https://memo.bank/en/pricing/)
- [Sign up](https://memo.bank/en/sign-up/)
- [Support](https://aide.memo.bank/)
- [Changelog](https://docs.api.memo.bank/changes)
- [Blog](https://memo.bank/en/magazine/)
- [Compliance](https://memo.bank/en/about/security-operations/)
- [Terms of service](https://memo.bank/en/agreement/)
- [Privacy policy](https://memo.bank/en/personal-data/)
- [GitHub](https://github.com/memobank)
- [Status](https://status.memo.bank/)

## API Evangelist rating

**Kin Score 68.8** — exemplar · rubric 0.25.0 · scored 2026-10-23

| Facet | Score |
|---|---|
| Discoverability | 81.7 |
| Contract quality | 68.9 |
| Contract governance | 35.6 |
| Operational transparency | 52.6 |
| Developer ergonomics | 66.1 |
| Access clarity | 73.7 |

**Agent readiness 62.3** — agent-native

| Agent-readiness dimension | Result |
|---|---|
| Spec Presence | yes |
| Agentic Access | derived (partial credit) |
| Reversibility Documented | documented (partial credit) |
| MCP Server | verified |
| Auth Clarity | served (partial credit) |
| Idempotency | documented (partial credit) |
| Error Semantics | documented (partial credit) |
| OpenAPI Examples | partial |
| Rate Limit Signal | documented (partial credit) |
| Event Surface Described | yes |
| Agent Skills | derived (partial credit) |
| Well Known Catalog | — |
| Consent Identity | — |
| Agent Card | — |
| Dry Run Mode | — |
| Delegated Identity | served (partial credit) |
| Protected Resource Metadata | verified |
| Dynamic Client Registration | yes |
| Agentic Commerce | — |
| Pagination Ratio | paginated |
| Operation Distinctiveness | distinct |

**Regulatory alignment 54.3** — Banking & Open Finance

The rubric is published at [github.com/api-evangelist/kin-score](https://github.com/api-evangelist/kin-score). The per-check breakdown behind these numbers — which checks were earned, partial or missed — along with score history and how this provider compares to its peers, is available at [apis.io/providers/memo-bank/](https://apis.io/providers/memo-bank/).

## How this profile is made

API Evangelist profiles a company's public API surface the way a developer would find it: the website, developer portal and documentation, the specifications it publishes, its public repositories, and its pricing, status and changelog pages. From that it derives the properties above — contracts split per API, workflows, schemas, collections, governance rules, plans and rate limits — and scores them against the published [Kin Score rubric](https://github.com/api-evangelist/kin-score).

The artifacts are published on [APIs.io](https://apis.io), not in this repository. This repository is the public record for Memo Bank: the place to say something is wrong, missing or out of date, and to ask for a re-score or a removal.

## Open an issue

- **Something about this listing** — a wrong description, a missing API, a stale artifact or a question about the rating: [open an issue on this repository](https://github.com/api-evangelist/memo-bank/issues/new).
- **A new API, or one that is not in the catalog yet** — [submit it on APIs.io](https://apis.io/add/).
- **Anything else** — [the APIs.io inbox](https://github.com/api-search/inbox/issues/new) or [info@apievangelist.com](mailto:info@apievangelist.com).
