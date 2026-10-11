# API Contract (v1)

> Phase 1 · Human-readable contract. The authoritative, generated spec will be available at `/swagger-ui.html` (springdoc-openapi) once implemented.
> Related: [domain model](domain-model.md) · [UI screens](ui-screens.md)

## 1. Conventions

| Topic | Convention |
|---|---|
| Base path | `/api/v1` |
| Format | JSON, `camelCase` fields |
| Auth | `Authorization: Bearer <JWT>` on every endpoint except `POST /auth/login` |
| Admin endpoints | Under `/api/v1/admin/**`, require role `ADMIN` |
| Time | Requests accept ISO-8601 with offset (`2026-10-14T10:00:00+02:00`). Responses always return UTC instants (`2026-10-14T08:00:00Z`). See [ADR-003](adr/003-time-handling.md). |
| State changes | Domain actions are modelled as sub-resources (`POST /bookings/{id}/cancel`), not `DELETE`, because nothing is physically deleted (BR-09) |
| Pagination | `?page=0&size=20&sort=startsAt,asc`. Response wrapper: `{ content, page, size, totalElements, totalPages }` |
| Errors | RFC 7807 `application/problem+json` (see §6) |

## 2. Auth

| Method | Path | Description | Story |
|---|---|---|---|
| `POST` | `/auth/login` | Exchange credentials for a JWT | US-01 |
| `GET` | `/auth/me` | Current user profile | US-01 |

Logout (US-02) is client-side: the frontend discards the token (stateless JWT, see [ADR-002](adr/002-stateless-jwt-auth.md)).

```http
POST /api/v1/auth/login
{ "email": "user@demo.com", "password": "demo1234" }

200 OK
{
  "accessToken": "eyJhbGciOi...",
  "tokenType": "Bearer",
  "expiresIn": 3600,
  "user": { "id": 1, "email": "user@demo.com", "firstName": "Anna", "lastName": "Nowak", "role": "USER" }
}
```

## 3. Resources (catalog)

| Method | Path | Description | Story |
|---|---|---|---|
| `GET` | `/resources?type=ROOM&minCapacity=6&amenityIds=1,3&page=0` | List **active** resources with filters | US-05 |
| `GET` | `/resources/{id}` | Resource details | US-06 |
| `GET` | `/resources/{id}/bookings?from=...&to=...` | Confirmed bookings in a range, for the calendar (max range 31 days) | US-07 |
| `GET` | `/amenities` | Amenity dictionary (for filters) | US-05 |

Calendar response (BR-12: booker contact details visible to logged-in users):

```json
[
  {
    "id": 42,
    "title": "Sprint planning",
    "startsAt": "2026-10-14T08:00:00Z",
    "endsAt": "2026-10-14T09:30:00Z",
    "bookedBy": { "id": 7, "fullName": "Jan Kowalski", "email": "jan.kowalski@demo.com" },
    "mine": false
  }
]
```

## 4. Bookings

| Method | Path | Description | Story |
|---|---|---|---|
| `POST` | `/bookings` | Create a booking | US-08 |
| `GET` | `/bookings/me?scope=upcoming\|past&page=0` | Current user's bookings | US-09 |
| `GET` | `/bookings/{id}` | Booking details (owner or admin) | US-09 |
| `POST` | `/bookings/{id}/cancel` | Cancel own booking before it starts | US-10 |
| `PATCH` | `/bookings/{id}` | Reschedule (`startsAt`, `endsAt`, `title`, `attendees`) with `version` for optimistic locking | US-11 · v1.1 |
| `GET` | `/bookings/availability?from=...&to=...&type=ROOM&minCapacity=6` | Resources free in the given range | US-12 · v1.1 |

```http
POST /api/v1/bookings
{
  "resourceId": 3,
  "title": "Client demo",
  "attendees": 6,
  "startsAt": "2026-10-14T10:00:00+02:00",
  "endsAt":   "2026-10-14T11:00:00+02:00"
}

201 Created
Location: /api/v1/bookings/57
{
  "id": 57,
  "resource": { "id": 3, "name": "Room Everest", "type": "ROOM" },
  "title": "Client demo",
  "attendees": 6,
  "startsAt": "2026-10-14T08:00:00Z",
  "endsAt": "2026-10-14T09:00:00Z",
  "status": "CONFIRMED",
  "version": 0,
  "createdAt": "2026-10-10T10:41:12Z"
}
```

## 5. Admin

| Method | Path | Description | Story |
|---|---|---|---|
| `GET` | `/admin/resources?active=true\|false&page=0` | All resources, including inactive | US-13 |
| `POST` | `/admin/resources` | Create resource | US-13 |
| `PUT` | `/admin/resources/{id}` | Update resource (with `version`) | US-13 |
| `GET` | `/admin/resources/{id}/deactivation-impact` | `{ "upcomingBookings": 3 }`, for the confirmation dialog | US-13 |
| `POST` | `/admin/resources/{id}/deactivate` | Deactivate and auto-cancel future bookings (BR-11). Returns `{ "cancelledBookings": 3 }` | US-13 |
| `POST` | `/admin/resources/{id}/activate` | Re-activate | US-13 |
| `GET` | `/admin/bookings?resourceId=&userId=&from=&to=&status=&page=0` | All bookings with filters | US-14 |
| `POST` | `/admin/bookings/{id}/cancel` | Cancel any booking, body `{ "note": "..." }` | US-15 · v1.1 |
| `GET/POST/PATCH` | `/admin/users[/{id}]` | Manage user accounts | US-03 · v1.1 |

## 6. Errors

Every error uses `ProblemDetail` with a stable machine-readable `code`, so the frontend can map it to a message without parsing text.

```json
{
  "type": "https://bookspace.example/problems/booking-conflict",
  "title": "Booking conflict",
  "status": 409,
  "detail": "Room Everest is already booked between 10:00 and 11:00.",
  "instance": "/api/v1/bookings",
  "code": "BOOKING_CONFLICT"
}
```

| HTTP | `code` | When |
|---|---|---|
| 400 | `VALIDATION_FAILED` | Bean Validation errors. Includes `errors: [{ field, message }]` |
| 401 | `UNAUTHORIZED` | Missing, invalid or expired token |
| 401 | `INVALID_CREDENTIALS` | Wrong e-mail/password or inactive account |
| 403 | `FORBIDDEN` | Not the owner / not an admin |
| 404 | `NOT_FOUND` | Unknown id |
| 409 | `BOOKING_CONFLICT` | Overlap (BR-01), including the concurrent-request case |
| 409 | `INVALID_STATE` | Cancel an already cancelled, ongoing or past booking |
| 409 | `STALE_VERSION` | Optimistic locking failure |
| 422 | `BOOKING_IN_PAST` | BR-03 |
| 422 | `DURATION_INVALID` | BR-02 / BR-04 (too short, too long, not 15-min aligned) |
| 422 | `BEYOND_HORIZON` | BR-05 |
| 422 | `CAPACITY_EXCEEDED` | BR-06 |
| 422 | `RESOURCE_INACTIVE` | BR-07 |
| 422 | `OUTSIDE_OFFICE_HOURS` | BR-10 |

Rule of thumb: **400** for a malformed request, **422** for a well-formed request that breaks a business rule, **409** for a conflict with current state.
