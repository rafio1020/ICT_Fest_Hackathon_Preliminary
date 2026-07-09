# Bug Report — CoWork Booking API

Each bug below is a deviation from the business rules in the README / contest
overview. Fixes are minimal (single-line where possible) and preserve the API
contract exactly (paths, status codes, error codes, JSON field names).

---

## 1. Pagination broken three ways
- **File/line:** `app/routers/bookings.py`, `list_bookings` (~L136–140)
- **Bug:** The query used `order_by(Booking.start_time.desc(), ...)`,
  `.offset(page * limit)`, and a hard-coded `.limit(10)`.
- **Why wrong (rule 11):** Items must be sorted **ascending** by `start_time`,
  page N must return items `[(N−1)·L, N·L)`, and `limit` must be honored. With
  `offset(page*limit)` the first page skipped its own items; the descending order
  and fixed limit further broke ordering and page sizes, causing skipped/repeated
  items.
- **Fix:** `.order_by(Booking.start_time.asc(), Booking.id.asc())`,
  `.offset((page - 1) * limit)`, `.limit(limit)`.

## 2. Booking detail overwrites `start_time` with `created_at`
- **File/line:** `app/routers/bookings.py`, `get_booking` (was L166)
- **Bug:** After serializing, the code did
  `response["start_time"] = iso_utc(booking.created_at)`.
- **Why wrong:** `GET /bookings/{id}` returned the creation timestamp in the
  `start_time` field instead of the real booking start.
- **Fix:** Removed that line; `serialize_booking` already sets the correct value.

## 3. Access-token lifetime was 54000 seconds, not 900
- **File/line:** `app/auth.py`, `create_access_token` (L50)
- **Bug:** `timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES * 60)` with
  `ACCESS_TOKEN_EXPIRE_MINUTES = 15` produced 900 **minutes**.
- **Why wrong (rule 8):** Access tokens must satisfy `exp − iat == 900` seconds.
- **Fix:** `timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)`.

## 4. Logout never invalidated the token
- **File/line:** `app/auth.py`, `get_token_payload` (L97)
- **Bug:** `revoke_access_token` stores the token's `jti`, but the revocation
  check was `if payload.get("sub") in _revoked_tokens` — it compared the user id
  against a set of jtis.
- **Why wrong (rule 8):** Logout must invalidate the presented access token
  (subsequent use → 401). The mismatched key made logout a no-op.
- **Fix:** `if payload.get("jti") in _revoked_tokens`.

## 5. Back-to-back bookings wrongly rejected
- **File/line:** `app/routers/bookings.py`, `_has_conflict` (L50)
- **Bug:** Overlap test used `if b.start_time <= end and start <= b.end_time`.
- **Why wrong (rule 3):** Two bookings overlap iff
  `existing.start < new.end AND new.start < existing.end`; back-to-back bookings
  (one ending exactly when the next starts) are allowed. The `<=` comparisons
  flagged back-to-back bookings as `ROOM_CONFLICT`.
- **Fix:** Strict `<` on both: `if b.start_time < end and start < b.end_time`.

## 6. Cancellation refund tiers wrong
- **File/line:** `app/routers/bookings.py`, `cancel_booking` (L200–205)
- **Bug:** `if notice_hours > 48: 100 / elif notice >= 24h: 50 / else: 50`.
- **Why wrong (rule 6):** Notice ≥ 48h → 100%, [24h, 48h) → 50%, **< 24h → 0%**.
  The `else` branch returned 50% instead of 0%, and `> 48` (on an integer-hour
  value) gave only 50% at exactly 48h instead of 100%.
- **Fix:** `if notice >= timedelta(hours=48): 100 / elif notice >= timedelta(hours=24): 50 / else: 0`
  (also removed the now-unused `notice_hours`).

## 7. UTC-offset datetimes not converted to UTC
- **File/line:** `app/timeutils.py`, `parse_input_datetime` (L13)
- **Bug:** Offset-aware input was handled with `dt.replace(tzinfo=None)`, which
  drops the offset **without converting**.
- **Why wrong (rule 1):** Input carrying a UTC offset must be converted to UTC
  before storage/comparison. `12:00+02:00` was stored as `12:00` instead of
  `10:00`.
- **Fix:** `dt = dt.astimezone(timezone.utc).replace(tzinfo=None)`.

## 8. Five-minute grace window on past bookings
- **File/line:** `app/routers/bookings.py`, `create_booking` (L86)
- **Bug:** `if start <= now - timedelta(seconds=300)` allowed bookings up to 5
  minutes in the past.
- **Why wrong (rule 2):** `start_time` must be strictly in the future — no grace
  window of any size.
- **Fix:** `if start <= now`.

## 9. Duplicate username silently returned instead of `409 USERNAME_TAKEN`
- **File/line:** `app/routers/auth.py`, `register` (was L37–43)
- **Bug:** When a username already existed in the org, the handler returned
  that existing user's `{user_id, org_id, username, role}` with `201`, without
  checking the submitted password at all.
