

AAI presents IUT 12th ICT Fest
powered by Therap (BD) Ltd.
organized by IUT Computer Society
Bdapps presents
Agentic AI Hackathon
powered by Codex
## Preliminary Round
## Problem Statement
CoWork: Multi-Tenant Coworking Space Booking API
## Repository Link
https://github.com/AlchemistReturns/ICT_Fest_Hackathon_
## Preliminary
## Duration
4 hours, 6:00 PM to 10:00 PM
Document version:  Preliminary Round
## July 9, 2026

IUT 12th ICT FestBdapps Agentic AI Hackathon
## 1  Overview
This is a bug fix challenge.  Participants are given a broken codebase and must find the bugs,
understand  why  they  are  broken,  and  fix  them.   There  are  bugs  hidden  across  the  project,
ranging from easy one-liners to subtle concurrency and logic issues.  Participants do not need
to add features or refactor anything.  Find the bugs.  Fix them.  That is the entire task.
Grading is automatic and black-box.  A grader will build your submitted repository and
talk  to  it  only  through  the  API.  It  will  assert  behavior  against  the  business  rules  and  API
contract  described  in  this  document  (Sections  3  and  4).   Your  fixes  must  preserve  this
contract exactly - paths, status codes, error codes, and JSON field names must not change.
## 2  The Project
CoWork  is  a  REST  API  for  managing  bookable  rooms  inside  a  coworking  space,  supporting
multiple tenant organizations.  Each organization has its own rooms, staff (admins), and mem-
bers.  Members book rooms for time slots; admins manage rooms and pull usage reports.
Stack:  Python 3.11, FastAPI, SQLAlchemy, SQLite (single file, no external database service).
Authentication is handled via JWT access and refresh tokens (HS256).
Out  of  scope:   There  is  no  real  payment  gateway  (refunds  are  calculated  and  logged,  not
processed),  and there is no real email delivery (a “send confirmation email” step only logs a
line).
## File Structure
app/
|-- main.py           # FastAPI app entrypoint
|-- config.py         # Environment/config loading
|-- database.py       # Database engine and session setup
|-- models.py         # SQLAlchemy database models
|-- schemas.py        # Pydantic request/response schemas
|-- serializers.py    # Model -> response object conversion
|-- auth.py           # JWT creation, password hashing, auth dependency
|-- cache.py          # In-memory caching for reports/availability
|-- errors.py         # Application error types and handler
|-- timeutils.py       # Datetime parsing/normalization helpers
|-- routers/          # API route handlers (auth, rooms, bookings, admin, health)
‘-- services/         # Business logic (refunds, stats, rate limiting,
# reference codes, export, notifications)
## 3  Data Model
-  Organization: id, name (unique)
-  User: id, org
id, username (unique within org), hashedpassword, role (admin| member),
createdat
-  Room: id, orgid, name, capacity, hourlyratecents
-  Booking: id, room
id, userid, starttime, endtime, status (confirmed | cancelled),
referencecode, pricecents, createdat
-  RefundLog: id, bookingid, amountcents, status (processed| failed), processedat
## 1

