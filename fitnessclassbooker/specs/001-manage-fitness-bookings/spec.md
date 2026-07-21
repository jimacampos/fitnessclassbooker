# Feature Specification: Fitness Class and Program Booking

**Feature Branch**: `main`

**Created**: 2026-07-20

**Status**: Draft

**Input**: User description: "Build a web application for personal trainers to
manage classes, bookings, and clients. Trainers create capacity-limited recurring
classes or fixed-duration programs. Students log in to book one class occurrence
or enroll in an entire program. Payments are excluded from the first version."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Publish a Recurring Class (Priority: P1)

A trainer creates and publishes a group class that repeats on selected weekdays
at a local start and end time. The trainer may leave the series open indefinitely,
and sets the maximum participants for each occurrence.

**Why this priority**: A bookable schedule cannot exist until trainers can publish
their core recurring classes.

**Independent Test**: Create a weekly Tuesday Zumba class from 6:00 PM to 7:00 PM
with a capacity of 12 and no end date, then verify that its future Tuesday
occurrences are available to eligible students.

**Acceptance Scenarios**:

1. **Given** a trainer is authenticated, **When** they enter a title, start date,
   Tuesday schedule from 6:00 PM to 7:00 PM, time zone, and capacity of 12 and
   publish without an end date, **Then** the recurring class is active and future
   Tuesday occurrences are available for booking.
2. **Given** a trainer enters an end time that is not after the start time or a
   capacity below one, **When** they try to publish, **Then** the class remains
   unpublished and each invalid field has a clear correction message.
3. **Given** a trainer saves a valid class as a draft, **When** a student views
   available classes, **Then** the draft and its occurrences are not visible.

---

### User Story 2 - Book the Next Class (Priority: P1)

An authenticated student views a recurring class, receives its earliest upcoming
occurrence that still has space as the suggested option, and books that one
occurrence without payment or an ongoing commitment.

**Why this priority**: Booking the next fitness class is the student's primary
reason to use the application.

**Independent Test**: Using a published recurring class with multiple future
occurrences, book the suggested occurrence and verify that only that occurrence
appears in the student's upcoming bookings and trainer's roster.

**Acceptance Scenarios**:

1. **Given** a recurring class has multiple future occurrences with space,
   **When** an eligible student opens the class, **Then** the earliest occurrence
   that has not started and that the student has not already booked is suggested.
2. **Given** the suggested occurrence has space, **When** the student confirms the
   booking, **Then** one place is reserved only for that occurrence and both the
   student and trainer can see the confirmed booking.
3. **Given** the student already booked an occurrence, **When** they try to book
   it again, **Then** no duplicate booking is created and the existing booking is
   shown.
4. **Given** another student takes the final place first, **When** the student
   confirms using stale availability, **Then** no over-capacity booking is created
   and the student is prompted to view the next available occurrence.

---

### User Story 3 - Publish a Fixed-Duration Program (Priority: P2)

A trainer creates and publishes a program with a start date, a fixed number of
weeks, one or more weekly session schedules, and one participant limit shared by
the whole program.

**Why this priority**: Multi-session programs are a distinct trainer offering and
cannot be represented accurately as unrelated class bookings.

**Independent Test**: Create a four-week functional training program starting
August 1 with sessions Monday at 8:00 AM, Wednesday at 7:00 AM, and Friday at
8:00 AM, then verify that 12 dated sessions are shown before publication.

**Acceptance Scenarios**:

1. **Given** a trainer enters a start date, duration of four weeks, three valid
   weekly schedules, a time zone, and a capacity, **When** they review the
   program, **Then** every resulting session date and time is shown before
   publication.
2. **Given** a weekly session schedule would fall within each of four consecutive
   seven-day program weeks, **When** the schedule is generated, **Then** one
   dated session is created in each week for that schedule.
3. **Given** two configured sessions overlap or any session ends before it starts,
   **When** the trainer tries to publish, **Then** publication is blocked with a
   clear explanation.

---

### User Story 4 - Enroll in the Next Program (Priority: P2)

An authenticated student finds the next upcoming program and enrolls once for the
complete set of sessions. The enrollment reserves one program place and includes
every session; partial enrollment is not offered.

