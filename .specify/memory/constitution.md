<!--
SYNC IMPACT REPORT
==================
Version change: (template / unversioned) → 1.0.0
Bump rationale: Initial ratification — placeholder template replaced with concrete
principles, governance, and quality gates. First numbered version per semver.

Modified principles:
  - [PRINCIPLE_1_NAME]            → I. Code Quality (NON-NEGOTIABLE)
  - [PRINCIPLE_2_NAME]            → II. Test-Driven Development (NON-NEGOTIABLE)
  - [PRINCIPLE_3_NAME]            → III. Consistency
  - [PRINCIPLE_4_NAME]            → IV. Performance Requirements
  - [PRINCIPLE_5_NAME]            → REMOVED (user requested four principles)

Added sections:
  - Core Principles (4 principles defined)
  - Quality Gates & Development Workflow
  - Governance

Removed sections:
  - Fifth principle slot ([PRINCIPLE_5_NAME])
  - Distinct [SECTION_3_NAME] block (merged into Governance + Quality Gates)

Templates requiring updates:
  - ✅ .specify/memory/constitution.md (this file)
  - ⚠ .specify/templates/plan-template.md — "Constitution Check" section is a
    placeholder; should be replaced with the concrete gate checklist below.
  - ⚠ .specify/templates/tasks-template.md — Tests are marked OPTIONAL, which
    conflicts with Principle II (TDD NON-NEGOTIABLE). Reconcile wording so test
    tasks are mandatory under this constitution.
  - ✅ .specify/templates/spec-template.md — Success Criteria with measurable
    outcomes already supports Principle IV (Performance Requirements); no change
    required.
  - ✅ .specify/templates/commands/*.md — No outdated agent-specific references
    detected requiring updates from this amendment.

Follow-up TODOs:
  - TODO(plan-template): Replace "[Gates determined based on constitution file]"
    with the four-gate checklist enumerated in this constitution.
  - TODO(tasks-template): Update the "Tests are OPTIONAL" notice to "Tests are
    MANDATORY under Principle II" for any project governed by this constitution.
  - TODO(PROJECT_NAME): Project name was inferred from the working directory
    ("speckit_test"). Confirm or rename in a PATCH amendment.
-->

# Speckit Test Constitution

## Core Principles

### I. Code Quality (NON-NEGOTIABLE)

All code MUST satisfy the following before merge:

- Automated linting and formatting checks pass with zero warnings; style is enforced by
  tooling, not by reviewers.
- Cyclomatic complexity per function MUST NOT exceed 10 without a written justification
  recorded in the PR description and a tracked refactor ticket.
- Public functions, classes, and modules MUST carry docstrings or header comments
  stating purpose, inputs, outputs, and error conditions. Internal helpers MAY omit
  documentation only when their name and signature make intent self-evident.
- Dead code, commented-out blocks, unused imports, and TODOs without an owner or issue
  link MUST be removed before merge.
- Static analysis findings (security, type, lint) MUST be either fixed, suppressed with
  an inline rationale, or converted into a tracked issue. They MUST NOT be ignored.
- Every change MUST receive at least one peer review whose explicit responsibility is
  verifying compliance with this constitution, not only functional correctness.

**Rationale**: Sustained delivery velocity depends on a codebase any contributor can
read, change, and extend safely. Quality enforced at submit time is dramatically cheaper
than quality recovered through later debugging or rewrites.

### II. Test-Driven Development (NON-NEGOTIABLE)

TDD is mandatory for all production code:

- Tests MUST be written before production code. The Red → Green → Refactor cycle MUST
  be visible in commit history (failing test, then implementation).
- Every functional requirement (`FR-*`) and acceptance scenario in `spec.md` MUST map
  to at least one automated test before its implementation task is marked complete.
- Line coverage MUST be ≥ 80% on new and changed code; branch coverage MUST be ≥ 70%.
  Coverage gates are enforced in CI and block merge when violated.
- Tests MUST be deterministic. Flaky tests MUST be quarantined within 24 hours of
  detection and either fixed or deleted within 7 days. Permanent skip annotations
  require an open issue link.
- Integration tests MUST cover: every public API contract, every cross-module boundary,
  every persistence-layer interaction, and every external-service integration.
- Implementations that lack a corresponding failing-test trail SHALL NOT be merged.
  Reviewers MUST confirm the test-first sequence before approval.

**Rationale**: TDD turns requirements into executable specifications, prevents drift
between intent and behavior, and makes refactoring safe. Skipping it trades a small
short-term saving for an unbounded long-term cost.

### III. Consistency

Uniformity is enforced across the codebase:

- One coding style per language, defined in repository config (`.editorconfig`, linter
  configs, formatter configs). Style is not subject to PR-time debate.
- Naming conventions, directory layout, and module boundaries MUST follow the patterns
  recorded in `plan-template.md`. Deviations MUST be entered in the plan's
  **Complexity Tracking** table with a justification.
- Public API shapes, error formats, logging structure (fields, levels, format), and
  configuration mechanisms (env vars, config files) MUST be uniform across modules of
  the same project.
- New dependencies MUST be justified in writing: problem solved, alternatives
  considered, license verified. Adding a dependency that duplicates an existing one's
  purpose is forbidden.
- User-facing terminology MUST be consistent across documentation, UI, error messages,
  and log output. One concept, one name.

**Rationale**: Consistency reduces cognitive load, accelerates onboarding, and prevents
the subtle class of bugs that emerges when conventions silently diverge between modules.

### IV. Performance Requirements

Performance is a first-class design concern, not a polish step:

- Every feature plan MUST declare measurable performance budgets in `plan.md` under
  **Performance Goals** and **Constraints**: throughput, p95 / p99 latency, memory
  ceiling, startup time, and any domain-specific targets that apply.
- Any change that touches a hot path (request handling, persistence, render loop,
  ingest pipeline, or any path explicitly tagged "performance-critical") MUST include
  or update an automated benchmark that verifies the declared budget.
- A regression exceeding 5% on a declared budget MUST block merge until either fixed
  or formally re-baselined with documented justification.
- Non-trivial routines MUST document algorithmic complexity (time and space, Big-O) in
  the source where they are defined.
- Performance claims MUST be backed by measurement, not intuition. Required
  observability (timing, counters, traces sufficient to validate the budget) MUST be
  in place before performance work is declared complete.

**Rationale**: Performance deferred is performance lost — and the rework cost
compounds. Explicit budgets, automated verification, and measurement-before-claim turn
performance from a vague aspiration into a testable contract.

## Quality Gates & Development Workflow

The following gates apply to every feature governed by this constitution. They are
referenced from `plan-template.md` (Constitution Check) and enforced in CI where
automatable.

1. **Code Quality Gate**: Lint, format, static analysis, and complexity checks pass.
   Peer review approved with explicit constitution-compliance sign-off.
2. **TDD Gate**: Failing test exists and was committed before its implementation.
   Coverage thresholds (≥ 80% line, ≥ 70% branch on changed code) met. No flaky or
   unreviewed skip markers introduced.
3. **Consistency Gate**: No new style violations. No undocumented deviations from the
   project's naming, layout, error-format, or logging conventions. Any new dependency
   has a recorded justification.
4. **Performance Gate**: Plan declares performance budgets. Hot-path changes carry a
   passing benchmark. No undeclared regression > 5% against the prior baseline.

A PR that cannot satisfy a gate MUST either remediate or record the violation in the
plan's **Complexity Tracking** table with a concrete simpler alternative considered and
rejected. Unjustified violations block merge.

## Governance

- This constitution supersedes ad-hoc team practice. When guidance conflicts, the
  constitution wins; conflicting guidance MUST be updated or removed.
- Amendments require a pull request that:
  1. Edits this file with the proposed change.
  2. Increments the version per the semver rules below.
  3. Updates `Last Amended` to the merge date in ISO format (YYYY-MM-DD).
  4. Includes a migration note for any change that affects existing artifacts.
- **Versioning policy**:
  - MAJOR — backward-incompatible governance or principle removals/redefinitions.
  - MINOR — new principle, new section, or materially expanded guidance.
  - PATCH — clarifications, wording, typos, non-semantic refinements.
- Every PR description MUST include a one-line **Constitution Check** confirming the
  change either complies with the four principles or documents the violation in
  Complexity Tracking.
- Maintainers SHOULD reassess principle effectiveness on a quarterly cadence and open
  amendment PRs where reality has outgrown the rules. A stale principle is a hazard.
- Runtime collaboration guidance for AI agents and human contributors lives in
  `CLAUDE.md` and the templates under `.specify/templates/`. Those files MUST stay
  consistent with this constitution; any divergence is a defect to be fixed.

**Version**: 1.0.0 | **Ratified**: 2026-05-13 | **Last Amended**: 2026-05-13
