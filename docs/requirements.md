# BookSpace — Requirements Specification

> Phase 0 · Version 0.4 · Status: accepted (change request D-5 applied)
> Related: [domain model](domain-model.md) · [API contract](api.md) · [UI screens](ui-screens.md) · [ADRs](adr/)

## 1. Product vision

BookSpace is a web application for booking company rooms and resources: meeting rooms, equipment (projectors, laptops) and company vehicles. Employees check availability and book a resource in one place, without e-mails or spreadsheets. An administrator manages the resource catalog and sees all bookings.

**Problem:** double bookings, no visibility of who has a room or a car, informal "verbal" reservations.
**Goal:** a single source of truth for resource availability, with a guarantee that bookings never collide.

## 2. Glossary

| Term | Meaning |
|---|---|
| **Resource** | Anything that can be booked. Type: `ROOM`, `EQUIPMENT`, `VEHICLE`. |
| **Room** | A resource of type `ROOM`. Has a capacity, a location (building/floor) and amenities. |
| **Amenity** | A feature of a room, e.g. projector, whiteboard, video conferencing. |
| **Booking** | Assignment of a resource to a user for a time range `[start, end)`. |
| **Half-open range** | `[start, end)`: a booking 9:00–10:00 does not collide with 10:00–11:00. |
| **Booking status** | `CONFIRMED` or `CANCELLED` (no approval workflow in MVP). |
| **Inactive resource** | Withdrawn from use. Visible in history, but accepts no new bookings. |

## 3. Actors and roles

| Role | Description | Permissions |
|---|---|---|
| **Guest** | Not logged in | Login screen only |
| **Employee** (`USER`) | Regular user | Browses resources, books, manages **own** bookings |
| **Administrator** (`ADMIN`) | Office manager / IT | Everything `USER` can do, plus resource CRUD, view and cancel **all** bookings, user management |

## 4. Functional requirements

MoSCoW priorities: **M** = Must (MVP), **S** = Should, **C** = Could, **W** = Won't (for now).

### E1. Authentication and accounts

| ID | User story | Prio |
|---|---|---|
| US-01 | As an employee, I want to log in with e-mail and password, so that I can use the application. | M |
| US-02 | As a logged-in user, I want to log out. | M |
| US-03 | As an administrator, I want to create and deactivate employee accounts. | S |
| US-04 | As an employee, I want to change my password. | C |

**Acceptance criteria US-01:**
- Valid credentials: the user receives a JWT and lands on the resources view.
- Invalid credentials: message "Invalid e-mail or password" (without saying which field is wrong).
- Inactive account: login rejected.
- Expired token: redirect to login.

> Demo accounts (`user@demo.com`, `admin@demo.com`) are created by a seed migration, needed for the public demo.

### E2. Resource catalog

| ID | User story | Prio |
|---|---|---|
| US-05 | As an employee, I want to browse active resources with filters (type, minimum capacity, amenities), so that I can find a suitable one. | M |
| US-06 | As an employee, I want to see resource details (description, capacity, location, amenities). | M |

### E3. Bookings

| ID | User story | Prio |
|---|---|---|
| US-07 | As an employee, I want to see a resource's occupancy in a day/week view, so that I can pick a free slot. | M |
| US-08 | As an employee, I want to book a resource for a chosen time range. | M |
| US-09 | As an employee, I want to see my upcoming and past bookings. | M |
| US-10 | As an employee, I want to cancel my booking, so that the resource becomes free. | M |
| US-11 | As an employee, I want to reschedule my booking. | S |
| US-12 | As an employee, I want to search for resources free in a given range ("find a free room tomorrow 10–11 for 6 people"). | S |

**Acceptance criteria US-07:**
- The calendar shows office hours only (BR-10). Time outside them is greyed out and not clickable.
- A busy slot shows the booking title and the booker's full name and e-mail (BR-12), with a `mailto:` link so they can be contacted.
- The user's own bookings are visually highlighted.

**Acceptance criteria US-08 (key story):**
- **Given** an active resource free in the range, **when** the employee submits a valid booking, **then** a `CONFIRMED` booking is created (HTTP 201) and shown in the calendar.
- **Given** the range overlaps an existing `CONFIRMED` booking, **then** HTTP 409 with a conflict message.
- **Given** two concurrent requests for the same range, **then** exactly one succeeds and the other gets 409.
- A booking ending exactly when another starts is **not** a conflict.
- Breaking any of BR-02…BR-07 or BR-10: HTTP 400/422 with an error description.
- A booking has a title (required, max 100 characters) and an optional number of attendees.

**Acceptance criteria US-10:**
- The owner can cancel a booking that has not started yet. Status changes to `CANCELLED` (the record is not deleted).
- A `USER` trying to cancel someone else's booking: HTTP 403.
- Cancelling an ongoing or past booking: HTTP 409.

### E4. Administration

| ID | User story | Prio |
|---|---|---|
| US-13 | As an administrator, I want to create, edit and deactivate resources. | M |
| US-14 | As an administrator, I want to see all bookings with filters (resource, user, date range). | M |
| US-15 | As an administrator, I want to cancel any booking with a reason. | S |
| US-16 | As an administrator, I want to manage the amenity dictionary. | C |