- **Why wrong (rule 15):** A duplicate username within the org must return
  `409 USERNAME_TAKEN`. The old behavior also leaked another user's
  `user_id`/`role` to anyone who "registered" with their username, with no
  password check.
- **Fix:** `raise AppError(409, "USERNAME_TAKEN", "Username already taken in this organization")`.

## 10. No minimum-duration / non-positive-duration validation
- **File/line:** `app/routers/bookings.py`, `create_booking` (was L93–94)
- **Bug:** Only `duration_hours > MAX_DURATION_HOURS` was checked. A
  zero-duration (`start == end`) or negative-duration (`end < start`) booking
  passed the "whole number of hours" check and was never rejected, producing a
  booking with `price_cents <= 0`.
- **Why wrong (rule 2):** `end_time` must be strictly after `start_time`, and
  duration must be a whole number of hours, minimum 1.
- **Fix:** `if duration_hours < MIN_DURATION_HOURS or duration_hours > MAX_DURATION_HOURS: raise AppError(400, "INVALID_BOOKING_WINDOW", ...)`.

## 11. Usage-report cache not invalidated on booking creation
- **File/line:** `app/routers/bookings.py`, `create_booking` (near L120–122)
- **Bug:** Only `cache.invalidate_availability(...)` was called after creating
  a booking; nothing invalidated the org's cached usage-report entries.
- **Why wrong (rule 12):** `GET /admin/usage-report` must reflect the current
  state immediately. A cached report kept showing stale `confirmed_bookings`
  / `revenue_cents` after a new booking was made.
- **Fix:** Added `cache.invalidate_report(user.org_id)` alongside the existing
  availability-cache invalidation.

## 12. `get_booking` missing owner check for members
- **File/line:** `app/routers/bookings.py`, `get_booking` (was L157–164)
- **Bug:** The query only filtered by `Room.org_id`; unlike `cancel_booking`,
  it never checked `booking.user_id == user.id` for non-admin callers.
