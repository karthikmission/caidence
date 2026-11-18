<!--
Sync Impact Report
Version change: 1.0.0 → 1.0.1
Modified principles: none
Added sections: none
Removed sections: none
Changed: Governance — named the initial-ratification PR exception explicitly;
every amendment after 1.0.0 requires a PR with no exception. This amendment
itself goes through a PR (branch: chore/bootstrap-governance-exception),
unlike 1.0.0 which could not.
Follow-up TODOs: none
-->

# Caidence Constitution

## Core Principles

### I. No Secrets In The Repo (NON-NEGOTIABLE)
This repository is public. No infrastructure secrets, account IDs, ARNs, resource
identifiers, access tokens, or private keys MUST ever be committed to it, in any
branch, commit, issue, or PR description. Configuration that is genuinely secret
is supplied at runtime via environment variables or a secrets manager, never via a
file tracked by git. Rationale: a public repo has no blast-radius boundary —
anything committed is permanently exposed, even if later deleted.

### II. Provisioning Is Pluggable
The privileged provisioning step — creating a real OS login, mutating a real
security group, or issuing a real private key — MUST sit behind an interface with
two implementations: a real one (used only by the production deployment) and a
safe stub (used by every other deployment, including every participant's own).
The stub MUST NOT have the ability to reach real infrastructure, by construction,
not by configuration. Rationale: this is what makes it safe to let anyone fork and
run this codebase without first auditing their intentions.

### III. The Starter Branch Cannot Touch Production
A participant-facing starter branch of this codebase MUST NOT be able to mutate
the production approval/provisioning path, whether by shared credentials, a
shared database, or a shared deployment target. Each participant's deployment is
independent of, and has no privileged access to, the production instance.
Rationale: the production instance gates real access during a live event; a bug
in a participant's in-progress Bolt must never be able to affect it.

### IV. Built By The Methodology It Teaches
Caidence is built using the AI-DLC + Balrog pipeline it exists to demonstrate:
spec → plan → tasks → implement → review → ship. Every feature traces back to a
spec and plan artifact in `specs/`, not to undocumented code. Rationale: the
artifacts this produces double as the workshop's own teaching material.

## Security Requirements

All IAM roles and permissions follow least privilege — a role is scoped to
exactly what its function needs, never broadened for convenience. Credentials
used by the production deployment (Cognito app client secrets, database
connection details, provisioning-service credentials) are supplied only through
environment variables or a secrets manager at deploy time, and are never logged,
echoed, or written to a file tracked by git. The stub provisioning
implementation (Principle II) must be the default for any environment that is
not explicitly the production deployment.

## Development Workflow

Work proceeds as: `/speckit-specify` to capture requirements, `/speckit-plan` to
produce the design and contracts, `/speckit-tasks` to break the plan into ordered
tasks, `/speckit-implement` to build against those tasks, then review and ship.
A change that skips straight to implementation without a corresponding spec/plan
artifact is out of process and must be brought back into `specs/` before it
merges.

## Governance

This constitution supersedes ad hoc practice. Amendments require a PR that
states the version bump and rationale, and are only adopted once merged. The
one exception is the initial ratification itself (version 1.0.0): a document
that defines the PR requirement cannot be reviewed under a rule it has not yet
adopted. Every version after 1.0.0 follows the PR requirement with no further
exception. Versioning is semantic: MAJOR for incompatible governance changes,
MINOR for a new or materially expanded principle, PATCH for wording/clarification
only. Every spec and plan is checked against this constitution before
implementation begins; a conflict is resolved by amending either the spec or
this document, not by silently ignoring one.

**Version**: 1.0.1 | **Ratified**: 2026-11-18 | **Last Amended**: 2026-11-18
