---
name: Authenticate a Memo Bank API request
description: >-
  Mint the per-request RS256 JWT that every Memo Bank Premium Bank API call requires, including the
  request-binding claims that make the token single-use. Read this before any other Memo Bank skill - all of
  them depend on it and none repeat it.
api: openapi/memo-bank-premium-bank-api-openapi.yml
operations:
  - listAccounts
generated: '2026-08-17'
method: generated
source:
  - https://docs.api.memo.bank/authentication
  - https://docs.api.memo.bank/topic/topic-getting-started
---

# Authenticate a Memo Bank API request

Memo Bank does not use a static API key. **Every request carries its own freshly signed JWT**, and the
token is cryptographically bound to that one request. A token you signed for `GET /v2/accounts` cannot be
replayed against `POST /v2/transfers`, and a token signed five seconds ago may already be too old.

Get this wrong and every call returns `401 Unauthorized` with no further detail, so implement it carefully
once and reuse it.

## Prerequisites

API access is **not self-serve**. A Memo Bank banker must activate the API feature on the workspace first.
Once activated, an owner or administrator creates an application at <https://client.memo.bank/api> and
receives three things:

1. A **certificate** and its SHA-256 thumbprint
2. A **secret code**
3. An **RSA private key**

Store all three as secrets. The private key never leaves your server.

## Base URLs

| Environment | API base | Web interface |
|---|---|---|
| Production | `https://api.memo.bank` | `https://client.memo.bank` |
| Sandbox | `https://api.sandbox.memo.bank` | `https://client.sandbox.memo.bank` |

There is no test/live key prefix — the environment is selected purely by base URL, so a sandbox credential
pointed at production simply fails to authenticate.

## Build the JWT

**Header:**

| Claim | Value |
|---|---|
| `alg` | `RS256` — RSA-SHA256 is required |
| `typ` | `JWT` |
| `x5t#S256` | SHA-256 thumbprint of your certificate, from the web interface |

**Payload:**

| Claim | Value |
|---|---|
| `sub` | The request method, a space, then the **full path including query parameters** |
| `aud` | The domain you are calling, e.g. `api.memo.bank` |
| `iat` | Creation timestamp. **Only 5 seconds of clock skew is tolerated** |
| `jti` | A unique UUID, **different for every request** |
| `sec` | Your secret code (a Memo Bank custom claim) |
| `dig#S256` | `base64url(sha256(body))` — **only when the request has a body** |

Sign with your RSA private key and send as `Authorization: Bearer <token>`.

## The five failure modes

Each of these returns an indistinguishable `401`. Check them in this order:

1. **Clock drift.** Your `iat` must be within 5 seconds of Memo Bank's server time. Run NTP. This is the
   most common cause and the least obvious.
2. **`sub` mismatch.** It must match the method and path *exactly*, including the query string and its
   parameter order. `GET /v2/transactions?size=50` and `GET /v2/transactions?size=50&order_by=date` need
   different tokens.
3. **Reused `jti`.** Generate a new UUID per request, never per session.
4. **Missing `dig#S256`.** Required whenever there is a body. Hash the exact bytes you transmit — if you
   serialise the JSON twice you may hash a different byte sequence than you send.
5. **Wrong `aud`.** Use the bare domain (`api.memo.bank`), not the full URL and not the sandbox host when
   calling production.

## Verify it works

Call the cheapest read operation. `listAccounts` (`GET /v2/accounts`) has no body, so no `dig#S256` is
needed:

```
GET /v2/accounts HTTP/1.1
Host: api.memo.bank
Authorization: Bearer <jwt>
```

`sub` for this request is exactly `GET /v2/accounts`.

A `200` confirms the whole chain — certificate thumbprint, secret code, signature and clock — is correct.

## Marketplace applications differ

If you are building a **Marketplace** integration acting on another company's behalf, the scheme changes:

- The OAuth 2.0 access token goes in `Authorization`
- Your JWT goes in **`X-Memo-Signature`**
- The JWT payload adds an **`oat#S256`** claim holding `base64url(sha256(access_token))`, binding your
  signature to that specific access token

See `authentication/memo-bank-authentication.yml` for the full Marketplace flow and token lifetimes.

## Note on tooling

Memo Bank publishes **no SDK in any language** (see `packages/memo-bank-packages.yml`). Its docs point at
<https://jwt.io/libraries> for the crypto and leave the claim construction to you. Wrap this logic in one
signing function and unit-test each of the five failure modes above — you are writing the part a vendor SDK
would normally provide.

There is **no `components.securitySchemes` block** in the published OpenAPI even though all 43 operations
reference a `JWT` scheme, so generated clients will not implement any of this for you. The definition is
supplied in `overlays/memo-bank-premium-bank-api-overlay.yaml`.
