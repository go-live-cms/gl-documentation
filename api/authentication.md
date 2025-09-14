# API Authentication (PASETO v4 + Session Binding)

## Summary

- **Dual-token model**
  - **Access token** (PASETO v4, short-lived): sent as `Authorization: Bearer <token>`
  - **Refresh token** (PASETO v4, long-lived): stored as **httpOnly** cookie, **rotated** on each refresh
- **Session binding**: refresh token is linked to a DB **session** via **JTI** + hash
- **Reuse detection**: submitting an old/used refresh token blocks all sessions for that user

---

## Endpoints

- `POST /api/v1/auth/login` → access token (Bearer) + refresh cookie
- `POST /api/v1/auth/refresh` → rotates tokens; returns new access token + sets new refresh cookie
- `POST /api/v1/auth/logout` → blocks current session; clears refresh cookie

> Protected endpoints require `Authorization: Bearer <access_token>`.

---

## Tokens & Claims

### Access token (example payload)

```json
{
  "sub": "user:42",
  "user_id": 42,
  "username": "jane",
  "token_type": "access",
  "iss": "go-live",
  "aud": "go-live-client",
  "exp": 1735689600,
  "iat": 1735680600,
  "nbf": 1735680600
}
```

- **Lifetime**: ~15–30 min (configurable)
- **Transport**: `Authorization: Bearer <access_token>`
- **Used by** middleware to authorize requests

### Refresh token (example payload)

```json
{
  "sub": "user:42",
  "user_id": 42,
  "username": "jane",
  "token_type": "refresh",
  "jti": "0e2d1fbf-0d1a-4e07-8a0b-9b7c9f2a7b19",
  "kid": "refresh-key-1",
  "iss": "go-live",
  "aud": "go-live-client",
  "exp": 1738295200,
  "iat": 1735680600
}
```

- **Lifetime**: ~7–30 days (configurable)
- **Storage**: httpOnly cookie `refresh_token`
- **Purpose**: only to obtain new access token (and rotate refresh)

### Cookie Settings (refresh)

- **Name**: `refresh_token`
- **httpOnly**: `true`
- **SameSite**: `Strict`
- **Secure**: `true` (prod); configurable in dev
- **Path**: `/api/v1/auth`
- **Domain**: configurable (`CookieDomain`)

These values reduce XSS/CSRF risk and scope cookies to auth routes.

---

## Flows

### 1) Login (password → tokens)

#### add flow here later!

### 2) Refresh (rotation + reuse detection)

TODO: add flow

### 3) Logout

TODO: add flow

---

## HTTP Status Codes

- **200 OK** — login/refresh/logout success
- **201 Created** — (unused for auth)
- **400 Bad Request** — missing refresh cookie, bad params
- **401 Unauthorized** — invalid credentials, invalid/malformed token, expired session
- **403 Forbidden** — (reserved for RBAC in protected endpoints)
- **404 Not Found** — session not found (rare surfaced case)
- **409 Conflict** — refresh token reuse detected
- **500 Internal Server Error** — token creation/parse or DB failure

---

## Middleware (Bearer auth)

1. Reads `Authorization: Bearer <access_token>`
2. Verifies signature, `token_type == "access"`, and expiry
3. Injects payload (`user_id`, `username`, etc.) into the request context
4. Handlers read it via a known context key (e.g., `authorizationPayloadKey`)

---

## Session Model (DB)

Each refresh token rotation creates a **session** row:

- **id**: uuid
- **user_id**: int
- **username**: text
- **refresh_token_hash**: text (hash of the refresh token)
- **refresh_kid**: text (for key rotation)
- **jti**: uuid
- **user_agent**: text
- **client_ip**: text
- **is_blocked**: bool
- **replaced_by**: uuid? (set when rotated)
- **expires_at**: timestamptz
- **created_at**: timestamptz

**Reuse detection:**

If a refresh is presented that doesn't match the current session (e.g., it was already rotated), we block all sessions for that user and return 409.

---

## Configuration

- **AccessTokenDuration**: e.g., 15m–30m
- **RefreshTokenDuration**: e.g., 7d–30d
- **PasetoRefreshKID**: string (key id used for refresh signing)
- **CookieDomain**: domain for cookies
- **CookieSecure**: bool (dev only)
- **IsDevelopment**: bool (toggles cookie security flexibility)

---

## cURL Examples

### Login

```bash
curl -i -X POST https://api.example.com/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"jane","password":"secret123"}'
# -> HTTP/200
# Set-Cookie: refresh_token=...; HttpOnly; Secure; SameSite=Strict; Path=/api/v1/auth
# { "access_token": "...", "access_token_expires_at": "...", "expires_at": 1700000000, "user": {...} }
```

### Refresh

```bash
curl -i -X POST https://api.example.com/api/v1/auth/refresh \
  --cookie "refresh_token=<value from login/previous refresh>"
# -> HTTP/200 with new access token + rotated refresh cookie
```

### Logout

```bash
curl -i -X POST https://api.example.com/api/v1/auth/logout \
  --cookie "refresh_token=<value>"
# -> HTTP/200 and cookie cleared
```

---

## FAQs

**Why PASETO instead of JWT?**  
Safer defaults, explicit versioning/purpose, no algorithm confusion. We use v4/public with asymmetric signatures.

**Why cookie for refresh?**  
httpOnly + SameSite=Strict reduces XSS/CSRF risks; access token remains in the header for API calls.

**What happens if a refresh token leaks?**  
On reuse detection, the system blocks all sessions for that user and returns 409 Conflict.

**Can I rotate keys?**  
Yes. Refresh tokens carry `kid`; the server can roll keys and validate by `kid`.
