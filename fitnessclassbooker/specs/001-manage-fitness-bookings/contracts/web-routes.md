# Web Route Contract: Fitness Class and Program Booking

## Contract Conventions

- All pages return server-rendered HTML except health endpoints.
- Successful form submissions use Post/Redirect/Get with HTTP `303`.
- Invalid form input returns HTTP `200` with the same form, a validation summary,
  field-level errors, and all valid input preserved.
- Stale edit tokens or availability races return HTTP `409` with an actionable
  page. Capacity conflicts include a link to the next available occurrence.
- Missing resources return `404`; authenticated users without permission receive
  `403`; unauthenticated users are redirected to login.
- Every state-changing request is `POST`, validates an anti-forgery token, and
  includes the aggregate concurrency token where applicable.
- Dates and times are rendered in the viewer's display context with the
  trainer's offering time zone visible.
- Route IDs are UUIDs. A malformed ID returns `404` without revealing ownership.

## Authentication Routes

| Method | Route | Access | Purpose |
|--------|-------|--------|---------|
| GET | `/account/login` | Anonymous | Show login form |
| POST | `/account/login` | Anonymous | Authenticate and issue secure cookie |
| POST | `/account/logout` | Authenticated | Revoke current cookie and return to login |
| GET | `/account/access-denied` | Authenticated | Explain authorization denial |

Login accepts email, password, remember-me, and an internal return path.
Authentication errors do not reveal whether an email exists. External login,
registration, and password recovery are outside this feature.

## Trainer Routes

### Dashboard and Classes

| Method | Route | Purpose |
|--------|-------|---------|
| GET | `/trainer` | Show upcoming classes/programs, capacity alerts, and shortcuts |
| GET | `/trainer/classes` | List owned draft, active, closed, completed, and cancelled series |
| GET | `/trainer/classes/create` | Show recurring class form |
| POST | `/trainer/classes/create` | Save a new draft |
| GET | `/trainer/classes/{seriesId}` | Show definition, future occurrences, capacity, and actions |
| GET | `/trainer/classes/{seriesId}/edit` | Show editable definition and schedules |
| POST | `/trainer/classes/{seriesId}/edit-impact` | Validate proposed edit and show affected future occurrences/bookings |
| POST | `/trainer/classes/{seriesId}/edit` | Apply a previously reviewed edit |
| POST | `/trainer/classes/{seriesId}/publish` | Validate and publish, materializing 12 weeks |
| POST | `/trainer/classes/{seriesId}/booking-state` | Open or close future bookings |
| POST | `/trainer/classes/{seriesId}/close` | Stop recurrence after reviewed effective date |
| POST | `/trainer/classes/{seriesId}/cancel` | Cancel series and future scheduled occurrences after impact review |

The create/edit model includes title, description, location, time zone, local
start/end dates, default capacity, and one or more schedule rows
(`ScheduleId`, ISO weekday, local start, local end, active flag).

An edit-impact response groups:

- occurrences whose local date/time/capacity changes;
- occurrences newly added or cancelled by schedule edits;
- confirmed students affected;
- any capacity reduction that is blocked;
- the effective local date and resolved DST adjustments.

Applying an edit requires the returned impact token plus current series version.
`POST /edit-impact` creates the token with ASP.NET Core Data Protection's
time-limited protector and a route-specific purpose. The protected payload
contains the actor ID, aggregate ID/version, canonical proposed-edit hash, and
impact digest and expires after 15 minutes. `POST /edit` verifies the purpose,
signature, expiry, actor, aggregate, proposal hash, and recomputed impact digest
before commit; it never trusts client-provided counts. A missing, expired,
tampered, stale, or mismatched token returns `409 impact_changed`.

### Class Occurrences and Rosters

| Method | Route | Purpose |
|--------|-------|---------|
| GET | `/trainer/classes/{seriesId}/occurrences/{occurrenceId}` | Show occurrence details and full roster |
| POST | `/trainer/classes/{seriesId}/occurrences/{occurrenceId}/cancel` | Cancel one future occurrence |
| POST | `/trainer/classes/{seriesId}/occurrences/{occurrenceId}/bookings/{bookingId}/remove` | Remove one confirmed participant |

Cancellation/removal forms include the current version and optional reason.
Success pages retain the cancelled/removed record in the roster history.

### Programs

| Method | Route | Purpose |
|--------|-------|---------|
| GET | `/trainer/programs` | List owned draft, active, completed, and cancelled programs |
| GET | `/trainer/programs/create` | Show program form |
| POST | `/trainer/programs/preview` | Validate and show every generated session without saving/publishing |
| POST | `/trainer/programs/create` | Save a draft and its schedule definition |
| GET | `/trainer/programs/{programId}` | Show program sessions, capacity, and roster |
| GET | `/trainer/programs/{programId}/edit` | Show editable future program details |
| POST | `/trainer/programs/{programId}/edit-impact` | Show impacted sessions/enrollments |
| POST | `/trainer/programs/{programId}/edit` | Apply a reviewed pre-start edit |
| POST | `/trainer/programs/{programId}/publish` | Materialize sessions and publish |
| POST | `/trainer/programs/{programId}/enrollment-state` | Open or close enrollment |
| POST | `/trainer/programs/{programId}/cancel` | Cancel the program and future sessions |
| POST | `/trainer/programs/{programId}/enrollments/{enrollmentId}/remove` | Remove a participant from the whole program |

