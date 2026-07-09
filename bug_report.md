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

---

## 9. Duplicate username silently logged the caller in instead of 409
- **File/line:** `app/routers/auth.py`, `register` (~L32–43)
- **Bug:** When a username already existed in the target org, the handler
  returned the *existing* user's info (200-style success) instead of rejecting
  the request.
- **Why wrong (rule 15):** A duplicate username within an org must return
  `409 USERNAME_TAKEN`. The old code let anyone "log in" as any user just by
  guessing a username during registration, with no password check.
- **Fix:** `raise AppError(409, "USERNAME_TAKEN", "Username already taken in this organization")`
  instead of returning the existing user. (Found during manual verification;
  not on the original difficulty-tier list.)

<<<<<<< HEAD
## 10. Minimum duration / `end ≤ start` not enforced
- **File/line:** `app/routers/bookings.py`, `create_booking` (~L98–100)
- **Bug:** Only `duration_hours > MAX_DURATION_HOURS` was checked;
  `MIN_DURATION_HOURS` was defined but never referenced.
- **Why wrong (rule 2):** Duration must be a whole number of hours, minimum 1;
  `end_time` must be strictly after `start_time`. A `0`-hour or negative-duration
  booking (`end <= start`) slipped through as `INVALID_BOOKING_WINDOW`-free.
- **Fix:** `if duration_hours < MIN_DURATION_HOURS or duration_hours > MAX_DURATION_HOURS: raise AppError(400, "INVALID_BOOKING_WINDOW", ...)`.

## 11. Refund rounding truncated instead of half-up, and response could diverge from the ledger
- **File/line:** `app/services/refunds.py`, `log_refund` (~L14–21); `app/routers/bookings.py`, `cancel_booking` (~L206, L216–219)
- **Bug:** `refunds.py` computed `amount_cents = int(refund_dollars * 100)`
  (truncation), while `cancel_booking` independently computed
  `round(booking.price_cents * (refund_percent / 100.0))` (Python banker's
  rounding). Two separate computations could disagree, and neither rounded
  half-cents up.
- **Why wrong (rule 6):** Refund amount must round to the nearest cent with
  half-cents rounding up, and the amount returned by the cancel response must
  equal the amount stored in the `RefundLog`.
- **Fix:** Added `compute_refund_cents()` in `refunds.py` using
  `Decimal(...).quantize(Decimal("1"), rounding=ROUND_HALF_UP)`, used by
  `log_refund` as the single source of truth; `cancel_booking` now reads
  `refund_amount_cents` from the `RefundLog` entry `log_refund` returns instead
  of recomputing it.

## 12. `get_booking` let a member read another member's booking
- **File/line:** `app/routers/bookings.py`, `get_booking` (~L164–173)
- **Bug:** The query only filtered by `Room.org_id == user.org_id`; it never
  checked booking ownership. `cancel_booking` has the ownership check,
  `get_booking` didn't.
- **Why wrong (rule 10):** Members may read only their own bookings; another
  member's booking id must 404.
- **Fix:** Added `if user.role != "admin" and booking.user_id != user.id: raise AppError(404, "BOOKING_NOT_FOUND", ...)`, matching `cancel_booking`.

## 13. Stale caches: usage report not invalidated on create, availability not invalidated on cancel
- **File/line:** `app/routers/bookings.py`, `create_booking` (~L127–129) and `cancel_booking` (~L225–227)
- **Bug:** `create_booking` invalidated only the availability cache, not the
  usage-report cache; `cancel_booking` invalidated only the usage-report cache,
  not the availability cache for the cancelled booking's room/date.
- **Why wrong (rules 12/13):** Usage report and availability must reflect
  current state "immediately". A newly created booking could be missing from
  a still-cached usage report; a cancelled booking could still show as busy in
  cached availability.
- **Fix:** `create_booking` now also calls `cache.invalidate_report(user.org_id)`;
  `cancel_booking` now also calls
  `cache.invalidate_availability(booking.room_id, booking.start_time.date().isoformat())`.

## 14. Refresh tokens were not single-use
- **File/line:** `app/auth.py` (new `redeem_refresh_token`); `app/routers/auth.py`, `refresh` (~L81–93)
- **Bug:** `refresh()` decoded and validated the presented refresh token and
  issued new tokens, but never recorded the old refresh token as used —
  only access tokens had a revocation set.
- **Why wrong (rule 8):** Refresh tokens are single-use; reusing a spent
  refresh token must return 401.
