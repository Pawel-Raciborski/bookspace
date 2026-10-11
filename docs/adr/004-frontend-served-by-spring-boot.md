# ADR-004: Frontend served by Spring Boot (single deployable)

- **Status:** Superseded by [ADR-005](005-react-spa-built-into-spring-boot-jar.md)
- **Date:** 2026-10-10

> The single-deployable idea is kept in ADR-005. What changed: the frontend is now a React + Vite app with a build step, and routing is no longer hash-based.

## Context

The frontend is plain HTML, CSS and ES modules with no build step (NFR-07). It can be hosted separately (static host, CDN) or served by the backend. The project must run with one `docker compose up` (NFR-06) and be deployed on a free host.

## Decision

The frontend lives in `frontend/` at the repository root and is copied into the Spring Boot jar as static resources (`classpath:/static`) during the build. The application serves:

- `/` → `index.html` (SPA with hash routing: `#/resources`, `#/my-bookings`, …)
- `/api/v1/**` → REST API
- `/swagger-ui.html` → API docs

Hash-based routing means the server never needs SPA fallback rules.

During development, the frontend is edited directly and the browser reloads. Spring Boot DevTools, or a small static server that proxies `/api` to the backend, avoids rebuilding the jar on every change.

## Consequences

- ✅ One container, one URL, no CORS configuration.
- ✅ Simplest possible deployment on a free host.
- ⚠️ The frontend and backend are deployed together. Acceptable for a monolith of this size.
- ⚠️ No bundler, so no minification. Acceptable. Native ES modules are fast enough for this app.
- If the frontend is ever split out, only CORS and the API base URL need to change.
