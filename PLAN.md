# CoWork Bug-Fix Plan — Fast One-Liner Tier

Status: **Completed** — all 8 fixes below applied, verified via `pytest` and an
end-to-end script, and committed (see `bug_report.md` for the write-up used for
tie-breaking).

## Context

This repo is the ICT Fest Agentic AI Hackathon **preliminary bug-fix challenge**
(`contest_overview.md`). CoWork is a FastAPI + SQLAlchemy + SQLite multi-tenant
booking API. The task: find bugs where behavior deviates from the business rules
in the README / overview, and fix **only** the broken lines — no refactors, no
contract changes (paths, status codes, error codes, JSON field names must stay
identical). Grading is black-box against the API. Scoring: Easy 3 / Medium 5 /
Hard 10 pts.

This plan targeted the fast, high-confidence single-line fixes — deterministic
logic bugs the grader will catch immediately, each a 1-line change with
near-zero risk. Harder bugs (concurrency, rounding, caching) are catalogued at
the bottom as *deferred*.

## Fixes (all one-liners)

### 1. Pagination is broken three ways — `app/routers/bookings.py` (list_bookings, ~L136–140)
Rule 11: ascending by `start_time`, page N = items `[(N−1)·L, N·L)`.
Was:
```python
items = (
    base.order_by(Booking.start_time.desc(), Booking.id.asc())  # wrong: desc
    .offset(page * limit)                                        # wrong: skips page 1
    .limit(10)                                                   # wrong: ignores limit
    .all()
)
```
Fix → `.order_by(Booking.start_time.asc(), Booking.id.asc())`, `.offset((page - 1) * limit)`, `.limit(limit)`.

### 2. Booking detail clobbers `start_time` — `app/routers/bookings.py` (get_booking, L166)
`response["start_time"] = iso_utc(booking.created_at)` overwrote the real
start_time with `created_at`. **Deleted this line** — `serialize_booking` already
sets the correct value.

### 3. Access-token lifetime was 54000s, not 900s — `app/auth.py` (create_access_token, L50)
`timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES * 60)` with `ACCESS_TOKEN_EXPIRE_MINUTES = 15`
→ 900 *minutes*. Rule 8 requires `exp − iat == 900` seconds.
Fix → `lifetime = timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)`.

### 4. Logout never invalidated the token — `app/auth.py` (get_token_payload, L97)
`revoke_access_token` stores `payload["jti"]`, but the check was
`if payload.get("sub") in _revoked_tokens`. It compared the user id against a set
of jtis, so logout was a no-op (rule 8). Fix → `if payload.get("jti") in _revoked_tokens`.

### 5. Back-to-back bookings wrongly rejected — `app/routers/bookings.py` (_has_conflict, L50)
Rule 3: overlap iff `existing.start < new.end AND new.start < existing.end`
(back-to-back allowed). Was using `<=`:
`if b.start_time <= end and start <= b.end_time`. Fix → strict `<` on both:
`if b.start_time < end and start < b.end_time`.

### 6. Refund tiers wrong — `app/routers/bookings.py` (cancel_booking, L201 & L206)
Rule 6: ≥48h→100%, [24,48)→50%, <24h→**0%**. Was:
```python
if notice_hours > 48:            # exactly 48h wrongly fell through to 50%
    refund_percent = 100
elif notice >= timedelta(hours=24):
    refund_percent = 50
else:
    refund_percent = 50          # wrong: <24h must be 0
```
Fix → first branch `if notice >= timedelta(hours=48):`, final `else: refund_percent = 0`.

### 7. UTC-offset datetimes not converted — `app/timeutils.py` (parse_input_datetime, L13)
Rule 1: offset-aware input must be **converted** to UTC. Was dropping the
tzinfo (`dt.replace(tzinfo=None)`), so `12:00+02:00` was stored as `12:00` instead
of `10:00`. Fix → `dt = dt.astimezone(timezone.utc).replace(tzinfo=None)`
(`timezone` already imported).

### 8. 5-minute past-booking grace window — `app/routers/bookings.py` (create_booking, L86)
Rule 2: `start_time` must be strictly in the future, **no grace**. Was allowing
up to 5 min in the past: `if start <= now - timedelta(seconds=300)`.
Fix → `if start <= now`.

## Deferred (higher-tier bugs found, not fixed in this pass)
Each is real but multi-line / riskier — left for a later round:
- **Min-duration / `end ≤ start` not enforced** (bookings.py L89–94): only `> MAX`
  checked; `< MIN_DURATION_HOURS` and zero/negative duration slip through.
- **Refund rounding + response↔RefundLog mismatch** (refunds.py L17 truncates;
  bookings.py L208 uses banker's `round`) — rule 6 wants half-cents-up and equal amounts.
- **get_booking member visibility** (bookings.py L156–163): a member can read another
  member's booking in-org (rule 10) — cancel has the check, get is missing it.
- **Report cache stale on create / availability cache stale on cancel**
  (bookings.py L121, L217; cache.py) — rules 12/13 "immediately".
- **Refresh tokens not single-use** (routers/auth.py refresh, L81–93) — rule 8.
- **Export cross-org leak** (services/export.py `fetch_bookings_raw`, L22–29 / L49–50):
  `include_all` + `room_id` ignores org scoping (rule 9).
- **Concurrency (hard)**: no locking around reference counter, stats, rate-limit
  buckets, conflict/quota checks, double refund; and a **lock-ordering deadlock**
  between `notify_created` and `notify_cancelled` (notifications.py). Artificial
  `time.sleep()` calls widen these race windows.

## Verification performed
1. `pytest` — smoke test green (`tests/test_smoke.py` covers
   register→login→room→booking→list).
2. End-to-end script against the running app confirmed:
   - Pagination: no skip/repeat, ascending order, correct page sizes (fix 1).
   - `GET /bookings/{id}` → `start_time` ≠ `created_at` (fix 2).
   - Decoded access token → `exp − iat == 900` (fix 3).
   - `POST /auth/logout` then reuse token → 401 (fix 4).
   - Back-to-back bookings → both 201; true overlap still → 409 (fix 5).
   - Cancel <24h notice → 0%; ≥48h notice → 100% (fix 6).
   - Booking with `+05:00` offset stored/returned as the equivalent UTC instant (fix 7).
   - Past `start_time` → 400 `INVALID_BOOKING_WINDOW` (fix 8).
3. No response shape / status-code / error-code changes vs the documented API contract.
