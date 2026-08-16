---
name: Authenticate against the Galatea Bio Octopod Ancestry API
description: Obtain, confirm, refresh and revoke a bearer token for the Octopod Ancestry API, using either an organization API key or an email and password login with MFA.
api: openapi/galatea-bio-octopod-openapi.yml
generated: '2026-08-16'
method: generated
source: https://docs.galatea.bio/#api-reference-auth-contents
operations:
  - users_auth_create
  - users_confirm_create
  - users_request-new-code_create
  - users_refresh_create
  - users_logout_create
  - users_me_list
  - organizations_generate-api-key_create
---

# Authenticate against the Octopod Ancestry API

Every operation on this API requires `Authorization: Bearer <token>`. There is no OAuth, no scopes,
and no anonymous surface — an unauthenticated call returns `401` with
`{"detail": "Authentication credentials were not provided."}`.

## Base URLs

| Environment | Base URL |
|---|---|
| Production | `https://api.galatea.bio/api/v1` |
| Sandbox | `https://api.sandbox.galatea.bio/api/v1` |

Pick the environment by base URL only. There is no key prefix to tell production and sandbox tokens
apart, so a misconfigured base URL is the main way to cross environments by accident.

## Choose a credential mode

**Mode 1 — organization API key (preferred for unattended work).** An organization admin mints a
long-lived key with `organizations_generate-api-key_create`
(`POST /organizations/{organization_id}/generate-api-key`). Use that value directly as the bearer
token. Nothing else is needed.

**Mode 2 — email and password.** Use this only when no API key is available. The resulting access
token is explicitly short-lived.

## Steps for mode 2

1. **Log in.** Call `users_auth_create` (`POST /users/auth`) with `{"email": ..., "password": ...}`.
   On success the response carries `access`, `refresh` and `websocket_access`.

2. **Complete MFA if challenged.** When the account has multi-factor enabled, confirm with
   `users_confirm_create` (`POST /users/confirm`), sending the `mfa_session_id` from the login step
   plus the numeric `code`. If the code expired, request another with
   `users_request-new-code_create` (`POST /users/request-new-code`).

3. **Use the token.** Send `Authorization: Bearer <access>` on every subsequent call.

4. **Verify identity and entitlements.** Call `users_me_list` (`GET /users/me`). The response
   contains an `org` object; read `available_models` from it to learn which analysis models this
   organization may submit orders against. Do this before submitting any order — a model the
   organization is not entitled to fails at submission.

5. **Refresh before expiry.** Call `users_refresh_create` (`POST /users/refresh`) with both the
   `refresh` token and the expired `access` token. Do not treat a `401` mid-workflow as a fatal
   error; refresh once and retry the call.

6. **Revoke when finished.** Call `users_logout_create` (`POST /users/logout`) with the `refresh`
   token. A successful logout returns HTTP `200`.

## Error handling

Errors are a flat `{"detail": "..."}` object — not RFC 9457 problem+json. Read the `detail` string.

- `401` — token missing, wrong, or expired. Refresh once, then re-authenticate.
- `403` — token is valid but the user's role does not permit the call. Organization, credit,
  SFTP-user and model-management endpoints are admin-scoped. Do not retry; escalate to an admin.
- `406` — send `Accept: application/json`. This status is also returned for malformed identifiers
  (`{"detail": "Incorrect ID"}`), which is unusual — do not assume `406` always means content
  negotiation.

## Cautions

- The API publishes no rate limits and returns no `RateLimit-*` or `Retry-After` headers. Apply your
  own conservative backoff rather than probing for a ceiling.
- There is no idempotency key on this API. Never blind-retry a `POST` you did not receive a response
  for — check state first.

See `authentication/galatea-bio-authentication.yml` and `conventions/galatea-bio-conventions.yml`.
