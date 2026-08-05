---
name: Authenticate against the OPAQUE Platform REST API
description: Turn an OPAQUE API key into a working session token and confirm the caller's identity.
api: openapi/opaque-platform-api-openapi.yml
operations: [refresh_user_tokens, get_user, get_api_key, get_user_keys]
generated: '2026-08-04'
method: generated
source: https://docs.opaque.co/en/latest/public_guide/developers/rest_api/authentication/
---

# Authenticate against the OPAQUE Platform REST API

Do this first. Every other OPAQUE skill assumes a valid session token.

## Before you start

- **Base URL.** OPAQUE runs the API inside the customer's own environment. The base URL is
  `https://api.<your-subdomain>/<version>` — the same subdomain as the OPAQUE web app with `app`
  replaced by `api`. Ask the workspace admin; do not guess it.
- **API key.** Sign in to the OPAQUE web application (SSO is required — Entra ID or Okta) and copy
  the API key from **API Keys** in the left nav. It is valid for six months.
- The API key is base64. Decode it and read two fields out of the JSON: `refresh_token` and
  `user_identity_secret`.

## Steps

1. **Exchange the refresh token for a session token.** Call `refresh_user_tokens`
   (`POST /{version}/auth/refresh-token`) with the refresh token set as the HttpOnly cookie
   `refreshTokenCookie`. Read `accessToken` from the response body.
   - `400` means the `refreshTokenCookie` was not set on the request.
   - `401` means the refresh token has expired — the user must sign in to the web app again and
     collect a new API key. There is no programmatic recovery.
2. **Send the session token.** Put it in `Authorization: Bearer <session_token>` on every request.
   It is a JWT and it lives for **10 minutes** — refresh on a timer, not on failure.
   For browser-originated calls (result and log downloads) send it as the `sessionTokenCookie`
   cookie instead.
3. **Carry the user identity secret where it is required.** Send `userIdentitySecret` as a cookie
   on operations that touch cryptographic material or sensitive data — data upload and job-run
   results. The specification does not declare this per operation, so treat the docs as the
   authority.
4. **Verify.** Call `get_user` (`GET /{version}/user`). A `200` with the caller's email confirms
   the token. `get_user_keys` (`GET /{version}/user/keys`) returns the user's keys;
   `get_api_key` (`GET /{version}/auth/api-key`) returns the API key for the session.
5. **Finish cleanly.** `user_logout` (`POST /{version}/auth/logout`) invalidates the session token.

## Rules

- HTTPS only. Plain HTTP is rejected.
- There is no OAuth flow and there are no scopes. Authorization is by organization and workspace
  role, not by token scope.
- Never write an API key, refresh token, session token or user identity secret into logs, traces
  or source. Test mode traces can expose payload data — treat them as sensitive.
- Errors come back as `{type, title, status, detail}` over `application/json`. See
  `errors/opaque-problem-types.yml`.