IUT 12th ICT FestBdapps Agentic AI Hackathon
## 4  Business Rules
These are the rules the API is expected to follow.  Some bugs are deviations from these rules -
use them as your source of truth when deciding whether behavior is correct.
-  Datetimes.  All API datetimes are ISO 8601.  Input datetimes carrying a UTC offset must
be  converted  to  UTC  before  storage  or  comparison;  naive  input  is  treated  as  UTC.  All
response datetimes are UTC with an explicit UTC designator.
-  Booking price. price
cents = hourlyratecents × durationhours.  Duration must
be  a  whole  number  of  hours,  minimum  1,  maximum  8. endtime  must  be  strictly  after
start
time. starttime must be strictly in the future at request time - no grace window.
-  No double-booking. Two confirmed bookings for the same room overlap iff existing.start
< new.end AND new.start < existing.end.  Back-to-back bookings are allowed.  Conflict
→ 409 ROOMCONFLICT. Must hold under concurrent requests.
-  Booking quota. A member may hold at most 3 confirmed bookings with starttime in the
window (now, now + 24h], across all rooms in their org.  Violation → 409 QUOTAEXCEEDED.
Must hold under concurrent requests.
-  Rate limit. POST /bookings is limited to 20 requests per rolling 60 seconds per user (all
requests count).  Excess → 429 RATE
LIMITED. Must hold under concurrent requests.
-  Cancellation refund policy.  Only the booking’s owner or an admin of the same org may
cancel.  Notice = starttime − cancellation time:
-  notice ≥ 48 hours → 100% refund
-  24 hours ≤ notice < 48 hours → 50% refund
-  notice < 24 hours → 0% refund
Refund amount rounds to the nearest cent, half-cents rounding up.  Cancelling an already-
cancelled  booking → 409 ALREADY
CANCELLED.  A  cancelled  booking  has  exactly  one  Re-
fundLog  entry,  and  the  amount  returned  by  the  cancel  response  must  equal  the  amount
stored in the RefundLog.  Must hold under concurrent cancel requests for the same booking.
-  Reference codes.  Every booking’s reference
code is unique, including under concurrent
creation.
-  Auth.  Tokens are JWTs (HS256) with claims sub (user id, string), org (org id), role, jti
(unique  per  token), iat, exp, type  (access | refresh).   Access  tokens  expire  in  exactly
900 seconds.  Refresh tokens expire in 7 days.  Logout immediately invalidates the presented
access token (subsequent use → 401).  Refresh tokens are single-use:  refreshing returns a
new access and refresh token and invalidates the presented refresh token (reuse → 401).
-  Multi-tenancy.  A user (including admins) may only ever read or act on data belonging to
their own organization, on every code path.  Cross-org resource IDs behave as non-existent
## (→ 404).
-  Booking  visibility.   Members  may  read  and  cancel  only  their  own  bookings  (another
member’s booking id→ 404 BOOKING
NOTFOUND). Admins may read and cancel any booking
in their org.
-  Pagination & ordering. GET /bookings takes page (default 1) and limit (default 10,
max  100).   Items  are  the  caller’s  own  bookings  sorted  ascending  by starttime  (ties  by
ascending id).  Sequential pages never skip or repeat items.  Response includes total.
## 2

IUT 12th ICT FestBdapps Agentic AI Hackathon
-  Usage report. GET /admin/usage-report?from=...&to=...  returns,  per room in the
caller’s org (including rooms with zero bookings), the count and summed price
cents of
confirmed bookings starting in [from, to] (UTC, inclusive).  Must reflect the current state
immediately.
-  Availability. GET /rooms/{id}/availability?date=...   returns  the  room’s  confirmed
bookings  starting  on  that  UTC  date  as  busy  intervals,  sorted  ascending,  reflecting  the
current state immediately.
-  Room stats. GET /rooms/{id}/stats returns the room’s current count of confirmed book-
ings and their summed price
cents, always consistent with the bookings themselves, in-
cluding after bursts of concurrent activity.
-  Registration. POST /auth/register with an unknown org
name creates the org and the
user as admin; with a known orgname it joins the caller as member.  A duplicate username
within the org → 409 USERNAMETAKEN.
-  Liveness.  The service must respond to all endpoints at all times; no combination of con-
current valid requests may hang the service.
5  API Contract
## Endpoints
Method  PathAuthDescription
POST   /auth/registerNoRegister org admin or join org as member
POST   /auth/loginNoReturns access + refresh token
POST   /auth/refreshNo(to-
kenin
body)
Rotates tokens
POST   /auth/logoutYesInvalidates presented access token
GET    /roomsYesList rooms in caller’s org
POST   /roomsYes   (ad-
min)
Create a room
GET    /rooms/{id}/availabilityYesBusy intervals for a date
GET    /rooms/{id}/statsYesLive confirmed-booking count & revenue
POST   /bookingsYesCreate a booking
GET    /bookingsYesCaller’s bookings, paginated
GET    /bookings/{id}YesSingle booking incl.  refunds
POST   /bookings/{id}/cancelYesCancel + refund calculation
GET    /admin/usage-reportYes   (ad-
min)
Per-room usage/revenue for range
GET    /admin/exportYes   (ad-
min)
Bookings CSV; room
id, includeall
GET    /healthNo{"status":  "ok"}
For all authenticated endpoints, pass the token in the Authorization header:
## Authorization: Bearer <your_token>
## 3

