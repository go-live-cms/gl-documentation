# PASETO v4 (Public) — Concepts

We use **PASETO v4 public tokens** (not JWT):

- Safer defaults (no "alg" foot-guns), versioned, explicit purpose.
- Signed with an asymmetric keypair (public verification).
- Access token and refresh token are distinct; refresh includes a JTI used to bind server-side sessions.

Key ideas:

- **Access token**: short-lived, carried in `Authorization: Bearer`.
- **Refresh token**: long-lived, httpOnly cookie, rotated on every refresh, bound to a DB session row via **JTI** + refresh hash.
- **KID**: refresh tokens include a key identifier to enable rotation.

See implementation details in [API Authentication](../api/authentication.md).
