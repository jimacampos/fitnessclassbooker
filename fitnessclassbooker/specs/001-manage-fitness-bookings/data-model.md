# Data Model: Fitness Class and Program Booking

## Conventions

- Primary keys are application-generated UUIDs.
- Persistent instants use UTC `timestamptz`; application code represents them as
  Noda Time `Instant`.
- Trainer-entered dates use PostgreSQL `date`/Noda Time `LocalDate`; schedule
  times use PostgreSQL `time`/Noda Time `LocalTime`.
- Time zones are canonical IANA identifiers such as `America/Denver`.
- Mutable aggregate roots use PostgreSQL `xmin` as an EF Core optimistic
  concurrency token.
- Lifecycle records are statused, not deleted. A draft with no dependent history
  is the only domain record that may be hard-deleted.
- All string status columns have database check constraints matching the values
  below.

## Identity and Access

### ApplicationUser

Extends the ASP.NET Core Identity user.

| Field | Type | Rules |
|-------|------|-------|
| Id | UUID | Primary key |
| Email | string | Required, normalized unique identity email |
| DisplayName | string | Required, 1-100 characters |
| TimeZoneId | string | Required valid IANA zone |
| CreatedAt | Instant | Required |
| LastSignedInAt | Instant? | Updated after successful authentication |

Roles use Identity's role tables. Each active user must have exactly one of
`Trainer` or `Student` in v1.

### TrainerStudentAccess

Represents an externally provisioned grant that lets a student see a trainer's
published offerings. Creating/inviting this relationship is outside v1.

| Field | Type | Rules |
|-------|------|-------|
| Id | UUID | Primary key |
| TrainerId | UUID | Required Trainer user |
| StudentId | UUID | Required Student user |
| Status | enum | `Active`, `Revoked` |
| GrantedAt | Instant | Required |
| RevokedAt | Instant? | Required only when revoked |

**Constraints and indexes**:

- Unique `(TrainerId, StudentId)`.
- Index `(StudentId, Status)` for student offering discovery.
- A trainer and student ID cannot be the same.

## Recurring Classes

### RecurringClassSeries

The trainer-owned definition and lifecycle of a weekly class.

| Field | Type | Rules |
|-------|------|-------|
| Id | UUID | Primary key |
| TrainerId | UUID | Required Trainer owner |
| Title | string | Required, 1-120 characters |
| Description | string? | Maximum 2,000 characters |
| Location | string? | Maximum 500 characters |
| TimeZoneId | string | Required valid IANA zone |
| StartDate | LocalDate | Required |
| EndDate | LocalDate? | Must be on/after StartDate |
| DefaultCapacity | integer | Required, 1-1,000 |
| Status | enum | `Draft`, `Published`, `Closed`, `Cancelled`, `Completed` |
| AcceptingBookings | boolean | False unless Published |
| PublishedAt | Instant? | Set on first publication |
| ClosedAt | Instant? | Set when bookings close/series closes |
| CancelledAt | Instant? | Set when entire series is cancelled |
| CreatedAt | Instant | Required |
| UpdatedAt | Instant | Required |
| Version | xmin | Concurrency token |

**Relationships**:

- One trainer owns many series.
- A series owns one or more active schedule rows before publication.
- A series owns materialized occurrences.

**Validation**:

- Publication requires at least one active schedule.
- `EndDate`, when present, must include at least one generated occurrence.
- Capacity cannot be reduced below the highest confirmed count of affected
  future occurrences.
- `AcceptingBookings` can be true only while status is `Published`.

### RecurringClassSchedule

One weekly meeting pattern within a series. A stable schedule ID lets edits map
existing occurrences to the same logical weekly slot.

| Field | Type | Rules |
|-------|------|-------|
| Id | UUID | Primary key |
| SeriesId | UUID | Required parent |
| DayOfWeek | integer | ISO 1 (Monday) through 7 (Sunday) |
| StartTime | LocalTime | Required |
| EndTime | LocalTime | Required; after StartTime |
| IsActive | boolean | Removed schedules are retained when referenced |
| CreatedAt | Instant | Required |
| UpdatedAt | Instant | Required |

