# ADR-003: Time handling (UTC storage, Europe/Warsaw business rules)

- **Status:** Accepted
- **Date:** 2026-10-10

## Context

Bookings are time ranges. BR-08 says time is stored in UTC and presented in `Europe/Warsaw`. Office hours (BR-10) are defined in local time ("7:00–20:00"), and Poland switches between CET and CEST, so a fixed UTC offset would be wrong for half of the year.

## Decision

| Layer | Type / format |
|---|---|
| Database | `timestamptz` (stored as UTC) |
| Java domain & persistence | `java.time.Instant` |
| Business rule checks (office hours, 15-min grid, max duration, horizon) | Convert to `ZonedDateTime` in the configured zone (`booking.zone`) |
| API request | ISO-8601 **with offset** (`2026-10-14T10:00:00+02:00`). Values without an offset are rejected (400) |
| API response | ISO-8601 UTC (`2026-10-14T08:00:00Z`) |
| Frontend | Renders in `Europe/Warsaw` using `Intl.DateTimeFormat` with `timeZone: 'Europe/Warsaw'`, independent of the browser's zone |

The current time is obtained from an injected `java.time.Clock` bean, never from `Instant.now()` directly. Tests use `Clock.fixed(...)`, so rules such as BR-03 (not in the past) and BR-05 (horizon) are deterministic.

The JVM and the DB session run with UTC (`-Duser.timezone=UTC`, `hibernate.jdbc.time_zone=UTC`), so nothing depends on the server's locale.

## Consequences

- ✅ DST transitions are handled correctly by `java.time`.
- ✅ Deterministic tests for time-based rules.
- ⚠️ The frontend must always send an offset. A small helper builds it from the selected local date and time.
- Multi-office support (different zones) would require moving the zone from configuration to the resource or location. This is out of scope.