- **Fix:** Added `_revoked_refresh_tokens` set and `redeem_refresh_token(payload)`
  in `app/auth.py`, which raises `401 UNAUTHORIZED` if the token's `jti` was
  already redeemed, else marks it redeemed. `refresh()` calls it before issuing
  new tokens.

## 15. Export cross-org leak via `include_all` + `room_id`
- **File/line:** `app/services/export.py`, `fetch_bookings_raw` / `generate_export` (~L22–29, L48–52)
- **Bug:** `fetch_bookings_raw(db, room_id)` filtered only by `Booking.room_id`
  with no org scoping, and `generate_export` called it whenever `include_all`
  was set and `room_id` was provided — letting an admin pull another org's
  bookings for a guessed room id.
- **Why wrong (rule 9):** A user (including admins) may only ever read data
  belonging to their own organization, on every code path.
- **Fix:** Removed `fetch_bookings_raw`; `generate_export` now always calls the
  org-scoped `_fetch_scoped(db, org_id, ..., room_id)`, for both the
  `include_all` and normal paths.

## 16. Concurrency: unlocked shared state, non-atomic checks, and a lock-ordering deadlock
- **File/line:** `app/services/reference.py`, `app/services/stats.py`,
  `app/services/ratelimit.py`, `app/routers/bookings.py`
  (`create_booking`/`cancel_booking`), `app/services/notifications.py`
- **Bug:**
  - `reference.next_reference_code()`, `stats.record_create`/`record_cancel`,
    and `ratelimit.record_and_check` each did an unlocked read-modify-write on
    shared module-level dicts — concurrent calls could interleave between the
    read and the write (widened by artificial `time.sleep()` calls), producing
    duplicate reference codes, wrong stats, and rate limits that let extra
    requests through.
  - `_has_conflict` + `_check_quota` + insert in `create_booking`, and the
    already-cancelled check + refund logging + status update in
    `cancel_booking`, were plain read-then-write sequences with nothing
    serializing them against concurrent requests for the same room/user/booking.
  - `services/notifications.py` acquired `_email_lock` then `_audit_lock` in
    `notify_created`, but `_audit_lock` then `_email_lock` in
    `notify_cancelled` — opposite lock ordering that can deadlock two
    concurrent requests and hang the service.
- **Why wrong (rules 3/4/5/6/7/14/16):** Double-booking prevention, quota,
  rate limiting, refund/RefundLog consistency, reference-code uniqueness, and
  room stats must all hold under concurrent requests, and no combination of
  concurrent valid requests may hang the service.
- **Fix:** Added a `threading.Lock` inside each of `reference.py`, `stats.py`,
  and `ratelimit.py` guarding their respective critical sections. Added a
  module-level `_booking_lock` in `bookings.py` wrapping the
  conflict-check/quota-check/insert section of `create_booking` and the
  already-cancelled-check/refund/status-update section of `cancel_booking`
  (the latter also calls `db.refresh(booking)` after acquiring the lock so a
  concurrently-committed cancellation from another request is observed before
  the status check). Reordered `notify_cancelled` to acquire `_email_lock`
  before `_audit_lock`, matching `notify_created`, eliminating the
  lock-ordering deadlock.
=======
- **Refund rounding + response↔RefundLog mismatch**: `services/refunds.py`
  truncates (`int(...)`) and `cancel_booking` uses banker's `round`; rule 6 wants
  half-cents rounded up and the response amount equal to the stored RefundLog
  amount.
- **`get_booking` member visibility** (`bookings.py`): a member can read another
  member's booking in the same org; the per-owner check present in
  `cancel_booking` is missing here (rule 10).
- **Stale caches**: booking create does not invalidate the usage-report cache and
  cancel does not invalidate the availability cache (rules 12/13, "immediately").
- **Refresh tokens not single-use** (`routers/auth.py` refresh): rotation returns
  new tokens but never invalidates the presented refresh token (rule 8).
- **Export cross-org leak** (`services/export.py` `fetch_bookings_raw`):
  `include_all` + `room_id` bypasses org scoping (rule 9).
- **Concurrency (hard)**: no locking around the reference-code counter, in-memory
  stats, and rate-limit buckets (lost updates → duplicate reference codes, wrong
  stats, over-limit requests); conflict/quota checks and refund logging are not
  atomic under concurrent requests; and `services/notifications.py` acquires
  `_email_lock`/`_audit_lock` in opposite orders in `notify_created` vs
  `notify_cancelled`, a lock-ordering **deadlock** that can hang the service
  (rules 3/4/5/6/7/14/16). Artificial `time.sleep()` calls widen these windows.
>>>>>>> 0eb61791dc382d53d9a8fdb5353fc45cd80cebb2
