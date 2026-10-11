# ADR-002: Stateless JWT authentication

- **Status:** Accepted
- **Date:** 2026-10-10

## Context

The frontend is a vanilla JS SPA calling a REST API (NFR-01). We need authentication that is simple to run on a free host with a single instance, works with Swagger UI, and is a recognisable industry pattern for a portfolio project.

Options considered:

1. **Server-side session** (`JSESSIONID` cookie, Spring Session). Simple, built-in logout. Requires CSRF protection and is less typical for a REST API showcase.
2. **Stateless JWT** sent as `Authorization: Bearer`. No server state, easy to test with Swagger, common in job requirements.
3. **External identity provider** (Keycloak, Auth0). Realistic for production, but too heavy for an MVP and the free hosting budget.

## Decision

Use option 2, with Spring Security's OAuth2 Resource Server support (`spring-boot-starter-oauth2-resource-server`) and a symmetric HS256 key from configuration (`JWT_SECRET` environment variable). This avoids hand-written JWT filters.

- `POST /api/v1/auth/login` validates credentials (BCrypt) and issues a token with claims `sub` (user id), `email`, `role`, `exp`.
- Token lifetime: **1 hour**. No refresh token in MVP. The user logs in again.
- The frontend keeps the token in `sessionStorage`, so closing the tab logs the user out.
- Logout (US-02) is client-side: the token is discarded.
- On every request the `active` flag is **not** re-checked against the DB in MVP. A deactivated user keeps access until the token expires (max 1 h). This is an accepted risk.

## Consequences

- ✅ No server-side session state. Simple horizontal scaling, if ever needed.
- ✅ Swagger UI works with the "Authorize" button.
- ⚠️ Tokens cannot be revoked before expiry. Mitigated by the short lifetime. A deny-list or refresh tokens can be added later.
- ⚠️ A token in `sessionStorage` is readable by JavaScript, so XSS would expose it. Mitigations: no `innerHTML` with user data in the frontend (use `textContent`) and a strict Content-Security-Policy header.
