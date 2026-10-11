# ADR-005: React SPA (Vite + JavaScript) built into the Spring Boot jar

- **Status:** Accepted
- **Date:** 2026-10-10
- **Supersedes:** [ADR-004](004-frontend-served-by-spring-boot.md)
- **Change request:** D-5 in [requirements](../requirements.md#8-decision-log)

## Context

The original plan was a vanilla JS frontend (NFR-07 v0.3). Before implementation started, the requirement changed to React, so the project matches the most common frontend stack in job offers for Java full-stack roles. The REST contract ([api.md](../api.md)), domain model and backend ADRs are not affected.

We still want one deployable, one URL and no CORS (the reasons from ADR-004 hold), plus a fast development loop with hot reload.

## Decision

### Stack

| Concern | Choice | Why |
|---|---|---|
| Build tool | **Vite** | Fast dev server with HMR, simple production build |
| Language | **JavaScript** (ES2022, ES modules, JSX) | Developer's preference; no type-check step. Mitigated by ESLint, zod validation at the form boundary and tests |
| Code quality | **ESLint** (`eslint-plugin-react-hooks`) + **Prettier** | Catches common React mistakes; consistent formatting |
| Routing | **React Router** (browser history, clean URLs) | De facto standard; nested layouts for the admin area |
| Server state | **TanStack Query** | Caching, loading/error states, refetch after mutations |
| Forms & validation | **React Hook Form + zod** | Schema mirrors BR-02/03/06/10 on the client; the server stays the source of truth |
| Styling | **CSS Modules** | No extra dependency, scoped class names |
| Calendar | **Own `WeekCalendar` component** | Small scope (week grid + positioned blocks), and a good showcase. No third-party calendar library |
| Tests | **Vitest + React Testing Library + MSW** | Unit/component tests with a mocked API |

### Build and packaging

- Monorepo: `backend/` (Spring Boot, Maven) and `frontend/` (Vite).
- `frontend-maven-plugin` installs Node, runs `npm ci` and `npm run build` during `mvn package`, and the Vite output (`frontend/dist`) is copied to `classpath:/static`. A single `./mvnw package` produces a complete jar, both locally and in CI.
- The Dockerfile builds the jar with Maven, so Docker Compose and deployment stay a single container.

### Serving and routing

- `/api/v1/**` → REST API, `/swagger-ui.html` → API docs, `/assets/**` → hashed static files (long cache).
- Any other `GET` without a file extension (e.g. `/resources/3`, `/admin/bookings`) is forwarded to `index.html`, so deep links and page refresh work with browser history routing. Implemented as a small controller or `WebMvcConfigurer`, covered by an integration test.
- Security: static files and the SPA fallback are public; `/api/v1/**` requires a JWT ([ADR-002](002-stateless-jwt-auth.md)).

### Development loop

`npm run dev` (Vite on :5173) proxies `/api` to Spring Boot on :8080 via `server.proxy` in `vite.config.js`. Same-origin from the browser's point of view, so there is no CORS in dev either.

## Consequences

- ✅ Single deployable, single URL, no CORS: all ADR-004 benefits kept.
- ✅ Hot reload during development; minified, hashed assets in production.
- ⚠️ No static types, so an API contract change is not caught at build time. Mitigated by keeping all API calls in one `api/` module, MSW handlers mirroring the examples in `api.md`, and component tests. TypeScript can be adopted later file by file, since Vite supports mixing `.jsx` and `.tsx`.
- ✅ React escapes rendered values by default, which reduces the XSS risk noted in ADR-002.
- ⚠️ The backend build now needs Node (downloaded by the Maven plugin). The first build is slower, so CI caches `~/.m2` and `frontend/node_modules`.
- ⚠️ SPA fallback must not swallow unknown `/api` paths: they still return 404 `ProblemDetail`, not `index.html`. Covered by a test.
- CI gets a separate frontend job (`npm run lint`, `npm test`) so frontend failures are reported clearly.