**Constraints and indexes**:

- Unique active `(SeriesId, DayOfWeek, StartTime)`.
- Overnight schedules are out of scope; `EndTime > StartTime`.

**Serialized application invariant**: No active schedules in the same series may
overlap on the same weekday. Every schedule create/edit/remove transaction first
locks the parent series row, then validates all sibling schedules before saving.
The parent lock serializes concurrent edits that a row-local check constraint
cannot detect.

**Occurrence formula**:

For a schedule, its base date is the first matching `DayOfWeek` on or after the
series `StartDate`. Occurrence sequence `n` is `base date + (7 * n)` days. This
stable sequence permits a schedule's weekday/time to change while preserving
the identity and participant history of its future occurrences.

### ClassOccurrence

A concrete, reservable meeting generated from one recurring schedule.

| Field | Type | Rules |
|-------|------|-------|
| Id | UUID | Primary key |
| SeriesId | UUID | Required parent |
| ScheduleId | UUID | Required source schedule |
| Sequence | integer | Zero-based occurrence within schedule |
| LocalDate | LocalDate | Resolved trainer-local date |
| StartsAt | Instant | Required UTC instant |
| EndsAt | Instant | Required, after StartsAt |
| Capacity | integer | Snapshot, 1-1,000 |
| ConfirmedCount | integer | Required, default 0 |
| Status | enum | `Scheduled`, `Cancelled`, `Completed` |
| IsScheduleException | boolean | True when retained outside current pattern |
| CancelledAt | Instant? | Set on cancellation |
| CompletedAt | Instant? | Set after meeting completion |
| CreatedAt | Instant | Required |
| UpdatedAt | Instant | Required |
| Version | xmin | Concurrency token |

**Constraints and indexes**:

- Unique `(ScheduleId, Sequence)`.
- Unique `(SeriesId, StartsAt)` to prevent duplicate materialization.
- Check `0 <= ConfirmedCount AND ConfirmedCount <= Capacity`.
- Check `EndsAt > StartsAt`.
- Index `(SeriesId, Status, StartsAt)`.
- Index `(Status, StartsAt)` for horizon/completion maintenance.

An occurrence snapshots capacity and concrete instants so historical records do
not change when the series definition changes. Confirmed future occurrences are
updated only through the trainer's reviewed edit-impact workflow.

### ClassBooking

A student's reservation for exactly one occurrence.

| Field | Type | Rules |
|-------|------|-------|
| Id | UUID | Primary key |
| OccurrenceId | UUID | Required |
| StudentId | UUID | Required Student |
| Status | enum | `Confirmed`, `Cancelled`, `Removed`, `Completed` |
| BookedAt | Instant | Required |
| CancelledAt | Instant? | Student cancellation time |
| RemovedAt | Instant? | Trainer removal time |
| CompletedAt | Instant? | Set when participation becomes historical |
| ChangedByUserId | UUID? | Actor for cancellation/removal |
| ChangeReason | string? | Maximum 500 characters |
| CreatedAt | Instant | Required |
| UpdatedAt | Instant | Required |
| Version | xmin | Concurrency token |

**Constraints and indexes**:

- Partial unique `(OccurrenceId, StudentId)` where `Status = 'Confirmed'`.
- Index `(StudentId, Status, OccurrenceId)` for student schedules.
- Index `(OccurrenceId, Status)` for rosters.

## Programs

### Program

A finite, trainer-owned offering whose one enrollment covers all sessions.

