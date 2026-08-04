---
name: Authenticate against the Hotmart Developers API
description: Exchange Hotmart developer credentials for an OAuth 2.0 access token and call any Hotmart endpoint with it, in production or in sandbox.
api: https://developers.hotmart.com/docs/en/start/app-auth/
operations:
  - POST https://api-sec-vlc.hotmart.com/security/oauth/token
generated: '2026-08-04'
method: generated
source: https://developers.hotmart.com/docs/en/start/app-auth/
---

# Authenticate against the Hotmart Developers API

Every Hotmart Developers call needs a Bearer access token. Hotmart uses OAuth 2.0
`client_credentials`. There is **no** SDK — call the token endpoint directly.

## Before you start

Credentials are created by a human in the Hotmart platform under
**Tools > Developer Credentials** (<https://app-vlc.hotmart.com/tools/credentials>).
Creating one yields three values: `client_id`, `client_secret`, and a pre-computed
`Basic` token.

The credential's environment is fixed at creation:

- leave the **Type** box blank → production credential
- tick **sandbox** → sandbox credential

A production credential will not authenticate against `sandbox.hotmart.com`, and the
type cannot be changed afterwards — a new credential must be created.

## Step 1 — get an access token

```
POST https://api-sec-vlc.hotmart.com/security/oauth/token
     ?grant_type=client_credentials
     &client_id=:client_id
     &client_secret=:client_secret

Content-Type: application/json
Authorization: Basic :basic
```

The same token endpoint serves both environments; the credential decides which
environment the token is valid for.

The response carries `access_token` and `expires_in`.

## Step 2 — call the API

Send the token on every resource request:

```
Authorization: Bearer <access_token>
```

Pick the host by environment:

| Environment | Host |
|---|---|
| production | `https://developers.hotmart.com` |
| sandbox | `https://sandbox.hotmart.com` |

Paths are identical between the two.

## Step 3 — handle expiry

Only the access token expires; `client_id`, `client_secret` and the `Basic` token do
not rotate on their own.

When the token expires every request returns:

```
401 {"error":"token_expired","error_description":"...","error_uri":"https://developers.hotmart.com/docs/en/start/http-response-codes/"}
```

Handle 401 by re-running Step 1 and retrying once. Treat `invalid_token` and
`unauthorized` the same way, but do not retry more than once — `unauthorized` is
also returned when the header was never sent.

## Rules

- There are **no OAuth scopes**. A token carries whatever the Hotmart account (and
  the collaborator permissions on it) can see. Do not attempt to request scopes.
- Do not cache a token past `expires_in`.
- If credentials may have leaked, delete and regenerate them in the credentials tool —
  Hotmart supports this explicitly.
- Rate limit is **500 requests per minute** across reads and writes. Watch
  `RateLimit-Remaining` / `X-RateLimit-Remaining-Minute` and back off on `429`
  (`too_many_requests`) using `RateLimit-Reset`.
