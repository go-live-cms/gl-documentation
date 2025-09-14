# Security Overview (stub)

- httpOnly + SameSite=Strict cookies for refresh.
- HTTPS required in production; `Secure` flag set for cookies.
- Token rotation with reuse detection → blocks sessions on suspected theft.
- User agent/IP change logs for anomaly detection.

Details: [Authentication](../api/authentication.md) and [Sessions](../api/sessions.md).