| Field | Type | Rules |
|-------|------|-------|
| Id | UUID | Primary key |
| TrainerId | UUID | Required Trainer owner |
| Title | string | Required, 1-120 characters |
| Description | string? | Maximum 2,000 characters |
| Location | string? | Maximum 500 characters |
| TimeZoneId | string | Required valid IANA zone |
| StartDate | LocalDate | Required |
| DurationWeeks | integer | Required, 1-52 |
| Capacity | integer | Required, 1-1,000 |
| ConfirmedCount | integer | Required, default 0 |
| Status | enum | `Draft`, `Published`, `Cancelled`, `Completed` |
| AcceptingEnrollments | boolean | False unless Published and not started |
| PublishedAt | Instant? | Set on first publication |
| EnrollmentClosedAt | Instant? | Set when enrollment closes |
| CancelledAt | Instant? | Set on cancellation |
| CompletedAt | Instant? | Set after final session |
| CreatedAt | Instant | Required |
| UpdatedAt | Instant | Required |
| Version | xmin | Concurrency token |

**Constraints and indexes**:

- Check `1 <= DurationWeeks AND DurationWeeks <= 52`.
- Check `0 <= ConfirmedCount AND ConfirmedCount <= Capacity`.
- Index `(TrainerId, Status, StartDate)`.
- Index `(Status, StartDate)` for student discovery.

### ProgramSchedule

One weekly meeting pattern within a finite program.

| Field | Type | Rules |
|-------|------|-------|
| Id | UUID | Primary key |
| ProgramId | UUID | Required parent |
| DayOfWeek | integer | ISO 1 through 7 |
| StartTime | LocalTime | Required |
| EndTime | LocalTime | Required; after StartTime |
| CreatedAt | Instant | Required |
| UpdatedAt | Instant | Required |

**Constraints**:

- Unique `(ProgramId, DayOfWeek, StartTime)`.
- Overnight schedules are out of scope.

**Serialized application invariant**: Schedules on the same weekday cannot
overlap. Program schedule mutations lock the parent Program row and validate all
sibling schedules in the same transaction.

### ProgramSession

A concrete session materialized during program publication.

| Field | Type | Rules |
|-------|------|-------|
| Id | UUID | Primary key |
| ProgramId | UUID | Required parent |
| ScheduleId | UUID | Required source schedule |
| WeekIndex | integer | Zero through DurationWeeks - 1 |
| LocalDate | LocalDate | Resolved trainer-local date |
| StartsAt | Instant | Required |
| EndsAt | Instant | Required, after StartsAt |
| Status | enum | `Scheduled`, `Cancelled`, `Completed` |
| CancelledAt | Instant? | Set on cancellation |
| CompletedAt | Instant? | Set after session |
| CreatedAt | Instant | Required |
| UpdatedAt | Instant | Required |
| Version | xmin | Concurrency token |

**Constraints and indexes**:

- Unique `(ScheduleId, WeekIndex)`.
- Unique `(ProgramId, StartsAt)`.
- Index `(ProgramId, StartsAt)`.
- Index `(Status, StartsAt)`.

For each `WeekIndex`, the applicable seven-day interval begins at
`StartDate + (7 * WeekIndex)`. The session uses the first matching weekday in
that interval, which guarantees one session per configured schedule per program
week.

### ProgramEnrollment

A student's reservation for the entire program.

| Field | Type | Rules |
|-------|------|-------|
| Id | UUID | Primary key |
| ProgramId | UUID | Required |
| StudentId | UUID | Required Student |
| Status | enum | `Confirmed`, `Withdrawn`, `Removed`, `Completed` |
| EnrolledAt | Instant | Required |
| WithdrawnAt | Instant? | Student withdrawal time |
| RemovedAt | Instant? | Trainer removal time |
| CompletedAt | Instant? | Set after program completion |
| ChangedByUserId | UUID? | Actor for withdrawal/removal |
| ChangeReason | string? | Maximum 500 characters |
| CreatedAt | Instant | Required |
| UpdatedAt | Instant | Required |
| Version | xmin | Concurrency token |

**Constraints and indexes**:

- Partial unique `(ProgramId, StudentId)` where `Status = 'Confirmed'`.
- Index `(StudentId, Status, ProgramId)`.
- Index `(ProgramId, Status)`.

## Derived Read Models

These are projections, not source-of-truth tables in v1.

### StudentOfferingCatalogItem

Joins an active `TrainerStudentAccess` grant to published series/programs and
their next available occurrence/session. It exposes trainer, offering type,
title, next date/time, time zone, and availability status.

