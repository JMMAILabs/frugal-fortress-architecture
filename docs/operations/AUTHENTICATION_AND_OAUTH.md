# Authentication & Identity Management

The Frugal Fortress utilizes a stateless, JWT-based authentication architecture, delegating identity verification to Google OAuth2 (OpenID Connect).

## 1. The OAuth2 Flow
We implement a secure, backend-orchestrated OAuth flow to prevent exposing client secrets to the browser.

1. **Initiation:** The frontend redirects the user to `GET /api/v1/auth/login/google?module=<module_name>`.
2. **Redirection:** The backend generates a PKCE `code_verifier`, mints a secure `state` JWT, and redirects the user to Google's consent screen.
3. **Callback:** Google redirects back to the backend (`/api/v1/auth/callback/google`). The backend exchanges the code for an access token, fetches the user profile, and upserts the user into the database.
4. **Frontend Handoff:** The backend redirects the user back to the frontend SPA (defined by `AUTH_FRONTEND_SUCCESS_URL`).

## 2. The URL Fragment Security Pattern
When redirecting back to the frontend, the backend passes the JWT via the URL fragment (hash), **not** as a query parameter:
`http://localhost:3001/auth/callback/google#access_token=eyJhbG...`

**Why? (Security Posture):**
URL fragments (`#`) are never sent to the server by the browser. This ensures that sensitive JWTs are never accidentally logged in frontend access logs, proxy logs, or analytics tools. The frontend Next.js application must parse `location.hash` on the client side to extract and store the token.

## 3. Module-Scoped Sessions
To enforce strict lateral security, JWTs are scoped to specific product modules.
*   A token issued for `module=pdf_anki` **cannot** be used to access endpoints under `/api/v1/receipts`.
*   The `require_session_payload_for_module` dependency enforces this at the FastAPI route level, returning HTTP 403 Forbidden if a token crosses module boundaries.

## 4. Google Cloud Console Configuration
To configure the OAuth consent screen in GCP:
1. Navigate to **APIs & Services → OAuth consent screen**.
2. Ensure the following scopes are requested:
   * `.../auth/userinfo.email`
   * `.../auth/userinfo.profile`
3. Add your backend callback URL to the **Authorized redirect URIs**:
   `https://<your-backend-domain>/api/v1/auth/callback/google`