**Why this priority**: Whole-program enrollment delivers the intended cohort
experience and gives trainers a dependable participant roster.

**Independent Test**: Enroll a student in a future four-week program and verify
that the student appears once on the program roster and can view all 12 sessions.

**Acceptance Scenarios**:

1. **Given** a published program has not started and has capacity, **When** a
   student confirms enrollment, **Then** the student receives one confirmed
   enrollment covering all program sessions without a payment step.
2. **Given** a student is enrolled in a program, **When** they view their upcoming
   schedule, **Then** every remaining program session is shown and identified as
   part of that program.
3. **Given** a program is full or has started, **When** a student views it,
   **Then** enrollment is unavailable and the reason is clearly stated.
4. **Given** a student already enrolled, **When** they attempt to enroll again,
   **Then** no duplicate enrollment is created and their existing enrollment is
   shown.

---

### User Story 5 - Manage Offerings and Participants (Priority: P2)

A trainer reviews active, draft, completed, and cancelled offerings, sees capacity
and participant rosters, changes future availability, and removes a participant
when necessary. Existing participation records remain auditable.

**Why this priority**: Trainers need accurate schedules and rosters to run each
class safely after publishing it.

**Independent Test**: Open a trainer's recurring class and program, inspect their
rosters, cancel one future class occurrence, and remove one future participant
without changing past records or unrelated occurrences.

**Acceptance Scenarios**:

1. **Given** students have bookings or enrollments, **When** the trainer views an
   offering, **Then** the trainer sees its capacity, places remaining, participant
   names, and participation status.
2. **Given** a recurring series has future bookings, **When** the trainer edits its
   future schedule or capacity, **Then** the impact on booked occurrences is shown
   and confirmation is required before the change is applied.
3. **Given** a trainer cancels one future occurrence, **When** students or the
   trainer view schedules, **Then** the occurrence is marked cancelled, no new
   bookings are accepted, and its prior booking records remain visible.
4. **Given** a trainer removes a participant from a future class or program,
   **When** the removal is confirmed, **Then** the place becomes available and
   the participation record is marked removed rather than erased.
5. **Given** an offering belongs to another trainer, **When** a trainer attempts
   to view or change its management details, **Then** access is denied.

---

### User Story 6 - Review Clients and Personal Schedule (Priority: P3)

A trainer can find clients who have participated in their offerings and review
each client's upcoming and past participation. A student can review and cancel
their own future bookings or pre-start program enrollment.

**Why this priority**: Basic client history and self-service schedule management
reduce roster administration without expanding v1 into a full client relationship
management product.

**Independent Test**: Book and complete an occurrence for a student, then verify
that the trainer can find the student in the client list and that only the
student's own schedule is visible to the student.

**Acceptance Scenarios**:

1. **Given** a student has booked or enrolled with a trainer, **When** the trainer
   searches their clients, **Then** the student appears with upcoming and past
   participation for that trainer only.
2. **Given** a student has a future class booking, **When** they cancel it before
   the occurrence starts, **Then** the booking is marked cancelled and the place
   becomes available.
3. **Given** a program has not started, **When** an enrolled student withdraws,
   **Then** the whole enrollment is cancelled, all associated future sessions are
   removed from their upcoming schedule, and one program place becomes available.
4. **Given** a class occurrence or program has started, **When** a student tries
   to cancel or withdraw, **Then** self-service cancellation is unavailable and
   the student is directed to contact the trainer.

### Edge Cases

- A recurrence start date may not match one of its selected weekdays; the first
  bookable occurrence is the first selected weekday on or after the start date.
- Program session dates are calculated in consecutive seven-day periods beginning
  on the program start date; only configured sessions within the full program
  duration are included.
- Schedules preserve the trainer's local wall-clock time across daylight-saving
  changes and display the time zone to students.
- A recurrence with no end date continues until the trainer closes it; students
  need only be shown enough future occurrences to find and book the next
  available one.
- Reducing capacity below the number of confirmed participants is blocked.
- Simultaneous attempts for the last class or program place produce exactly one
  successful reservation.
