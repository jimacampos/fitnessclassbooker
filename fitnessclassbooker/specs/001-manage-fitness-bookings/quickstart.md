# Quickstart: Validate Fitness Class and Program Booking

This guide describes the runnable validation flow the implementation must
provide. It proves local/cloud parity and the feature's critical trainer,
student, capacity, accessibility, and performance outcomes.

## Prerequisites

- .NET 10 SDK
- Docker Engine with Docker Compose
- A POSIX shell for repository scripts
- Chromium installed by the end-to-end test setup script
- k6 for the optional local load test
- Azure CLI and Azure Developer CLI (`azd`) only for Azure deployment

No Azure account or cloud service is required for local development or the
automated test suite.

## Start the Local Application

From the repository root:

```bash
cp .env.example .env
./scripts/dev-up.sh
```

`dev-up.sh` must:

1. start PostgreSQL 18 and the web application with Docker Compose;
2. wait until `/health/ready` succeeds;
3. verify through readiness that startup migrations completed under the
   advisory lock and PostgreSQL is reachable;
4. verify that development-only trainer/student accounts and access grants were
   created;
5. print the local URL and seed account emails without printing passwords.

Set development passwords in `.env`; that file is ignored by Git. The checked-in
`.env.example` contains names and safe placeholders only.

Verify readiness:

```bash
curl --fail http://localhost:8080/health/live
curl --fail http://localhost:8080/health/ready
```

Expected result: both commands return HTTP 200. The application is available at
`http://localhost:8080`.

Stop and remove local containers:

```bash
docker compose down
```

Add `--volumes` only when intentionally resetting local development data.

## Run Automated Quality Gates

### Build, static checks, unit, integration, and contract tests

```bash
./scripts/test.sh
```

The script must restore locked dependencies, verify formatting/analyzers, build
with warnings as errors, report vulnerable dependencies, and run:

- unit tests for schedule generation, DST, validation, state transitions, and
  edit-impact calculation;
- integration tests against an isolated Testcontainer pinned to `postgres:18`;
- concurrent final-place and booking-versus-cancellation tests;
- route/authorization/anti-forgery contracts from
  [contracts/web-routes.md](contracts/web-routes.md).

Expected result: no failures and no database or container state shared across
test runs.

### Critical browser journeys and accessibility

```bash
./scripts/test-e2e.sh
```

The script must install the pinned Chromium version if missing, start an isolated
application/database, run Playwright journeys, and run axe-core on each critical
page/state.

Expected result: publish/book/enroll/manage/cancel journeys pass at desktop and
320 CSS pixel viewports, keyboard-only navigation passes, and there are no
automatically detectable WCAG 2.1 AA violations.

### Reference performance and concurrency profile

```bash
./scripts/test-load.sh
```

The script must seed the reference workload from the specification and run k6
with 100 concurrent users.

Expected thresholds:

- zero class or program overbookings;
- all accepted reservations are unique;
- transactional actions complete within 2 seconds at p95;
- primary pages expose useful content within 2.5 seconds at p75;
- error rate remains below 1%, excluding expected full-capacity conflicts.

## Manual Acceptance Walkthrough

Use the development accounts printed by `dev-up.sh`.

### 1. Publish an indefinite Zumba class

1. Sign in as the trainer.
2. Create a recurring class named **Zumba**.
3. Select Tuesday, 6:00 PM to 7:00 PM, the trainer's time zone, capacity 2,
   and no end date.
4. Save the draft, review generated dates, and publish it.

Expected:

- the draft is invisible to students before publication;
- publication shows 12 weeks of future Tuesday occurrences;
- each occurrence displays the trainer's time zone and two places;
- the trainer can open an occurrence roster.

### 2. Book one class occurrence

1. Sign in as a student with access to the trainer.
2. Open Zumba.
3. Confirm the suggested next available occurrence.
4. Open **My schedule**.

Expected:

