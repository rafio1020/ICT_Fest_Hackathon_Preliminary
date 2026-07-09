# CoWork Bug-Fix Plan — Full Bug Inventory & Checklist

Full re-scan of `app/` against the business rules in `contest_overview.md` /
`README.md`. Bugs below are grouped by difficulty tier (matching the contest's
Easy 3 / Medium 5 / Hard 10 scoring) and each is checked off as solved or not.

**Progress: 23 of 23 identified bugs fixed** (6 Easy, 8 Medium, 9 Hard). All
identified bugs are solved.

---

## Easy (6/6 solved)

- [x] **Pagination broken three ways** — `app/routers/bookings.py::list_bookings`
  Used `order_by(...desc())`, `offset(page*limit)`, hard-coded `limit(10)`.
  Fixed → ascending order, `offset((page-1)*limit)`, `limit(limit)`.
- [x] **`get_booking` clobbers `start_time`** — `app/routers/bookings.py::get_booking`
  Overwrote the correct `start_time` with `created_at`. Fixed → line removed.
- [x] **Access-token lifetime wrong by 60x** — `app/auth.py::create_access_token`
  `ACCESS_TOKEN_EXPIRE_MINUTES * 60` passed to `timedelta(minutes=...)` gave
  54000s instead of 900s. Fixed → drop the `* 60`.
- [x] **Logout never invalidates token** — `app/auth.py::get_token_payload`
  Checked `payload["sub"]` against a set of `jti`s (always false). Fixed → check `jti`.
- [x] **5-minute grace window on past bookings** — `app/routers/bookings.py::create_booking`
  Allowed `start_time` up to 5 minutes in the past. Fixed → strict `start <= now` check.
- [x] **Duplicate username silently "succeeded" instead of `409 USERNAME_TAKEN`** — `app/routers/auth.py::register`
  When a username already existed in the org, the handler returned that
  existing user's `{user_id, org_id, username, role}` with `201`, **without
  checking the password**. Rule 15 requires `409 USERNAME_TAKEN`. This also
  meant anyone could fetch another user's `user_id`/`role` by "registering"
  with their username — no auth needed. Fixed → `raise AppError(409,
  "USERNAME_TAKEN", ...)`. Verified: duplicate username (even with wrong
  password) → 409; same username in a different org still succeeds; normal
  registration flow unaffected.

## Medium (8/8 solved)

- [x] **Back-to-back bookings wrongly rejected** — `app/routers/bookings.py::_has_conflict`
  Used `<=` instead of strict `<` in the overlap test. Fixed.
- [x] **Refund tier boundaries wrong** — `app/routers/bookings.py::cancel_booking`
  `<24h` gave 50% instead of 0%; exactly-48h fell into the 50% bucket. Fixed.
- [x] **UTC-offset input not converted** — `app/timeutils.py::parse_input_datetime`
  Dropped the offset instead of converting to UTC. Fixed → `astimezone(utc)`.
- [x] **No minimum-duration / non-positive-duration validation** — `app/routers/bookings.py::create_booking`
  Only `duration_hours > MAX_DURATION_HOURS` was checked. A zero-duration
  (`start == end`) or negative-duration (`end < start`) booking passed the
  "whole number of hours" check and was never rejected, producing a booking with
  `price_cents <= 0`. Rule 2 requires `end_time` strictly after `start_time` and
  duration `>= 1`. Fixed → `if duration_hours < MIN_DURATION_HOURS or duration_hours > MAX_DURATION_HOURS:`.
  Verified: zero- and negative-duration bookings → 400 `INVALID_BOOKING_WINDOW`;
  1h and 8h boundary bookings still succeed; 9h still rejected.
- [x] **`get_booking` missing owner check for members** — `app/routers/bookings.py::get_booking`
  Only filtered by `Room.org_id`; unlike `cancel_booking`, it never checked
  `booking.user_id == user.id` for non-admins, so a member could read another
  member's booking by id. Rule 10 requires `404 BOOKING_NOT_FOUND` in that case.
  Fixed → added the same owner/admin guard used in `cancel_booking`. Verified:
  a member gets 404 on another member's booking; the owner and an admin can
  still view it.
