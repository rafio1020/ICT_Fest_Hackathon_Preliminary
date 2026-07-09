# CoWork Bug-Fix Plan — Full Bug Inventory & Checklist

Full re-scan of `app/` against the business rules in `contest_overview.md` /
`README.md`. Bugs below are grouped by difficulty tier (matching the contest's
Easy 3 / Medium 5 / Hard 10 scoring) and each is checked off as solved or not.

**Progress: 8 of 23 identified bugs fixed** (5 Easy, 3 Medium). Everything
Hard-tier, plus a handful of Medium items and one newly-found Easy item, are
still open.

---

## Easy (5/6 solved)

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
- [ ] **Duplicate username silently "succeeds" instead of `409 USERNAME_TAKEN`** — `app/routers/auth.py::register` (~L32–43)
  When a username already exists in the org, the handler returns that existing
  user's `{user_id, org_id, username, role}` with `201`, **without checking the
  password**. Rule 15 requires `409 USERNAME_TAKEN`. This also means anyone can
  fetch another user's `user_id`/`role` by "registering" with their username —
  no auth needed. *(Newly found this pass — not yet fixed.)*

## Medium (3/8 solved)

- [x] **Back-to-back bookings wrongly rejected** — `app/routers/bookings.py::_has_conflict`
  Used `<=` instead of strict `<` in the overlap test. Fixed.
- [x] **Refund tier boundaries wrong** — `app/routers/bookings.py::cancel_booking`
  `<24h` gave 50% instead of 0%; exactly-48h fell into the 50% bucket. Fixed.
- [x] **UTC-offset input not converted** — `app/timeutils.py::parse_input_datetime`
  Dropped the offset instead of converting to UTC. Fixed → `astimezone(utc)`.
- [ ] **No minimum-duration / non-positive-duration validation** — `app/routers/bookings.py::create_booking` (~L89–94)
  Only `duration_hours > MAX_DURATION_HOURS` is checked. A zero-duration
  (`start == end`) or negative-duration (`end < start`) booking passes the
  "whole number of hours" check and is never rejected, producing a booking with
  `price_cents <= 0`. Rule 2 requires `end_time` strictly after `start_time` and
  duration `>= 1`. Needs an added `duration_hours < MIN_DURATION_HOURS` check.
- [ ] **`get_booking` missing owner check for members** — `app/routers/bookings.py::get_booking`
  Only filters by `Room.org_id`; unlike `cancel_booking`, it never checks
  `booking.user_id == user.id` for non-admins. A member can read another
  member's booking by id. Rule 10 requires `404 BOOKING_NOT_FOUND` in that case.
- [ ] **Usage-report cache not invalidated on booking creation** — `app/routers/bookings.py::create_booking`
  Only `cache.invalidate_availability(...)` is called on create; nothing
  invalidates `cache` report entries for the org. A cached `GET
  /admin/usage-report` will miss newly created bookings, violating rule 12
  ("reflects the current state immediately").
- [ ] **Availability cache not invalidated on cancellation** — `app/routers/bookings.py::cancel_booking`
  Only `cache.invalidate_report(...)` is called on cancel; nothing invalidates
  the room/day's cached availability, so a cancelled booking keeps showing as
  busy. Violates rule 13.
- [ ] **Export cross-org data leak** — `app/services/export.py::generate_export` / `fetch_bookings_raw`
  When `include_all=true` **and** `room_id` is given, `fetch_bookings_raw` fetches
  every booking for that room id with no org filter at all — an admin from org A
  can export bookings from a room belonging to org B. Violates rule 9
  (cross-org resource IDs must behave as non-existent).

## Hard (0/9 solved)

- [ ] **Refund rounding is wrong and inconsistent between two code paths** — `app/services/refunds.py::log_refund` vs `app/routers/bookings.py::cancel_booking`
  `log_refund` truncates (`int(refund_dollars * 100)`, always rounds down);
  `cancel_booking`'s response uses Python's banker's `round()` (rounds half-to-even).
  Rule 6 requires "nearest cent, half-cents rounding up" **and** the response
  amount must equal the stored `RefundLog` amount — currently they can disagree
  (e.g. `price_cents=999`, 50% → response `500`, ledger `499`). Needs one shared,
  correctly-rounded (half-up) calculation used by both.
