# Implementation Plan: Fitness Class and Program Booking

**Branch**: `main` | **Date**: 2026-07-20 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from
`/specs/001-manage-fitness-bookings/spec.md`

## Summary

Build the initial product as one server-rendered ASP.NET Core modular monolith.
Trainers manage recurring classes, programs, rosters, and participating clients;
students book one occurrence or enroll in a complete program. PostgreSQL is the
system of record and enforces capacity atomically.

The same .NET application artifact and PostgreSQL major version run locally and
in Azure. Local development uses Docker Compose; Azure uses Linux App Service,
Azure Database for PostgreSQL Flexible Server, Key Vault references, Application
Insights through OpenTelemetry, and Bicep orchestrated by Azure Developer CLI.
This avoids a separate JavaScript application and microservice infrastructure
until the product demonstrates a need for either.

## Technical Context

**Language/Version**: C# 14 on .NET 10 LTS

**Primary Dependencies**: ASP.NET Core 10 Razor Pages, ASP.NET Core Identity,
Entity Framework Core 10, Npgsql EF Core provider, Noda Time, Bootstrap 5
(bundled), Azure Monitor OpenTelemetry

**Storage**: PostgreSQL 18 locally and Azure Database for PostgreSQL Flexible
Server; EF Core migrations; no cache or message broker in v1

**Testing**: xUnit, ASP.NET Core `WebApplicationFactory`, Testcontainers for
PostgreSQL pinned to the `postgres:18` image, Playwright for critical browser
journeys, axe-core integration for accessibility, and k6 for concurrency and
latency

**Target Platform**: Modern evergreen desktop/mobile browsers; Linux Azure App
Service in Azure; macOS or Linux with .NET SDK and Docker for local development

**Project Type**: Server-rendered web application in a modular monolith

**Performance Goals**: Visible feedback within 100 ms; useful primary content
within 2.5 seconds at p75; booking, enrollment, cancellation, publication, and
roster changes within 2 seconds at p95; zero overbookings with 100 concurrent
users

**Constraints**: WCAG 2.1 AA; no payments, waitlists, attendance, messaging, or
public marketplace; preserve trainer-local wall-clock schedules across DST;
one deployable application; deterministic local tests with no required Azure
connection

**Scale/Scope**: Initial reference load of 100 offerings and 1,000 participating
clients per trainer, 10,000 upcoming sessions application-wide, and 100
concurrent active users

## Constitution Check

*GATE: Passed before research and re-checked after Phase 1 design.*

- **Code quality — PASS**: One deployable application, one database, explicit
  feature modules, and framework conventions are the simplest design that
  supports the two booking models. No generic repository, service bus,
  microservices, SPA toolchain, or distributed cache is introduced. Build
  warnings, formatting, nullable reference types, analyzers, and dependency
  vulnerability checks are required in CI.
- **Testing — PASS**: Pure scheduling and state rules receive unit tests;
  PostgreSQL constraints, migrations, authorization, routes, and concurrency
  receive integration/contract tests against a real containerized PostgreSQL
  instance; Playwright covers the critical trainer and student journeys.
  k6 proves the capacity and latency budgets. Regression tests must reproduce a
  defect before its fix.
- **User experience — PASS**: Razor Pages share one layout, bundled design
  tokens/components, vocabulary, field validation, status badges, confirmation
  summaries, and loading/empty/success/full/cancelled/failure patterns. Critical
  pages are keyboard tested, scanned with axe-core, and manually checked at
  narrow and wide viewports.
- **Performance — PASS**: The spec budgets are retained. Queries are paginated
  and indexed; occurrences and program sessions are materialized; roster data
  is loaded only on management pages; booking capacity is changed with one
  atomic database statement. k6 runs the documented 100-user reference profile
  locally and against a non-production Azure environment.

### Post-Design Re-check

The data model, route contracts, and deployment design add no constitutional
exceptions. Capacity counters are deliberate denormalization protected by the
same database transaction and check constraints; they avoid expensive locking
or count queries on the most time-sensitive path. All cloud dependencies have
local equivalents or are optional telemetry/configuration adapters.

