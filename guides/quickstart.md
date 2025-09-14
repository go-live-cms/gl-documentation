# Quickstart

1. Run the API.
2. Create an admin user.
3. Authenticate:
   - `POST /api/v1/auth/login` → get **access token** (Bearer) + **refresh cookie**.
4. Call protected endpoint with:
   - `Authorization: Bearer <access_token>`
5. Refresh:
   - `POST /api/v1/auth/refresh` (cookie-based) → new access + refresh (rotation).