- [x] **Usage-report cache not invalidated on booking creation** — `app/routers/bookings.py::create_booking`
  Only `cache.invalidate_availability(...)` was called on create; nothing
  invalidated report cache entries for the org, so a cached `GET
  /admin/usage-report` missed newly created bookings (rule 12, "reflects the
  current state immediately"). Fixed → added `cache.invalidate_report(user.org_id)`
  alongside the availability invalidation. Verified: creating a booking
  immediately changes the room's `confirmed_bookings` count in a subsequent
  usage-report call.
- [x] **Availability cache not invalidated on cancellation** — `app/routers/bookings.py::cancel_booking`
  Only `cache.invalidate_report(...)` was called on cancel; nothing invalidated
  the room/day's cached availability, so a cancelled booking kept showing as
  busy. Violates rule 13. Fixed → added
  `cache.invalidate_availability(booking.room_id, booking.start_time.date().isoformat())`.
  Verified: availability shows the booking busy before cancel, empty right after.
- [x] **Export cross-org data leak** — `app/services/export.py::generate_export`
  When `include_all=true` **and** `room_id` was given, the code called
  `fetch_bookings_raw(db, room_id)`, which fetches every booking for that room
  id with no org filter at all — an admin from org A could export bookings from
  a room belonging to org B. Violates rule 9 (cross-org resource IDs must
  behave as non-existent). Fixed → that branch now calls
  `_fetch_scoped(db, org_id, None, room_id)` instead. Verified: an org-B admin
  exporting org-A's `room_id` gets an empty CSV (header only); an org-A admin
  exporting its own room via `include_all` still works. (`fetch_bookings_raw`
  is now unused but left in place — removing it would be an unrelated cleanup,
  not a bug fix.)

## Hard (9/9 solved)

- [x] **Refund rounding was wrong and inconsistent between two code paths** — `app/services/refunds.py::log_refund` vs `app/routers/bookings.py::cancel_booking`
  `log_refund` truncated (`int(refund_dollars * 100)`, always rounded down);
  `cancel_booking`'s response used Python's banker's `round()` (rounds half-to-even).
  Rule 6 requires "nearest cent, half-cents rounding up" **and** the response
  amount must equal the stored `RefundLog` amount — they could disagree (e.g.
  `price_cents=999`, 50% → response `500`, ledger `499`).
  Fixed → added `calculate_refund_amount_cents(price_cents, percent)` in
  `refunds.py` using exact integer math (`divmod(price_cents * percent, 100)`,
  round up when `remainder * 2 >= 100`); `log_refund` uses it to compute the
  stored amount, and `cancel_booking` now reads `refund_amount_cents` straight
  off the `RefundLog` entry `log_refund` returns instead of recomputing it —
  guaranteeing response and ledger are always identical.
  Verified: `calculate_refund_amount_cents(1001, 50) == 501` (matches the
  spec's own example); `calculate_refund_amount_cents(999, 50) == 500`
  (previously diverged between the two paths); a live cancel with a
  fractional-cent case (`price_cents=2003`, 50%) returns `refund_amount_cents
  == 1002` and the `RefundLog` entry has the identical `1002`, with exactly
  one log entry.
- [x] **Refresh tokens were not single-use** — `app/routers/auth.py::refresh`
  Rotation issued new tokens but never revoked the presented refresh token, so
  it could be replayed indefinitely. Rule 8 requires reuse → 401.
  Fixed → added a `_revoked_refresh_tokens` jti store in `app/auth.py`
  (mirroring the existing access-token one), with `revoke_refresh_token()` /
  `is_refresh_token_revoked()` helpers. The `refresh` endpoint now checks the
  presented token isn't already revoked (401 if it is), then revokes its jti
  right before issuing the new access+refresh pair.
  Verified: first refresh with a token succeeds and returns a working new
  access token; replaying the same (now-rotated) refresh token → 401; the
  newly issued refresh token itself works exactly once and its own reuse also
  → 401 (chained rotation); logout/refresh revocation stores are independent,
  so an unrelated, still-valid access token from the original login remains
  usable.
- [x] **Reference-code counter race** — `app/services/reference.py::next_reference_code`
  Read-then-sleep-then-increment on a shared dict with no lock; concurrent
  requests could read the same `current` value and emit duplicate
  `reference_code`s. Rule 7 requires uniqueness under concurrent creation.
  Fixed → wrapped the read-increment in a module-level `threading.Lock`.
  Verified: 6 concurrent bookings on 6 non-overlapping slots (different users,
  same room) all created successfully with 6 distinct reference codes.
- [x] **Stats service race** — `app/services/stats.py::record_create` / `record_cancel`
  Same read-sleep-write-without-lock pattern; concurrent creates/cancels for the
  same room could lose updates, leaving `/rooms/{id}/stats` inconsistent with
  the actual bookings. Violates rule 14. Fixed → wrapped both functions'
  bodies in a shared module-level `threading.Lock`. Verified: 6 concurrent
  creates for the same room → `total_confirmed_bookings == 6` and
  `total_revenue_cents` exactly `6 × price` (no lost updates).
- [x] **Rate limiter race** — `app/services/ratelimit.py::record_and_check`
  Bucket trim/append was not locked; concurrent requests from the same user
  could race past each other, under- or over-counting toward the 20/60s
  limit. Violates rule 5's "must hold under concurrent requests". Fixed →
  wrapped the whole trim-append-check in a module-level `threading.Lock`.
  Verified: 25 concurrent requests from one user → exactly 20 succeed and
  exactly 5 get `429 RATE_LIMITED` (no over/under count).
- [x] **Room-conflict check was not atomic** — `app/routers/bookings.py::_has_conflict` + `create_booking`
  Conflict was checked, then (after an artificial delay) the booking was
  inserted, with no locking/transaction isolation in between — two concurrent
  requests for the same slot could both pass the conflict check and both get
  inserted (double booking). Violates rule 3's "must hold under concurrent
  requests". Fixed → wrapped the conflict-check-through-insert-commit section
  in a new module-level `_create_lock` (`app/routers/bookings.py`). Verified:
  5 concurrent requests for the exact same room/slot (different users) →
  exactly 1 succeeds (201), the other 4 get `409 ROOM_CONFLICT`.
- [x] **Quota check was not atomic** — `app/routers/bookings.py::_check_quota`
  Same pattern as the conflict check — concurrent requests could each observe
  `count < QUOTA_LIMIT` and all succeed, exceeding the quota. Violates rule 4.
  Fixed → covered by the same `_create_lock` critical section as the
  conflict-check fix above (quota check happens inside the same locked
  block). Verified: 5 concurrent booking requests from one user (quota limit
  3) → exactly 3 succeed, exactly 2 get `409 QUOTA_EXCEEDED`.
- [x] **Cancel was not atomic → possible duplicate refunds** — `app/routers/bookings.py::cancel_booking`
  The `status == "cancelled"` guard was checked, then (after an artificial
  delay) the refund was logged and status updated, with no locking in
  between — two concurrent cancel requests for the same booking could both
  pass the guard and both write a `RefundLog` row. Violates rule 6's "exactly
  one RefundLog entry ... must hold under concurrent cancel requests". Fixed
  → wrapped the guard-through-commit section in a new module-level
  `_cancel_lock`, and added `db.refresh(booking)` immediately after acquiring
  the lock (the booking object was fetched *before* the lock, so without a
  refresh a queued thread would still see the stale pre-cancellation status
  from its own earlier read). Verified: 5 concurrent cancel requests for the
  same booking → exactly 1 succeeds (`200`), the other 4 get `409
  ALREADY_CANCELLED`, and the booking ends up with exactly one `RefundLog`
  entry.
- [x] **Lock-ordering deadlock** — `app/services/notifications.py`
  `notify_created` acquired `_email_lock` then `_audit_lock`; `notify_cancelled`
  acquired `_audit_lock` then `_email_lock` — the opposite order. A concurrent
  create + cancel could deadlock and hang the request threads indefinitely,
  violating rule 16 ("no combination of concurrent valid requests may hang the
  service"). Fixed → rewrote `notify_cancelled` to acquire the locks in the
  same order as `notify_created` (`_email_lock` outer, `_audit_lock` inner),
  while preserving the original side-effect order (audit write still happens
  before the email send). Verified: 20 mixed concurrent create+cancel
  requests (10 creates, 10 cancels of pre-existing bookings by their owners)
  all completed within ~5s with a hard 15s join timeout — no thread hung —
  and all 20 requests succeeded with the expected status codes.

---

## Verification performed so far
1. `pytest` — smoke test green (`tests/test_smoke.py`).
2. End-to-end script against the running app confirmed all 8 original fixes:
   pagination ordering/sizing, `get_booking` field correctness, 900s token
   lifetime, logout invalidation, back-to-back booking acceptance vs. real
   overlap rejection, 0%/100% refund boundaries, UTC-offset normalization, and
   rejection of past `start_time`.
3. Targeted script confirmed the duplicate-username fix: duplicate username in
   the same org (even with a wrong password) → `409 USERNAME_TAKEN`; same
   username in a different org still succeeds; normal registration/login flow
   unaffected.
4. Targeted script confirmed the min-duration fix: zero/negative-duration
   bookings → 400; 1h/8h boundaries still work; 9h still rejected.
5. Targeted script confirmed the remaining Medium fixes together: a member
   gets 404 viewing another member's booking (owner/admin can still view it);
   availability reflects a cancellation immediately; an export with
   `include_all=true&room_id=<foreign>` returns no rows for a foreign org,
   while an admin's own room export still works.
6. Multi-threaded concurrency script (real `threading.Thread`s, not just
   async concurrency) exercised all 5 locking fixes together against a live
   app instance: reference-code uniqueness across 6 concurrent creates on
   distinct slots; stats exactness (`count`/`revenue`) across 6 concurrent
   creates on the same room; rate-limiter exactness (20 succeed, 5 limited)
   across 25 concurrent requests from one user; room-conflict atomicity
   (exactly 1 of 5 concurrent same-slot requests succeeds); quota atomicity
   (exactly 3 of 5 concurrent requests from one user succeed); cancel
   atomicity (exactly 1 of 5 concurrent cancels of the same booking succeeds,
   exactly one `RefundLog` entry results).
7. No response shape / status-code / error-code changes vs the documented API
   contract were introduced by any fix.

## Status
All 23 identified bugs are fixed and verified. No remaining work from this
inventory. Recommended final steps before submission:
1. Re-run `pytest` and the full concurrency/deadlock verification scripts one
   more time against a clean checkout.
2. Spin up the app via `docker compose up --build` (the real grading path) and
   smoke-test a few endpoints manually, since local verification here ran
   against a Python 3.13 venv with newer dependency versions (only
   `requirements.txt`'s pinned 3.11-targeted versions ship in the container —
   `requirements.txt` itself was never modified).
3. Review `git diff` once more to confirm no unrelated/refactor changes crept
   in, then commit.