- [ ] **Refresh tokens are not single-use** — `app/routers/auth.py::refresh`
  Rotation issues new tokens but never revokes the presented refresh token, so it
  can be replayed indefinitely. Rule 8 requires reuse → 401. Needs a revocation
  store for refresh `jti`s (mirroring the access-token one) plus a check.
- [ ] **Reference-code counter race** — `app/services/reference.py::next_reference_code`
  Read-then-sleep-then-increment on a shared dict with no lock; concurrent
  requests can read the same `current` value and emit duplicate
  `reference_code`s. Rule 7 requires uniqueness under concurrent creation.
- [ ] **Stats service race** — `app/services/stats.py::record_create` / `record_cancel`
  Same read-sleep-write-without-lock pattern; concurrent creates/cancels for the
  same room can lose updates, leaving `/rooms/{id}/stats` inconsistent with the
  actual bookings. Violates rule 14.
- [ ] **Rate limiter race** — `app/services/ratelimit.py::record_and_check`
  Bucket trim/append is not locked; concurrent requests from the same user can
  race past each other, under- or over-counting toward the 20/60s limit.
  Violates rule 5's "must hold under concurrent requests".
- [ ] **Room-conflict check is not atomic** — `app/routers/bookings.py::_has_conflict` + `create_booking`
  Conflict is checked, then (after an artificial delay) the booking is inserted,
  with no locking/transaction isolation in between. Two concurrent requests for
  the same slot can both pass the conflict check and both get inserted — double
  booking. Violates rule 3's "must hold under concurrent requests".
- [ ] **Quota check is not atomic** — `app/routers/bookings.py::_check_quota`
  Same pattern as the conflict check — concurrent requests can each observe
  `count < QUOTA_LIMIT` and all succeed, exceeding the quota. Violates rule 4.
- [ ] **Cancel is not atomic → possible duplicate refunds** — `app/routers/bookings.py::cancel_booking`
  The `status == "cancelled"` guard is checked, then (after an artificial delay)
  the refund is logged and status updated, with no locking in between. Two
  concurrent cancel requests for the same booking can both pass the guard and
  both write a `RefundLog` row. Violates rule 6's "exactly one RefundLog entry
  ... must hold under concurrent cancel requests".
- [ ] **Lock-ordering deadlock** — `app/services/notifications.py`
  `notify_created` acquires `_email_lock` then `_audit_lock`; `notify_cancelled`
  acquires `_audit_lock` then `_email_lock` — the opposite order. A concurrent
  create + cancel can deadlock and hang the request threads indefinitely,
  violating rule 16 ("no combination of concurrent valid requests may hang the
  service").

---

## Verification performed so far
1. `pytest` — smoke test green (`tests/test_smoke.py`).
2. End-to-end script against the running app confirmed all 8 solved fixes:
   pagination ordering/sizing, `get_booking` field correctness, 900s token
   lifetime, logout invalidation, back-to-back booking acceptance vs. real
   overlap rejection, 0%/100% refund boundaries, UTC-offset normalization, and
   rejection of past `start_time`.
3. No response shape / status-code / error-code changes vs the documented API
   contract were introduced by any fix.

## Suggested order for the remaining work
1. Finish Easy: duplicate-username `409 USERNAME_TAKEN` (single-block fix, high value).
2. Medium correctness items: min-duration validation, `get_booking` ownership
   check, export org-scoping — each small, isolated, low risk.
3. Medium cache-staleness items: invalidate report cache on create, invalidate
   availability cache on cancel.
4. Hard tier, roughly in order of risk/complexity: refund rounding
   unification, refresh-token single-use, then the concurrency-locking cluster
   (reference codes → stats → rate limiter → conflict/quota checks → cancel),
   and finally the notifications deadlock (fix lock ordering to match in both
   functions).