### StudentScheduleItem

Unifies confirmed class bookings and program sessions for confirmed enrollments.
It exposes source type, source ID, title, trainer, concrete start/end, time zone,
location, and participation status.

### TrainerClientSummary

Unifies class booking and program enrollment history by trainer/student. It
exposes student identity, next participation, last participation, active
reservation count, and total historical participation. A student appears only
after at least one booking or enrollment with that trainer.

## State Transitions

### Recurring Class Series

```text
Draft -> Published -> Closed
  |          |          |
  +----------+----------+-> Cancelled
             |
             +-> Completed (only date-bounded series after final occurrence)
```

- Publication materializes the rolling horizon in the same operation.
- Closing stops new bookings and future horizon extension without deleting
  existing occurrences.
- Cancelling marks all future scheduled occurrences cancelled and retains
  bookings as historical statuses.

### Class Occurrence and Booking

```text
Occurrence: Scheduled -> Cancelled
                       -> Completed

Booking: Confirmed -> Cancelled (student)
                   -> Removed (trainer)
                   -> Completed (occurrence completed)
```

Only `Confirmed` bookings consume capacity.

### Program and Enrollment

```text
Program: Draft -> Published -> Completed
                         |
                         +-> Cancelled

Enrollment: Confirmed -> Withdrawn (student, before start)
                      -> Removed (trainer)
                      -> Completed (program completed)
```

Only `Confirmed` enrollments consume capacity. A program can remain Published
with `AcceptingEnrollments = false`.

## Transaction Boundaries

### Book a Class Occurrence

1. Verify Student role and active trainer access. If the student already has a
   Confirmed booking, return that booking without changing capacity.
2. Begin transaction.
3. Atomically increment occurrence `ConfirmedCount` only if scheduled,
   `StartsAt > @now`, the series accepts bookings, and capacity remains. Capture
   `@now` once from the trusted injected server clock; never accept it as request
   input.
4. Insert `ClassBooking(Confirmed)`.
5. Commit. If the partial unique constraint detects a concurrent duplicate, roll
   back the counter increment and return the existing booking. If the update
   affects zero rows, first re-read for an existing Confirmed booking and return
   it when found; only then classify the result as `CapacityUnavailable`,
   `OfferingUnavailable`, or `OfferingStarted`.

### Cancel or Remove a Class Booking

1. Resolve the booking and occurrence IDs with an actor-scoped read.
2. Begin transaction and lock the occurrence row first.
3. Conditionally transition the actor-owned booking from Confirmed.
4. If one booking changed, decrement occurrence count with a lower-bound guard.
5. Commit.

All capacity-changing transactions lock the capacity owner
(`ClassOccurrence`/`Program`) before a booking/enrollment row to prevent
cross-operation deadlocks.

### Enroll or Withdraw

Use the same duplicate-first pattern against the Program counter and
ProgramEnrollment: an existing Confirmed enrollment always redirects to that
enrollment, including when it consumes the final place. The conditional Program
update MUST include the authoritative deadline predicate, not an earlier
application-only check:

```sql
AND (
  SELECT MIN(starts_at)
  FROM program_sessions
  WHERE program_id = program.id
) > @now
```

`AcceptingEnrollments` is an additional trainer-controlled gate and display
hint; it never replaces the session-time predicate. Capture `@now` from the
trusted injected server clock in the same way as class booking.

### Publish an Offering

Validate the aggregate and generate concrete sessions using one captured clock
instant and the IANA time-zone database. Persist the status transition and all
materialized sessions in one transaction. Any collision or validation failure
rolls back publication.

## Data Retention and Privacy

- Operational history is retained according to the application's general
  retention policy; v1 does not implement user-selectable retention.
- UI and logs show only the minimum profile data needed for a roster or client
  history.
- Trainer queries always include owner scope; student queries always include
  the authenticated student or an active access grant.
- Credentials remain in Identity's protected tables and never appear in domain
  projections, logs, or telemetry.