The program model includes title, description, location, time zone, start date,
duration weeks, capacity, and schedule rows. Preview returns ordered session
dates/times and DST adjustments. Published program schedule edits are accepted
only before the program starts and require impact review. Program edits use the
same 15-minute, time-limited Data Protection impact-token protocol as recurring
class edits, with a program-specific purpose.

### Clients

| Method | Route | Purpose |
|--------|-------|---------|
| GET | `/trainer/clients` | Search/paginate students with participation history |
| GET | `/trainer/clients/{studentId}` | Show that student's history with the current trainer |

Client search accepts a normalized name/email query, participation status, and
page cursor. Results never include history owned by another trainer.

## Student Routes

### Discovery and Class Booking

| Method | Route | Purpose |
|--------|-------|---------|
| GET | `/student` | Show next available offerings and upcoming schedule |
| GET | `/classes` | List published classes from accessible trainers |
| GET | `/classes/{seriesId}` | Show class details and suggested next occurrence |
| POST | `/classes/{seriesId}/occurrences/{occurrenceId}/book` | Confirm one occurrence booking |

The class detail page shows the exact occurrence date/time, trainer time zone,
location, places/full state, and text stating that only one class is reserved.
The POST body contains the anti-forgery token and occurrence ID; displayed
availability is not accepted as authoritative input.

Outcomes:

- **Available**: create booking and redirect to `/my-schedule` with confirmation.
- **Already booked**: redirect to the existing schedule item without duplication.
- **Final place lost**: return `409 capacity_unavailable`, preserve class
  context, and show the next available occurrence when one exists.
- **Cancelled/closed since display**: return `409 offering_unavailable` and
  explain the current status.
- **Started since display**: return `409 offering_started` and explain that
  booking is closed.
- **No access/not published**: return `404` to avoid exposing inaccessible
  offerings.

### Program Enrollment

| Method | Route | Purpose |
|--------|-------|---------|
| GET | `/programs` | List published upcoming programs from accessible trainers |
| GET | `/programs/{programId}` | Show complete session list and availability |
| POST | `/programs/{programId}/enroll` | Confirm whole-program enrollment |

The confirmation page states that every displayed session is included and no
payment occurs. Outcomes mirror class booking, except an unavailable/full
program has no partial-session fallback.

### Personal Schedule and Cancellation

| Method | Route | Purpose |
|--------|-------|---------|
| GET | `/my-schedule` | List upcoming and past class/program participation |
| GET | `/my-schedule/bookings/{bookingId}` | Show one class booking |
| POST | `/my-schedule/bookings/{bookingId}/cancel` | Cancel a future class booking |
| GET | `/my-schedule/enrollments/{enrollmentId}` | Show enrollment and all sessions |
| POST | `/my-schedule/enrollments/{enrollmentId}/withdraw` | Withdraw from a pre-start program |

Self-service changes require the authenticated student to own the participation.
Once the occurrence/program starts, the server returns `409` with trainer
contact guidance and does not release capacity.

## Health Routes

| Method | Route | Response |
|--------|-------|----------|
| GET | `/health/live` | `200 text/plain` when the process can serve |
| GET | `/health/ready` | `200` when migrations are current and PostgreSQL is reachable; otherwise `503` |

Health responses contain no connection details, exception messages, or secrets.

## Shared Error Vocabulary

Pages and tests use stable internal error keys while displaying localized,
human-readable text.

| Key | Meaning | HTTP |
|-----|---------|------|
| `validation_failed` | One or more submitted fields are invalid | 200 |
| `concurrency_conflict` | Aggregate changed since the form was shown | 409 |
| `capacity_unavailable` | Final place was taken | 409 |
| `offering_unavailable` | Offering was cancelled, closed, or unpublished | 409 |
| `already_reserved` | Active booking/enrollment already exists | 303 to existing record |
| `offering_started` | Requested self-service action is too late | 409 |
| `impact_changed` | Reviewed edit impact no longer matches current data | 409 |
| `forbidden` | User lacks role, ownership, or student ownership | 403 |
| `not_found` | Resource is absent or intentionally concealed | 404 |

Booking/enrollment handlers MUST check for the authenticated student's existing
Confirmed reservation before reporting any availability failure. The atomic
update remains authoritative for a new reservation. A uniqueness conflict or a
zero-row atomic update triggers another existing-reservation check first; when
one exists, the handler returns `already_reserved` even if that reservation
consumes the final place. Only requests with no existing reservation are
classified as full, unavailable, or started.

## Pagination and Query Limits

- Offering, roster, client, and history lists use cursor pagination with a
  default of 25 and maximum of 100 rows.
- Student class discovery requests at most 12 weeks of occurrences.
- Trainer occurrence views default to the next 12 weeks and allow explicit
  archived date ranges.
- Search input is trimmed, normalized, and limited to 100 characters.

## Accessibility Contract

- Each page has one descriptive `h1`, a logical heading order, landmarks, and a
  skip link.
- Every input has a persistent label and programmatically associated error.
- Validation summaries receive focus after an invalid submission.
- Status is conveyed with text in addition to color.
- Confirmation dialogs are server-rendered pages/forms rather than
  pointer-only controls.
- After successful Post/Redirect/Get, the destination contains a focusable
  confirmation message.
- All critical actions are usable by keyboard at 320 CSS pixels width and at
  desktop widths.