- **Why wrong (rule 10):** Members may read and cancel only their own
  bookings; another member's booking id must behave as `404
  BOOKING_NOT_FOUND`. A member could read any booking in the org.
- **Fix:** Added `if user.role != "admin" and booking.user_id != user.id: raise AppError(404, "BOOKING_NOT_FOUND", ...)`,
  the same guard already present in `cancel_booking`.

## 13. Availability cache not invalidated on cancellation
- **File/line:** `app/routers/bookings.py`, `cancel_booking` (near L215–217)
- **Bug:** Only `cache.invalidate_report(...)` was called after cancelling a
  booking; nothing invalidated the room/day's cached availability.
- **Why wrong (rule 13):** `GET /rooms/{id}/availability` must reflect the
  current state immediately. A cancelled booking kept showing as a busy
  interval in cached responses.
- **Fix:** Added `cache.invalidate_availability(booking.room_id, booking.start_time.date().isoformat())`
  alongside the existing report-cache invalidation.

## 14. Export cross-org data leak
- **File/line:** `app/services/export.py`, `generate_export` (was L48–50)
- **Bug:** When `include_all=true` **and** `room_id` was supplied, the code
  called `fetch_bookings_raw(db, room_id)`, which loads every booking for that
  room id with **no organization filter at all**.
- **Why wrong (rule 9):** A user (including admins) may only ever act on data
  belonging to their own organization; cross-org resource IDs must behave as
  non-existent. An admin from org A could pass a room id belonging to org B
  and export its bookings.
- **Fix:** Changed that branch to call `_fetch_scoped(db, org_id, None, room_id)`
  instead, which applies the existing `Room.org_id == org_id` filter.

## 15. Refund rounding wrong and inconsistent between two code paths
- **File/line:** `app/services/refunds.py::log_refund` (was L15–17) and
  `app/routers/bookings.py::cancel_booking` (was L209)
- **Bug:** `log_refund` truncated (`int(refund_dollars * 100)`, always rounds
  down); `cancel_booking`'s response used Python's banker's `round()` (rounds
  half-to-even). The two independent calculations could disagree.
- **Why wrong (rule 6):** Refund amount must round to the nearest cent with
  half-cents rounding up, and the response amount must equal the amount
  stored in the `RefundLog`. Example divergence: `price_cents=999`, 50% →
  old response `500`, old ledger `499`.
- **Fix:** Added `calculate_refund_amount_cents(price_cents, percent)` to
  `refunds.py` using exact integer math: `divmod(price_cents * percent, 100)`,
  incrementing the quotient when `remainder * 2 >= 100` (half-up). `log_refund`
  uses it for the stored amount; `cancel_booking` now reads
  `refund_amount_cents` directly from the `RefundLog` entry `log_refund`
  returns, instead of recomputing it — so response and ledger can never
  diverge. Verified against the spec's own example (`50% of 1001 = 501`) and
  a live cancel with a fractional-cent case.

## 16. Refresh tokens were not single-use
- **File/line:** `app/routers/auth.py::refresh` (was L76–88), `app/auth.py`
- **Bug:** Rotation issued a new access+refresh token pair but never revoked
  the presented refresh token, so it could be replayed indefinitely.
- **Why wrong (rule 8):** Refresh tokens are single-use — refreshing must
  invalidate the presented refresh token (reuse → 401).
- **Fix:** Added `_revoked_refresh_tokens` (a jti store in `app/auth.py`,
  mirroring the existing access-token `_revoked_tokens`) with
  `revoke_refresh_token()` / `is_refresh_token_revoked()` helpers. The
  `refresh` endpoint now rejects an already-used refresh token with 401
  *before* processing, and revokes the presented token's jti right before
  returning the new pair. Verified with a chained rotation test: token A → 200
  (new pair B); replaying A → 401; token B → 200 (new pair C); replaying B →
  401; an unrelated access token issued at login remains valid throughout
  (access- and refresh-token revocation are independent).

## 17. Reference-code counter race
- **File/line:** `app/services/reference.py::next_reference_code`
- **Bug:** Read-then-sleep-then-increment on a shared dict with no lock;
  concurrent requests could read the same `current` value and emit duplicate
  `reference_code`s.
- **Why wrong (rule 7):** Every booking's `reference_code` must be unique,
  including under concurrent creation.
- **Fix:** Wrapped the read-increment body in a module-level `threading.Lock`
  (single-process app, so a plain lock suffices). Verified: 6 concurrent
  bookings on distinct slots all succeeded with 6 distinct reference codes.

## 18. Stats service race
- **File/line:** `app/services/stats.py::record_create` / `record_cancel`
- **Bug:** Same read-sleep-write-without-lock pattern; concurrent
  creates/cancels for the same room could lose updates.
- **Why wrong (rule 14):** Room stats must always be consistent with the
  bookings themselves, including after bursts of concurrent activity.
- **Fix:** Wrapped both functions' bodies in a shared module-level
  `threading.Lock`. Verified: 6 concurrent creates for the same room →
  `total_confirmed_bookings == 6` and `total_revenue_cents` exactly
  `6 × price_cents` (no lost updates).

## 19. Rate limiter race
- **File/line:** `app/services/ratelimit.py::record_and_check`
- **Bug:** Bucket trim/append was not locked; concurrent requests from the
  same user could race past each other, under- or over-counting toward the
  20/60s limit.
- **Why wrong (rule 5):** The rate limit must hold under concurrent requests.
- **Fix:** Wrapped the trim-append-check body in a module-level
  `threading.Lock`. Verified: 25 concurrent requests from one user → exactly
  20 succeed (`201`) and exactly 5 get `429 RATE_LIMITED`.

## 20. Room-conflict check and quota check not atomic
- **File/line:** `app/routers/bookings.py::create_booking` (conflict check,
  quota check, insert)
- **Bug:** The conflict check, quota check, and booking insert were three
  separate, unsynchronized steps with an artificial delay between them.
  Concurrent requests could each pass the conflict/quota checks before any of
  them committed, allowing double-booking or quota bypass.
- **Why wrong (rules 3/4):** No double-booking and the booking quota must
  both hold under concurrent requests.
- **Fix:** Added a module-level `_create_lock` and wrapped the
  conflict-check → quota-check → insert → commit section in it. Verified:
  5 concurrent requests for the identical room/slot from different users →
  exactly 1 succeeds, 4 get `409 ROOM_CONFLICT`; 5 concurrent requests from
  one user against a quota of 3 → exactly 3 succeed, 2 get `409
  QUOTA_EXCEEDED`.

## 21. Cancel not atomic → possible duplicate refunds
- **File/line:** `app/routers/bookings.py::cancel_booking`
- **Bug:** The `status == "cancelled"` guard was checked, then (after an
  artificial delay) the refund was logged and the status updated, with no
  locking in between. Two concurrent cancel requests for the same booking
  could both pass the guard and both write a `RefundLog` row.
- **Why wrong (rule 6):** A cancelled booking must have exactly one
  `RefundLog` entry, and this must hold under concurrent cancel requests.
- **Fix:** Added a module-level `_cancel_lock` wrapping the
  guard-through-commit section. Also added `db.refresh(booking)` immediately
  after acquiring the lock — the `booking` object is fetched *before* the
  lock is taken, so without a refresh a thread that queued behind the lock
  would still see the stale pre-cancellation status from its own earlier
  read. Verified: 5 concurrent cancel requests for the same booking →
  exactly 1 succeeds (`200`), 4 get `409 ALREADY_CANCELLED`, and the booking
  ends up with exactly one `RefundLog` entry.

---

## Additional bugs identified (not yet fixed)
This is Hard-tier and left for a later pass:

- **Notifications lock-ordering deadlock** (`services/notifications.py`):
  `notify_created` acquires `_email_lock` then `_audit_lock`;
  `notify_cancelled` acquires `_audit_lock` then `_email_lock` — the opposite
  order. A concurrent create + cancel can deadlock and hang the request
  threads indefinitely (rule 16, "no combination of concurrent valid requests
  may hang the service").