IUT 12th ICT FestBdapps Agentic AI Hackathon
## Request / Response Schemas
- POST /auth/register body{orgname, username, password}→{userid, orgid, username,
role}
- POST /auth/login body{org
name, username, password}→{accesstoken, refreshtoken,
tokentype:  "bearer"}; bad credentials → 401 INVALIDCREDENTIALS
- POST /auth/refresh body {refreshtoken} → same shape as login
-  Room: {id, orgid, name, capacity, hourlyratecents}; POST /rooms body {name,
capacity, hourlyratecents}
## •  Availability: {room
id, date, busy:  [{starttime, endtime}, ...]}
-  Stats: {roomid, totalconfirmedbookings, totalrevenuecents}
- POST /bookings body{roomid, starttime, endtime}→ Booking: {id, referencecode,
roomid, userid, starttime, endtime, status, pricecents, createdat}
- GET /bookings → {items:  [Booking, ...], page, limit, total}
- GET /bookings/{id}→ Booking plus refunds:  [{amount
cents, status, processedat},
## ...]
- POST /bookings/{id}/cancel → {id, status:  "cancelled", refundpercent,
refundamountcents}
-  Usage report → {from, to, rooms:  [{roomid, roomname, confirmedbookings,
revenuecents}, ...]}
-  Export CSV header (exact): id,referencecode,roomid,userid, starttime,
endtime,status,pricecents
## Errors
Application errors return JSON{"detail":  <string>, "code":  <CODE>} with codes: USERNAME
## TAKEN
## (409), INVALID
## CREDENTIALS (401), ROOMCONFLICT (409), QUOTAEXCEEDED (409), RATELIMITED
(429), ALREADYCANCELLED (409), BOOKINGNOTFOUND (404), ROOMNOTFOUND (404), FORBIDDEN
## (403), INVALID
BOOKINGWINDOW (400 — past start, non-whole/out-of-range duration, or endtime
≤ starttime).  Missing/invalid/expired/blacklisted tokens → 401.  Framework validation er-
rors (422) may use FastAPI’s default shape.
Fixes must preserve this contract exactly; grading is black-box against it.
## 4

IUT 12th ICT FestBdapps Agentic AI Hackathon
6  Running the Project
## With Docker (recommended)
docker compose up --build
Without Docker (Python 3.11)
python -m venv .venv
## # Windows
.venv\Scripts\activate
# macOS / Linux
source .venv/bin/activate
pip install -r requirements.txt
uvicorn app.main:app --reload
App runs at http://localhost:8000.  Interactive API docs at http://localhost:8000/docs.
7  How to Test
Use the Swagger UI at /docs, curl, or clients like Postman.
curl Example
## # Register
curl -X POST http://localhost:8000/auth/register \
-H "Content-Type: application/json" \
## -d ’{"org_name": "acme", "username": "alice", "password": "pass123"}’
## # Login
curl -X POST http://localhost:8000/auth/login \
-H "Content-Type: application/json" \
## -d ’{"org_name": "acme", "username": "alice", "password": "pass123"}’
# Use the token
curl http://localhost:8000/rooms \
-H "Authorization: Bearer <TOKEN>"
## 8  The Challenge
There are multiple bugs in the codebase, distributed across the difficulty tiers below.  Each bug
causes clearly observable, wrong behavior when interacting with the API.
What counts as a valid fix:
-  The fixed code produces the correct behavior described in Sections 3–4
-  Only the broken code should be changed - do not refactor or rewrite unrelated code
-  The API contract (paths, status codes, error codes, JSON field names) must remain exactly
as specified
## 5

IUT 12th ICT FestBdapps Agentic AI Hackathon
## 9  Submission
-  Fork the preliminary-round repository to your own GitHub account: https://github.com/
AlchemistReturns/ICT_Fest_Hackathon_Preliminary
-  Leave  the  fork’s  network  so  your  copy  is  no  longer  linked  to  the  original  repository
(GitHub → your repo’s Settings → scroll to Danger Zone → Leave fork network ).  Do this
before you start editing.
-  Fix the bugs in your repository.
-  Your repository may be kept private during the competition, if you prefer.
-  You must make it public within 1 hour of the competition ending - repositories that remain
private after this window will not be evaluated.
-  Submit your repository URL via the provided Google Form link.
10  Scoring & Tie-Breaking
Points are awarded per bug based on difficulty:
Difficulty   Points each
## Easy3
## Medium5
## Hard10
Tie-Breaking
Ties are resolved in the following order:
-  Difficulty of bugs solved - the participant who fixed harder bugs ranks higher.
- bug
report.md  (optional)  -  if  a  tie  remains  after  step  1,  participants  who  submitted  a
bugreport.md in the root of their repository go through manual evaluation.
bug
report.md should include, for each bug found:
-  Which file(s)/line(s) the bug is on
-  What the bug was and why it caused incorrect behavior
-  How it was fixed
Manual evaluation of bugreport.md is the final tie-breaking mechanism.
Good luck.
## 6