- Cancelled, removed, completed, and past records remain distinguishable from
  active participation and cannot consume future capacity.
- If no upcoming occurrence has space, the application states that none is
  currently available; v1 does not create a waitlist.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The system MUST support authenticated trainer and student roles and
  restrict management actions to the trainer who owns the offering.
- **FR-002**: Trainers MUST be able to create, save as draft, publish, view, and
  close recurring class series.
- **FR-003**: A recurring class series MUST include a title, start date, one or
  more weekdays, start time, end time, time zone, and maximum participants per
  occurrence; description and location or joining instructions MAY be included.
- **FR-004**: A recurring class series MUST support either no end date or a
  trainer-selected end date.
- **FR-005**: The system MUST expose future occurrences from a published recurring
  class without requiring the trainer to create each occurrence manually.
- **FR-006**: Students MUST be able to view published offerings available to
  their account and their dates, times, trainer, capacity status, and relevant
  instructions.
- **FR-007**: For a recurring class, the system MUST suggest the earliest future
  occurrence that has capacity and is not already booked by the student.
- **FR-008**: A class booking MUST reserve one place for exactly one occurrence
  and MUST NOT enroll the student in later occurrences.
- **FR-009**: Trainers MUST be able to create, save as draft, publish, view, and
  close fixed-duration programs.
- **FR-010**: A program MUST include a title, start date, positive whole-number
  duration in weeks, one or more weekly session schedules, time zone, and maximum
  participants for the program; description and location or joining instructions
  MAY be included.
- **FR-011**: Before publication, the system MUST show the trainer every dated
  program session resulting from the start date, duration, and weekly schedules.
- **FR-012**: A program enrollment MUST reserve one program place for one student
  and MUST include all program sessions; partial enrollment is out of scope.
- **FR-013**: Students MUST be able to enroll only before the program starts and
  only while program capacity remains.
- **FR-014**: The system MUST prevent duplicate active bookings for the same
  student and class occurrence and duplicate active enrollments for the same
  student and program.
- **FR-015**: Capacity enforcement MUST be authoritative at confirmation time so
  concurrent requests cannot overbook a class occurrence or program.
- **FR-016**: Booking and enrollment confirmation MUST NOT request or record
  payment.
- **FR-017**: Trainers MUST be able to view capacity, remaining places, and the
  roster and status for each class occurrence and program they own.
- **FR-018**: Trainers MUST be able to edit future offering details, cancel a
  future class occurrence, close future bookings or enrollments, and remove a
  participant while retaining historical records.
- **FR-019**: Before a trainer applies a change that affects confirmed
  participants, the system MUST summarize the affected occurrences or program
  sessions and require explicit confirmation.
- **FR-020**: Students MUST be able to view their own upcoming and past class
  bookings, program enrollments, and program sessions.
- **FR-021**: Students MUST be able to cancel a future class booking and withdraw
  from an entire program before it starts; released places MUST immediately
  become available.
- **FR-022**: Trainers MUST be able to search clients who have booked or enrolled
  with them and review those clients' participation history for their own
  offerings.
- **FR-023**: The system MUST preserve the trainer's configured local start and
  end times across daylight-saving changes and display the applicable time zone.
- **FR-024**: The system MUST retain the status of confirmed, cancelled, removed,
  and completed participation instead of deleting operational history.
- **FR-025**: The first version MUST exclude payments, waitlists, attendance
  tracking, package or membership management, automated messaging, and a public
  trainer marketplace.

### User Experience Requirements *(mandatory for user-facing changes)*

- **UX-001**: Trainer and student schedules MUST use consistent class, program,
  session, capacity, booking, and enrollment terminology throughout the product.
- **UX-002**: Creation and editing flows MUST show field-level validation without
  discarding valid input and MUST provide a review of recurrence or program dates
  before publication.
- **UX-003**: Offering lists, schedules, booking controls, rosters, and client
  views MUST define loading, empty, success, validation, full-capacity, cancelled,
  and failure states.
- **UX-004**: Booking and enrollment confirmation MUST clearly state whether the
  student is reserving one class occurrence or every session in a program.
