# ADR-001: Prevent overlapping bookings with a PostgreSQL exclusion constraint

- **Status:** Accepted
- **Date:** 2026-10-10

## Context

BR-01 requires that confirmed bookings of the same resource never overlap, also when two users submit conflicting requests at the same moment (NFR-02). A naive "check then insert" in the service layer has a race condition: both transactions see a free slot and both insert.

Options considered:

1. **Application check only.** Simple, but broken under concurrency.
2. **Pessimistic lock** (`SELECT ... FOR UPDATE` on the resource row) before checking. Correct, but every booking of a resource serialises on one row lock, and correctness depends on every code path remembering to lock.
3. **`SERIALIZABLE` isolation.** Correct, but requires retry logic for serialization failures and is easy to misconfigure.
4. **Exclusion constraint** on `(resource_id, tstzrange(starts_at, ends_at, '[)'))` for `status = 'CONFIRMED'`. The database guarantees the invariant regardless of the code path.

## Decision

Use option 4:

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;

ALTER TABLE bookings ADD CONSTRAINT ex_bookings_no_overlap
  EXCLUDE USING gist (
    resource_id WITH =,
    tstzrange(starts_at, ends_at, '[)') WITH &&
  ) WHERE (status = 'CONFIRMED');
```

The service still performs a friendly pre-check, which gives a better error message in the common case. The constraint is the guarantee. A `DataIntegrityViolationException` caused by `ex_bookings_no_overlap` (SQLState `23P01`) is translated into HTTP 409 `BOOKING_CONFLICT`.

The half-open range `'[)'` implements "ends at 10:00 does not conflict with starts at 10:00". Cancelled bookings are excluded by the `WHERE` clause, so cancelling frees the slot immediately.

## Consequences

- ✅ The invariant holds for every write path, including future features and manual SQL.
- ✅ The GiST index also speeds up calendar range queries.
- ⚠️ PostgreSQL-specific. The schema cannot run on H2, so integration tests **must** use Testcontainers. This is a deliberate trade-off, and tests run against the same engine as production.
- ⚠️ Requires the `btree_gist` extension. It is available on all mainstream managed PostgreSQL hosts, but must be verified on the chosen free host.
- A dedicated integration test fires concurrent booking requests and asserts that exactly one succeeds.