**Acceptance criteria US-13 (deactivation):**
- An inactive resource disappears from the catalog for `USER` and accepts no new bookings.
- Before deactivating a resource with future bookings, the admin sees a confirmation with their number, e.g. "3 upcoming bookings will be cancelled".
- After confirmation, all future `CONFIRMED` bookings change to `CANCELLED` with reason `RESOURCE_DEACTIVATED` (BR-11). Deactivation and cancellations run in one transaction.
- Bookings ongoing at the moment of deactivation are not cancelled.

### E5. Notifications

| ID | User story | Prio |
|---|---|---|
| US-17 | As an employee, I want an e-mail when my booking is confirmed or cancelled. | C |

> Locally: Mailpit in Docker Compose.

### Out of scope for v1 (Won't)
Recurring bookings, approval workflow (e.g. for vehicles), iCal export and Google/Outlook integration, SSO login, mobile app, multi-tenancy.

## 5. Business rules

| ID | Rule |
|---|---|
| BR-01 | `CONFIRMED` bookings of the same resource **must not overlap** (ranges `[start, end)`). Guaranteed by the database, not only the application ([ADR-001](adr/001-overlap-prevention-exclusion-constraint.md)). |
| BR-02 | `start < end`, minimum booking length 15 min, 15-min granularity. |
| BR-03 | Bookings cannot be created in or moved to the past. |
| BR-04 | Maximum length: room 8 h, equipment and vehicle 5 days (configurable per type). |
| BR-05 | Bookings can be made at most 90 days ahead. |
| BR-06 | Number of attendees (if given) ≤ room capacity. |
| BR-07 | An inactive resource accepts no new bookings. |
| BR-08 | Time is stored in UTC (`timestamptz`) and presented in `Europe/Warsaw` ([ADR-003](adr/003-time-handling.md)). |
| BR-09 | Bookings are never physically deleted; only their status changes (history and audit). |
| BR-10 | **Office hours:** Mon–Fri 7:00–20:00 (configurable). A room booking must fit entirely within the office hours of one working day. Equipment and vehicles may span several days, but pick-up (`start`) and return (`end`) must fall within office hours. |
| BR-11 | **Deactivating a resource** automatically cancels its future bookings (reason `RESOURCE_DEACTIVATED`). Ongoing bookings stay unchanged. |
| BR-12 | **Booker visibility:** every logged-in user sees the booker's full name and e-mail on a busy slot. Not available to guests. |

## 6. Non-functional requirements

| ID | Area | Requirement |
|---|---|---|
| NFR-01 | Security | Passwords hashed with BCrypt. JWT authentication ([ADR-002](adr/002-stateless-jwt-auth.md)). Authorization per endpoint and per resource owner. HTTPS in production. |
| NFR-02 | Concurrency | Concurrent bookings cannot break BR-01. Concurrent edits of the same entity are detected (optimistic locking). |
| NFR-03 | API | REST, OpenAPI/Swagger UI docs. Consistent error format (RFC 7807 `ProblemDetail`). Versioning `/api/v1`. |
| NFR-04 | Quality | Backend: unit tests for domain logic and integration tests with Testcontainers (PostgreSQL). Frontend: component tests (Vitest + React Testing Library). CI runs both on every PR. |
| NFR-05 | Database | PostgreSQL, schema managed exclusively by Flyway migrations. |
| NFR-06 | Local run | `docker compose up` starts the whole environment with one command. |
| NFR-07 | Frontend | React SPA (Vite, JavaScript), packaged into the Spring Boot jar ([ADR-005](adr/005-react-spa-built-into-spring-boot-jar.md)). Responsive (desktop and mobile). UI language: English. |
| NFR-08 | Performance | API responses < 300 ms under typical demo load. Paginated lists. |
| NFR-09 | Observability | Spring Boot Actuator (`/actuator/health`), structured logs. |
| NFR-10 | Documentation | README, OpenAPI descriptions, API error messages and commits in English. |

## 7. MVP scope (v1.0)

US-01, US-02, US-05, US-06, US-07, US-08, US-09, US-10, US-13, US-14, all BRs, NFR-01…NFR-10, and deployment with demo accounts.

**Next versions:** v1.1: US-11, US-12, US-15, US-03 · v1.2: US-17, US-04, US-16.

## 8. Decision log

| # | Question | Decision | Where |
|---|---|---|---|
| D-1 | UI and README language | English | NFR-07, NFR-10 |
| D-2 | Future bookings after resource deactivation | Cancelled automatically | BR-11, US-13 |
| D-3 | Office hours | Apply: Mon–Fri 7:00–20:00, configurable | BR-10, US-07 |
| D-4 | Booker visibility | Full name and e-mail visible, so people can contact each other | BR-12, US-07 |
| D-5 | **Change request (2026-10-10):** frontend technology | React + Vite (JavaScript) instead of vanilla JS, still served from the Spring Boot jar. Raised before implementation; API contract and domain model unaffected | NFR-04, NFR-07, ADR-005 (supersedes ADR-004) |

**Assumptions:** office hours are shared by all resources; public holidays are not handled in MVP (possible holiday list in v1.2).
