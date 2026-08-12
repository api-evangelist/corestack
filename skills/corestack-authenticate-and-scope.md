---
name: corestack-authenticate-and-scope
description: >-
  Establish a CoreStack API session and resolve the master account / tenant context that every other
  CoreStack operation requires. Run this before any other CoreStack skill.
api: CoreStack External API
base_url: https://api.corestack.io
spec: openapi/corestack-external-api-openapi-original.json
operations:
  - authToken
  - RefreshToken
  - UserSessionDetails
  - SwitchMasterAccount
generated: '2026-08-11'
method: generated
source: openapi/corestack-external-api-openapi-original.json + https://docs.corestack.io/docs/corestack-api-modules
---

# Authenticate and scope a CoreStack session

Nothing else in CoreStack works until this is done. Almost every operation in the API takes a
`tenant_id` in its path — 163 of 767 paths do — and many also need an account id. Both come out of
the token call, not out of configuration.

## Before you start

You need an **Access Key** and a **Secret Key**. These are issued per user by a CoreStack Account
Administrator from *Identity and Access Management > Users*, and delivered by email along with the
API URL for your environment. There is no self-service signup and no sandbox — see
`sandbox/` (absent) and `plans/corestack-plans-pricing.yml`.

Confirm which host your keys belong to. `https://api.corestack.io` and
`https://api-discover.corestack.io` both serve the contract, but the API URL you were emailed is
authoritative — regional environments exist (`cloud`, `portal`, `mea`, `in`, `us3`).

## Steps

1. **Mint a token — `authToken`**
   `POST /v1/auth/tokens` with `Content-Type: application/json` and a body of
   `{"access_key": "...", "secret_key": "..."}`.
   Do **not** send auth headers on this call — it is the one operation exempt from them.
   From the response, capture: the access token, the **tenant id**, and the **account id**. You will
   need all three. Calling this endpoint with an empty body returns
   `400 {"message": "Please Provide Username/Password or AccessKey/SecretKey"}`, which is a useful
   liveness check.

2. **Set your headers for everything that follows**
   ```
   Content-Type: application/json
   X-Auth-User:  <your CoreStack username>
   X-Auth-Token: <the token from step 1>
   ```
   `X-Auth-User` is required and is **not** declared in the published specification — the spec only
   declares `X-Auth-Token`. Omitting it produces a 401 that looks like an expired token. This is the
   single most common integration failure on this API.

3. **Confirm the session — `UserSessionDetails`**
   `GET /v1/user_details`. Returns the roles and access the token actually carries. Do this once at
   the start of a run rather than discovering a permission gap thirty calls in: CoreStack authorizes
   by RBAC role, and there is no scope on the token to inspect.

4. **Switch master account if needed — `SwitchMasterAccount`**
   `GET /v1/user/switch_account/{master_account_id}`. This mints a **new** token — replace the one
   in your headers, do not keep using the old one.

5. **Keep the session alive — `RefreshToken`**
   `POST /v1/auth/tokens/refresh` extends validity by another hour and increments `refresh_count`.
   You may refresh **three times**. After the third, the token cannot be extended and you must call
   `authToken` again. Read `expires_at` in the response rather than assuming a duration — the
   provider documents both "one hour" and "a maximum of 6 hours" on different pages.

## Failure handling

- **401 after a quiet period** — the token expired. Refresh if `refresh_count < 3`, otherwise
  re-mint. Access and Secret keys stay valid until an administrator revokes or regenerates them.
- **401 immediately** — you almost certainly omitted `X-Auth-User`.
- **403** — do not retry. The operation is forbidden regardless of state; check the caller's role.
- Every error body is `{"message": "<string>"}`. There is no error code to branch on. See
  `errors/corestack-problem-types.yml`.

## Carry this forward

Hold `tenant_id`, `account_id` and the current token for the rest of the run. A CoreStack account
holds many tenants and a user may have different roles in each, so the tenant you resolve here
determines what every subsequent call can see.