- **UX-005**: Capacity races and expired availability MUST preserve the student's
  context and provide a direct path to the next available option.
- **UX-006**: Primary trainer and student journeys MUST support keyboard
  operation, visible focus, semantic headings and labels, sufficient contrast,
  and responsive use at WCAG 2.1 AA expectations.
- **UX-007**: Destructive or participation-affecting actions MUST identify the
  affected class, occurrence, program, or student and require explicit
  confirmation.

### Performance Requirements *(mandatory when performance can be affected)*

- **PR-001**: User actions MUST show visible feedback within 100 milliseconds
  under the reference workload.
- **PR-002**: Primary trainer and student pages MUST show useful schedule or
  offering content within 2.5 seconds for at least 75% of visits under the
  reference workload.
- **PR-003**: Booking, cancellation, enrollment, withdrawal, publication, and
  roster changes MUST complete and show a definitive result within 2 seconds for
  at least 95% of attempts under the reference workload.
- **PR-004**: The booking and enrollment capacity rules MUST remain correct with
  100 concurrent active users, including simultaneous attempts for the final
  available place.
- **PR-005**: Performance MUST be evaluated with a repeatable representative
  workload containing 100 offerings per trainer, 1,000 clients per trainer,
  10,000 upcoming sessions across the application, and 100 concurrent active
  users.

### Key Entities

- **User**: An authenticated person with a trainer or student role and a local
  time zone used to display schedules.
- **Trainer**: A user who owns offerings and can manage only their schedules,
  rosters, and participating clients.
- **Student**: A user who can discover available offerings, reserve places, and
  manage their own participation.
- **Recurring Class Series**: An open-ended or date-bounded offering with weekly
  recurrence rules and a per-occurrence participant limit.
- **Class Occurrence**: One dated instance of a recurring class, with its own
  schedule, status, capacity, and roster.
- **Program**: A fixed-duration offering with a shared participant limit and a
  defined collection of dated sessions.
- **Program Session**: One dated meeting generated from a program's weekly
  schedules and duration.
- **Class Booking**: A student's status-bearing reservation for one class
  occurrence.
- **Program Enrollment**: A student's status-bearing reservation for every
  session in one program.
- **Client Relationship**: The trainer-specific view of a student established by
  that student's booking or enrollment history with the trainer.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: At least 90% of trainers can publish a valid recurring class with
  capacity in under three minutes on their first attempt.
- **SC-002**: At least 90% of trainers can configure and verify a four-week,
  three-session-per-week program in under five minutes on their first attempt.
- **SC-003**: At least 95% of students can find and confirm the next available
  class occurrence in under two minutes without assistance.
- **SC-004**: At least 95% of students can distinguish single-class booking from
  whole-program enrollment before confirming.
- **SC-005**: No confirmed class occurrence or program exceeds its participant
  limit, including when multiple students attempt to take the final place at the
  same time.
- **SC-006**: Trainers can find any participant in an offering roster and identify
  their status in under 30 seconds.
- **SC-007**: All primary creation, booking, enrollment, roster, and schedule
  actions are operable by keyboard with visible focus and meet WCAG 2.1 AA
  expectations.
- **SC-008**: Under the documented reference workload, at least 75% of primary
  pages show useful content within 2.5 seconds and at least 95% of transactional
  actions show a definitive result within 2 seconds.

## Assumptions

- Trainer and student accounts, authentication, and account recovery are provided
  as shared application capabilities; account registration is not defined by this
  feature.
- Students see offerings from trainers already available to their account.
  Public trainer discovery and client invitation workflows are outside v1.
- One user has one role in v1; trainer accounts do not book as students.
- All schedules are defined in the trainer's selected time zone and displayed
  with enough time-zone context for students to understand the local time.
- Program enrollment closes when the program starts. Late enrollment and partial
  enrollment are outside v1.
- Cancellation has no fee or penalty because v1 has no payment handling.
- Client management in v1 is a searchable participation directory, not a full
  customer relationship management system; private health notes, messaging, and
  marketing features are excluded.
- Historical participation is retained according to the application's general
  data-retention and privacy policy.
