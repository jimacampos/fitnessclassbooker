<!--
Sync Impact Report
- Version change: template -> 1.0.0
- Added principles:
  - I. Maintainable Code by Default
  - II. Tests Define Done
  - III. Consistent, Accessible User Experience
  - IV. Measurable Performance
- Added sections:
  - Quality and Delivery Standards
  - Development Workflow and Quality Gates
- Removed sections: none
- Templates:
  - updated: .specify/templates/plan-template.md
  - updated: .specify/templates/spec-template.md
  - updated: .specify/templates/tasks-template.md
  - no change required: .specify/templates/checklist-template.md
  - no command templates present under .specify/templates/commands/
- Deferred items: none
-->
# Fitness Class Booker Constitution

## Core Principles

### I. Maintainable Code by Default

All production code MUST be clear, cohesive, and no more complex than the behavior
requires. Changes MUST follow the repository's established structure, naming, typing,
formatting, linting, and error-handling conventions. Shared behavior MUST be reused or
extracted rather than duplicated. Public contracts and non-obvious decisions MUST be
documented. Errors MUST be surfaced explicitly; broad exception handling, silent
fallbacks, and dead code are prohibited. Reviewers MUST reject unjustified complexity.

Rationale: Fitness booking is trust-sensitive. Straightforward code reduces defects,
speeds review, and makes booking rules safe to change.

### II. Tests Define Done

Every behavior change MUST include automated tests at the lowest effective level.
Business rules require unit tests, boundaries and integrations require integration or
contract tests, and critical user journeys require end-to-end coverage. Bug fixes MUST
first add a regression test that reproduces the failure. Tests MUST cover success,
failure, validation, and relevant boundary cases; they MUST be deterministic and
isolated from uncontrolled time, network, and shared state. All affected tests and
required static checks MUST pass before merge. A documented exception is permitted only
for changes that cannot alter runtime behavior.

Rationale: Executable evidence protects booking correctness and prevents regressions
across independently delivered features.

### III. Consistent, Accessible User Experience

Features MUST reuse established interaction patterns, components, terminology, visual
tokens, and navigation behavior. Every user-facing flow MUST define loading, empty,
success, validation, and failure states and preserve the user's context when recovery is
possible. Interfaces MUST support keyboard operation, visible focus, semantic structure,
clear labels, sufficient contrast, and WCAG 2.1 AA expectations. User-facing copy MUST
be concise and consistent with the product vocabulary. Specifications and reviews MUST
include acceptance criteria for responsive behavior and accessibility where a user
interface changes.

Rationale: Consistency and accessibility let users book classes confidently across
devices and abilities.

### IV. Measurable Performance

Every feature that can affect latency, rendering, throughput, memory, or payload size
MUST define a measurable performance budget and the expected workload before
implementation. Unless a stricter feature budget is approved, user input MUST receive
visible feedback within 100 ms, primary pages MUST render useful content within 2.5
seconds at the 75th percentile under the documented reference conditions, and
user-facing booking operations MUST complete within 2 seconds at the 95th percentile
under expected load. Performance-sensitive changes MUST include a repeatable benchmark,
load test, or instrumentation evidence. Regressions beyond an approved budget block
release unless explicitly accepted with a remediation plan.

Rationale: Booking demand is time-sensitive; predictable responsiveness is a functional
requirement, not a post-release optimization.

## Quality and Delivery Standards

- Feature specifications MUST contain independently testable acceptance scenarios,
  explicit UX states, accessibility requirements, and measurable performance outcomes
  whenever those concerns apply.
- Plans MUST identify quality tooling, test layers, performance budgets, expected scale,
  and any complexity that needs constitutional justification.
- Tasks MUST include implementation, automated testing, UX/accessibility validation, and
  performance verification as part of the relevant user story rather than deferred
  polish.
- Dependencies MUST be justified and kept current. New dependencies require evidence
  that existing project or platform capabilities cannot meet the need.
- Security, privacy, and data integrity requirements MUST not be weakened to satisfy
  delivery speed or performance targets.

## Development Workflow and Quality Gates

Work MUST proceed from specification to plan to independently deliverable tasks. Before
implementation, the plan MUST pass a Constitution Check covering code simplicity,
required test levels, UX consistency and accessibility, and measurable performance
budgets. After design, that check MUST be repeated against the concrete architecture.

Pull requests MUST be narrowly scoped, explain user-visible and contract changes, and
provide evidence for each applicable quality gate. Reviewers MUST verify tests and
acceptance criteria, inspect error and recovery states, and compare measured performance
with the approved budget. Any principle violation MUST appear in the plan's Complexity
Tracking table with a reason, rejected simpler alternatives, an owner, and a removal or
review date.

## Governance

This constitution supersedes conflicting local practices and templates. Amendments MUST
be proposed in writing, explain their impact, update dependent templates in the same
change, and be approved through review. Versioning follows semantic versioning: MAJOR
for incompatible principle or governance changes, MINOR for new principles or material
expansions, and PATCH for clarifications that do not change obligations.

Every specification, plan, task list, and pull request MUST be checked for compliance.
Reviewers MUST block unexplained violations. Approved exceptions MUST identify scope,
rationale, owner, risk, and an expiration or review date; recurring exceptions require a
constitutional amendment rather than precedent by repetition.

**Version**: 1.0.0 | **Ratified**: 2026-07-20 | **Last Amended**: 2026-07-20
