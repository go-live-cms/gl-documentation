# Sessions (stub)

- List current user's sessions
- Block a session (remote logout)
- Tracks UA/IP, expiry, replaced_by
- Powers refresh rotation + reuse detection

Endpoints (auth required):

- `GET /api/v1/sessions`
- `PUT /api/v1/sessions/block` (by session_id)
