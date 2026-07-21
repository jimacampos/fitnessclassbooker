# Research: Fitness Class and Program Booking

## Decision 1: Use a server-rendered .NET modular monolith

**Decision**: Use C# 14/.NET 10 with ASP.NET Core Razor Pages in one deployable
application, organized by Classes, Programs, Clients, Schedule, Identity, and
shared infrastructure.

**Rationale**: The product is form-, schedule-, and list-oriented. Razor Pages
supports accessible HTML, server validation, anti-forgery, cookie authentication,
and progressive enhancement without a separate frontend build/runtime. A modular
monolith gives clear domain boundaries while keeping transactions and deployment
simple.

**Alternatives considered**:

- **React/Vue SPA plus API**: Rejected for v1 because it doubles application and
  testing surfaces without a requirement for rich client-state behavior.
- **Microservices**: Rejected because all modules share transactional booking
  data and do not need independent scaling.
- **Azure Container Apps**: Rejected because App Service is a more direct fit for
  one continuously available web application; revisions, scale-to-zero, and
  container orchestration add no v1 value.

**Sources**:

- [Deploy ASP.NET Core apps to Azure App Service](https://learn.microsoft.com/aspnet/core/host-and-deploy/azure-apps/?view=aspnetcore-10.0)
- [Common web application architectures](https://learn.microsoft.com/dotnet/architecture/modern-web-apps-azure/common-web-application-architectures)

## Decision 2: Use PostgreSQL 18 in every environment

**Decision**: Use PostgreSQL 18 through Npgsql/EF Core 10, running with the
explicit `postgres:18` image locally and in Testcontainers, and Azure Database
for PostgreSQL Flexible Server in Azure.

**Rationale**: Using the same engine and major version catches SQL, constraint,
index, transaction, and time-zone behavior locally. PostgreSQL supplies partial
unique indexes, check constraints, atomic conditional updates, advisory locks,
and `timestamptz`, which directly support booking correctness.

**Alternatives considered**:

- **SQLite locally**: Rejected because its concurrency, types, migrations, and
  constraints differ on the highest-risk paths.
- **Azure SQL**: Viable, but PostgreSQL offers equally managed Azure hosting and
  simpler local parity through a standard container.
- **In-memory EF provider for integration tests**: Rejected because it does not
  validate relational behavior. It may be used only for isolated UI tests that
  do not assert persistence semantics.

**Sources**:

- [Supported PostgreSQL versions in Azure](https://learn.microsoft.com/azure/postgresql/configure-maintain/concepts-supported-versions)
- [.NET connectivity to PostgreSQL Flexible Server](https://learn.microsoft.com/azure/postgresql/connectivity/connect-csharp)

## Decision 3: Use ASP.NET Core Identity with application cookies

**Decision**: Store local accounts in ASP.NET Core Identity and use secure,
HTTP-only, same-site cookies. Authorize Trainer/Student roles at the page level
and enforce ownership/resource checks in application services.

**Rationale**: It works identically locally and in Azure, needs no external
identity tenant for tests, and provides password hashing, lockout, CSRF
integration, and cookie lifecycle behavior. The feature assumes account
provisioning exists, so v1 only requires login/logout and development seeds.

**Alternatives considered**:

- **Microsoft Entra External ID**: A strong future option for managed identity
  journeys, but it introduces an Azure tenant dependency into local tests.
- **JWT tokens**: Rejected because a same-origin server-rendered application does
  not need browser-managed bearer tokens.

## Decision 4: Model weekly schedules explicitly and materialize occurrences

**Decision**: Store weekly schedule rows with local date/time and an IANA time
zone. Noda Time converts each concrete session to UTC. Persist all program
sessions on publication and keep a rolling 12-week materialized horizon for
recurring classes.

**Rationale**: The specification needs weekly schedules only, so explicit rows
are clearer and easier to validate than general iCalendar rules. Persisted
occurrences give stable IDs for bookings, cancellation, capacity snapshots,
rosters, and audit history. A finite horizon prevents unbounded storage while
making student queries simple and indexed.

**Alternatives considered**:

- **Store RFC 5545 RRULE and expand every query**: Rejected for v1 because the
  supported recurrence language is narrower and bookings still need stable
  occurrence records.
- **Materialize an indefinite series forever**: Impossible and unnecessary.
- **Generate an occurrence only when booked**: Rejected because listing,
  cancellation, availability, and simultaneous booking become harder to reason
  about.

**Sources**:

- [Noda Time concepts](https://nodatime.org/3.2.x/userguide/concepts)
- [PostgreSQL date/time types](https://www.postgresql.org/docs/current/datatype-datetime.html)

## Decision 5: Enforce capacity with atomic database updates

**Decision**: Keep a confirmed counter on each class occurrence and program.
Within one transaction, conditionally increment the counter and insert the
booking/enrollment. Use partial unique indexes for one active reservation per
student and check constraints for counter bounds.

**Rationale**: A statement such as `UPDATE ... SET confirmed_count =
confirmed_count + 1 WHERE confirmed_count < capacity RETURNING ...` serializes
competing updates on one row in PostgreSQL. Exactly one request wins the final
place. The conditional update also checks booking/enrollment state and concrete
start deadlines against a trusted server-clock parameter. Rolling back the
transaction also rolls back the increment.

**Alternatives considered**:

- **Count bookings before insert**: Rejected because two transactions can observe
  the same available place.
- **Application/distributed lock**: Rejected because correctness belongs in the
  system of record and a distributed lock adds infrastructure.
- **Serializable transactions around every booking**: Correct but broader and
  more failure-prone than the focused atomic update required here.

**Source**:

- [PostgreSQL UPDATE](https://www.postgresql.org/docs/current/sql-update.html)
- [PostgreSQL explicit locking](https://www.postgresql.org/docs/current/explicit-locking.html)

## Decision 6: Deploy to Azure App Service with managed platform services

**Decision**: Deploy the framework-dependent application to Linux Azure App
Service. Use PostgreSQL Flexible Server, Key Vault, Application Insights/Log
Analytics, system-assigned managed identity, App Service VNet integration, and
private database networking. Define all resources in Bicep and orchestrate
environments with `azd`.

**Rationale**: App Service provides managed TLS, health checks, patching, scale
settings, and straightforward .NET deployment. Bicep makes environments
repeatable; `azd` keeps environment configuration and deployment commands
consistent. Key Vault references expose secrets as normal app settings without
secret-fetching code.

**Alternatives considered**:

- **Virtual machines**: Rejected due to operating-system and web-server
  maintenance.
- **AKS**: Rejected because there is one service and no Kubernetes requirement.
- **Connection strings committed in configuration**: Prohibited.

**Sources**:

- [App Service Key Vault references](https://learn.microsoft.com/azure/app-service/app-service-key-vault-references)
- [Work with Azure Developer CLI environments](https://learn.microsoft.com/azure/developer/azure-developer-cli/work-with-environments)
- [App Service private endpoints and networking](https://learn.microsoft.com/azure/app-service/overview-private-endpoint)

## Decision 7: Use OpenTelemetry with Azure Monitor

**Decision**: Instrument ASP.NET Core, Npgsql/EF Core, and the domain's booking
operations with OpenTelemetry. Export to Azure Monitor/Application Insights only
when its connection string is configured; retain console telemetry locally.

**Rationale**: Instrumentation remains vendor-neutral and local development does
not depend on Azure. Azure's supported distribution captures traces, metrics,
logs, and exceptions with minimal platform-specific code.

**Alternative considered**: The classic Application Insights SDK was rejected
in favor of Microsoft's current OpenTelemetry guidance.

**Source**:

- [Enable Azure Monitor OpenTelemetry](https://learn.microsoft.com/azure/azure-monitor/app/opentelemetry-enable)

## Decision 8: Test real boundaries, not framework substitutes

**Decision**:

- Unit-test schedule expansion, DST resolution, validation, state transitions,
  impact calculation, and capacity command outcomes.
- Use `WebApplicationFactory` and Testcontainers PostgreSQL for migrations,
  persistence, authorization, ownership, partial indexes, and concurrent final
  place tests.
- Treat the documented route/action table as a contract and test status,
  redirects, validation, anti-forgery, and role behavior.
- Use Playwright for the critical publish/book/enroll/manage/cancel journeys and
  integrate axe-core checks.
- Use k6 for the 100-user reference workload and last-place race.

**Rationale**: Fast pure tests cover permutations while focused real-database
tests prove the semantics that protect bookings. A small browser suite protects
the highest-value UX without making all feedback dependent on E2E tests.

**Alternatives considered**:

- **All browser tests**: Rejected as slow and brittle.
- **Mocked database integration tests**: Rejected because they cannot prove
  transactional capacity behavior.
- **A fixed coverage percentage as the only gate**: Rejected; requirement and
  risk coverage are more meaningful. Coverage is reported to identify gaps.

**Source**:

- [Integration tests in ASP.NET Core](https://learn.microsoft.com/aspnet/core/test/integration-tests?view=aspnetcore-10.0)

## Decision 9: Keep v1 operationally synchronous

**Decision**: Perform booking, enrollment, cancellation, publication, and
occurrence generation synchronously in the web process. Use an idempotent daily
hosted service only to extend recurring class horizons and complete elapsed
records.

**Rationale**: Users require a definitive result and v1 excludes notifications
and other asynchronous workflows. A broker, outbox, and worker would add
failure modes without user value.

**Alternatives considered**:

- **Service Bus plus worker**: Defer until notifications or long-running work
  become requirements.
- **Redis cache**: Defer until measured page latency exceeds its budget after
  query/index tuning.