- confirmation explicitly states that only one occurrence is booked;
- one occurrence appears in the student's schedule;
- the trainer roster shows the student;
- later occurrences remain unbooked.

### 3. Prove final-place protection

1. Leave one place available on an occurrence.
2. Submit booking attempts for that occurrence as two different students at the
   same time, or run the focused concurrency test from `test-load.sh`.

Expected:

- exactly one new booking is confirmed;
- confirmed count equals capacity;
- the losing student sees a full-capacity message and the next available
  occurrence;
- retrying either request creates no duplicate.

### 4. Publish and enroll in a four-week program

1. Sign in as the trainer and create **Functional Training**.
2. Set a four-week duration and start date.
3. Add Monday 8:00 AM, Wednesday 7:00 AM, and Friday 8:00 AM schedules, each
   with a valid end time.
4. Preview and publish the program.
5. Sign in as a student and enroll.

Expected:

- preview lists exactly 12 ordered sessions before publication;
- enrollment confirmation states that all sessions are included and no payment
  occurs;
- one program place is consumed;
- all 12 sessions appear in the student's schedule.

### 5. Manage future participation

1. As trainer, preview a recurring schedule/capacity change.
2. Confirm that impacted occurrences and students are listed before applying it.
3. Cancel one future occurrence and remove one participant.
4. As a student, cancel another future class and withdraw from a program that has
   not started.

Expected:

- stale edit previews are rejected rather than applied;
- cancelled/removed records remain visible with status;
- each released place becomes available immediately;
- self-service changes are blocked after the relevant start time.

### 6. Review client isolation

1. As trainer, search clients and open a participating student's history.
2. Attempt the same route using a different trainer account.
3. As a student, attempt another student's schedule route.

Expected:

- only participation owned by the trainer appears;
- cross-trainer and cross-student access is denied without leaking data.

## Time-Zone Validation

Run the focused schedule tests for at least:

- a normal weekly transition;
- a spring-forward skipped local time;
- a fall-back ambiguous local time;
- an occurrence generated when the series start date is not its selected
  weekday;
- a series spanning a DST change while retaining the trainer's wall-clock time.

The preview and saved occurrence must resolve to the same instants using the
rules in [data-model.md](data-model.md).

## Deploy a Non-production Azure Environment

Authenticate and create an isolated `azd` environment:

```bash
az login
azd auth login
azd env new dev
azd up
```

The Bicep deployment must provision:

- Linux App Service with Always On, system-assigned managed identity, readiness
  health check, HTTPS-only cookies, and configurable SKU;
- PostgreSQL 18 Flexible Server with backups and private connectivity;
- Key Vault with the database secret referenced from App Service settings;
- Application Insights and Log Analytics;
- VNet integration and private DNS required for database connectivity.

Secure Bicep parameters are prompted or supplied by an approved secret store;
they are never written to source control or command output.

After deployment, obtain the application URL from `azd show` and validate:

```bash
curl --fail https://<application-host>/health/live
curl --fail https://<application-host>/health/ready
```

Run smoke-tagged browser tests against the Azure URL, then run the reference k6
profile against this non-production environment before release.

## Azure Operational Acceptance

Before production release, verify:

- readiness removes an unhealthy instance from rotation;
- failed migrations keep readiness unhealthy and surface an alert;
- booking/enrollment traces include correlation IDs and durations but no
  credentials or sensitive profile fields;
- alerts exist for readiness failures, elevated server errors, p95 transaction
  budget breaches, database saturation, and occurrence horizon lag;
- a PostgreSQL point-in-time restore is rehearsed into a non-production server;
- changing the Key Vault database secret and restarting App Service resolves the
  new reference successfully.

## Reset

For a clean local reset:

```bash
docker compose down --volumes
./scripts/dev-up.sh
```

For Azure, delete the disposable environment only after preserving any required
diagnostic evidence:

```bash
azd down --purge
```