## Architecture

```text
Browser
  |
  v
ASP.NET Core Razor Pages
  |-- Identity (cookie authentication, Trainer/Student roles)
  |-- Classes module
  |-- Programs module
  |-- Clients module (read projections)
  |-- Shared UI, time, authorization, and observability
  |
  v
EF Core + focused PostgreSQL commands
  |
  v
PostgreSQL 18
```

The modules share one process and database but own their page handlers,
application services, domain types, persistence mappings, and tests. Cross-module
reads use explicit query services rather than direct UI access to another
module's tables. Domain behavior stays in aggregates and focused application
services; Razor Page models coordinate requests but do not contain booking or
scheduling rules.

### Scheduling and Time

- Store a trainer's time zone as an IANA identifier, local dates as PostgreSQL
  `date`, local schedule times as `time`, and concrete session instants as UTC
  `timestamptz`.
- Use Noda Time to map local schedules through DST. For an ambiguous local time,
  choose the earlier offset consistently; for a skipped local time, move to the
  first valid instant after the gap and show the resolved time during preview.
- Store explicit weekly schedule rows instead of a general RRULE parser because
  v1 only supports weekly patterns.
- Materialize all program sessions when a program is previewed and persist them
  atomically when published.
- Materialize class occurrences for a rolling 12-week horizon at publication
  and extend the horizon daily with an idempotent hosted service. Application
  startup also repairs a missing horizon. A unique `(series_id, starts_at_utc)`
  constraint makes retries and multiple app instances safe.

### Capacity and Concurrency

`ClassOccurrence` and `Program` hold a confirmed participant counter alongside
capacity. Booking/enrollment uses a database transaction:

1. Atomically increment the counter only when status allows reservations,
   start time has not passed, and `confirmed_count < capacity`.
2. Insert the confirmed booking or enrollment with a filtered unique constraint
   preventing a second active reservation for the same student.
3. Commit both changes together; any duplicate or insert failure rolls back the
   increment.

Cancellation, withdrawal, and trainer removal update participation status and
decrement the counter in the same transaction. PostgreSQL check constraints
enforce `0 <= confirmed_count <= capacity`. This design requires no distributed
lock and returns a deterministic conflict when the final place is lost.

The conditional update also enforces lifecycle deadlines from persisted
instants, including `starts_at > @now` for a class and the program's earliest
session being in the future. `@now` is captured once from the server's injected
clock and is never accepted from the browser. Display flags never replace these
authoritative guards. If the update affects no rows, the application re-reads
current state to distinguish full, closed/cancelled, and started outcomes for
the user.

### Authentication and Authorization

ASP.NET Core Identity provides local email/password accounts and secure cookies.
Trainer and Student roles are mutually exclusive in v1. Page authorization
conventions provide the first gate; ownership/student checks in application
services provide the resource-level gate. Development-only seeds create one
trainer and multiple students for local validation. Registration, password
recovery delivery, and trainer-client invitation remain shared capabilities
outside this feature.

A `TrainerStudentAccess` grant represents the pre-existing relationship that
makes a trainer's offerings visible to a student. v1 reads these grants but has
no UI to create them; development seeds provide them. The trainer's client
directory itself remains a projection of actual booking/enrollment history.

### Azure and Local Environments

| Concern | Local | Azure |
|---------|-------|-------|
| Web host | `dotnet run` or Docker Compose | Linux Azure App Service |
| Database | PostgreSQL 18 container | PostgreSQL 18 Flexible Server |
| Secrets | `.env`/user-secrets, never committed | App settings with Key Vault references |
| Identity | ASP.NET Core Identity | Same application and Identity store |
| Telemetry | Console plus optional OTLP endpoint | Azure Monitor OpenTelemetry/Application Insights |
| Provisioning | Docker Compose | Bicep through `azd` |

Azure infrastructure includes App Service with Always On and health checks,
system-assigned managed identity, Key Vault, Application Insights/Log Analytics,
PostgreSQL Flexible Server, and private networking between App Service and the
database. Bicep parameters select low-cost development SKUs or production SKUs
without changing the application.

Database migrations run before request handling and acquire a PostgreSQL
advisory lock so only one instance applies migrations. A failed migration fails
startup and readiness rather than serving against a partial schema. Move
migrations to a dedicated deployment job if growth makes startup migrations too
slow.

### Observability and Operations

- Structured logs include correlation ID, actor ID, trainer ID, offering ID,
  operation, result, and duration but never credentials or sensitive profile
  data.
- OpenTelemetry records HTTP, EF Core/Npgsql, booking, enrollment, recurrence
  materialization, and exception telemetry. Azure exports to Application
  Insights; local development works with console output alone.
- `/health/live` verifies the process, while `/health/ready` verifies database
  connectivity and migration state. Azure Health Check uses readiness.
- Metrics include booking/enrollment attempts and conflicts, full-capacity
  rejections, transaction duration, occurrence horizon lag, page/handler
  duration, and database pool saturation.

## Delivery Sequence

1. **Foundation**: solution structure, Razor shell/design tokens, Identity roles
   and development seeds, PostgreSQL/EF migrations, Compose, health checks,
   telemetry, Bicep/azd, and CI quality gates.
2. **Recurring classes and booking (P1)**: class creation/preview/publication,
   occurrence materialization, student discovery, next-occurrence suggestion,
   atomic booking, schedules, and rosters.
3. **Programs and enrollment (P2)**: program schedules/session preview,
   publication, atomic whole-program enrollment, schedules, and rosters.
4. **Management (P2)**: future edits and impact summaries, occurrence/program
   cancellation, booking closure, participant removal, and audit statuses.
5. **Clients and self-service (P3)**: trainer client projection/history plus
   student cancellation and program withdrawal.
6. **Release validation**: end-to-end accessibility, concurrent final-place load,
   reference-data performance, Azure smoke tests, backup/restore check, and
   operational dashboards/alerts.

## Project Structure

### Documentation (this feature)

```text
specs/001-manage-fitness-bookings/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/
│   └── web-routes.md
└── tasks.md
```

### Source Code (repository root)

```text
FitnessClassBooker.sln
Directory.Build.props
Directory.Packages.props
azure.yaml
compose.yaml
Dockerfile

src/
└── FitnessClassBooker.Web/
    ├── Features/
    │   ├── Classes/
    │   │   ├── Domain/
    │   │   ├── Application/
    │   │   ├── Infrastructure/
    │   │   └── Pages/
    │   ├── Programs/
    │   │   ├── Domain/
    │   │   ├── Application/
    │   │   ├── Infrastructure/
    │   │   └── Pages/
    │   ├── Clients/
    │   │   ├── Application/
    │   │   └── Pages/
    │   └── Schedule/
    │       ├── Application/
    │       └── Pages/
    ├── Identity/
    ├── Infrastructure/
    │   ├── Data/
    │   ├── Observability/
    │   └── Time/
    ├── Pages/
    │   ├── Account/
    │   ├── Shared/
    │   └── Index.cshtml
    ├── wwwroot/
    │   ├── css/
    │   └── js/
    └── Program.cs

tests/
├── FitnessClassBooker.UnitTests/
├── FitnessClassBooker.IntegrationTests/
├── FitnessClassBooker.ContractTests/
├── FitnessClassBooker.EndToEndTests/
└── performance/
    ├── booking.js
    └── browsing.js

infra/
├── main.bicep
├── main.parameters.json
└── modules/
    ├── app-service.bicep
    ├── database.bicep
    ├── key-vault.bicep
    ├── monitoring.bicep
    └── networking.bicep

scripts/
├── dev-up.sh
├── test.sh
├── test-e2e.sh
└── test-load.sh
```

**Structure Decision**: Use a single executable Razor Pages project organized
into vertical feature modules. Separate test projects isolate fast unit tests,
real-database integration tests, route contracts, and browser journeys without
creating production-layer projects that add no independent deployment boundary.
Infrastructure is declarative and kept beside the application.

## Complexity Tracking

No constitutional violations require exceptions.
