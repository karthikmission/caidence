# Implementation Plan: Participant Registration & Approval

**Branch**: `001-participant-registration-approval` | **Date**: 2026-11-18 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/001-participant-registration-approval/spec.md`

**Note**: This template is filled in by the `/speckit-plan` command; its definition describes the execution workflow.

## Summary

A public web app lets a visitor register with an email and GitHub ID; an administrator reviews pending registrations, assigns a team and role, and approves or rejects them. Approval triggers an automatic, exactly-once provisioning step (unique SSH keypair, Linux login on the team's EC2 instance, security-group IP admission) performed entirely behind a pluggable `ProvisioningService` interface — a real AWS-backed implementation in production, and a safe stub (incapable of reaching real infrastructure by construction) everywhere else. Once approved, the participant signs in via Amazon Cognito and sees their private key exactly once, plus team/role-specific setup instructions.

## Technical Context

**Language/Version**: TypeScript 5.x on Node.js 20 LTS (frontend and backend share one language/runtime)

**Primary Dependencies**: Next.js 14+ (App Router — server actions/route handlers double as the backend), AWS SDK for JavaScript v3 (`@aws-sdk/client-ec2`, `@aws-sdk/client-ssm`), `amazon-cognito-identity-js` / Cognito Hosted UI for sign-in, AWS CDK (TypeScript) for infrastructure

**Storage**: Amazon DynamoDB (Registrations, Teams, ProvisioningRecords tables) — serverless, no shared database connection details to manage per deployment, consistent with Principle III (no shared database between a starter deployment and production)

**Testing**: Vitest for unit/contract tests (including a shared contract-test suite run against both the stub and real `ProvisioningService` implementations), Playwright for end-to-end coverage of the three user stories against the stub implementation

**Target Platform**: Web application, server-side rendered, deployed on AWS (Amplify Hosting or Lambda via OpenNext for the Next.js app)

**Project Type**: Web application — single Next.js codebase providing both the public/admin/participant UI and its backend route handlers, plus a separate CDK app for infrastructure

**Performance Goals**: Registration submission and admin approval actions complete in under 2 seconds p95; no sustained-throughput target — this serves a single workshop cohort (tens to low hundreds of participants), not continuous public traffic

**Constraints**: The stub `ProvisioningService` implementation MUST NOT import or hold a reference to any AWS SDK client capable of mutating EC2/security-group state or creating OS logins — safety by construction, not by a feature flag (Principle II); no infrastructure identifiers, secrets, or credentials committed to the repo (Principle I); each deployment (production vs. any other, including a participant's own fork) uses its own independent Cognito user pool and DynamoDB tables (Principle III)

**Scale/Scope**: O(10) teams per workshop run, O(100) participants per run, a small fixed set of administrators; not designed for multi-tenant/multi-workshop-simultaneously use in v1

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Check | Status |
|---|---|---|
| I. No Secrets In The Repo | Cognito app client secrets, AWS credentials, and any resource identifiers are supplied via environment variables / a secrets manager at deploy time; `data-model.md` and `contracts/` below contain no account IDs, ARNs, or sample credentials | PASS |
| II. Provisioning Is Pluggable | `ProvisioningService` interface defined in `contracts/provisioning-interface.md`, with the stub implementation structurally incapable of importing real-infrastructure AWS clients (see Constraints above and `research.md` Decision 3) | PASS |
| III. Starter Branch Cannot Touch Production | Each deployment gets its own Cognito user pool and DynamoDB tables via CDK context, with the stub `ProvisioningService` as the default for anything not explicitly the production stack; no shared credentials or database — see `research.md` Decision 4 | PASS |
| IV. Built By The Methodology It Teaches | This plan traces to `spec.md` in this same feature directory; `/speckit-tasks` and `/speckit-implement` continue the chain | PASS |

No violations requiring justification — Complexity Tracking is not needed.

*Re-checked after Phase 1 design (`data-model.md`, `contracts/`, `quickstart.md`): the `ProvisioningService` contract keeps the stub's inability to reach real infrastructure a structural property (no AWS client import), `data-model.md` holds no real identifiers, and `quickstart.md` validates entirely against the stub. All four rows above remain PASS.*

## Project Structure

### Documentation (this feature)

```text
specs/001-participant-registration-approval/
├── plan.md              # This file (/speckit-plan command output)
├── research.md          # Phase 0 output (/speckit-plan command)
├── data-model.md        # Phase 1 output (/speckit-plan command)
├── quickstart.md        # Phase 1 output (/speckit-plan command)
├── contracts/           # Phase 1 output (/speckit-plan command)
└── tasks.md             # Phase 2 output (/speckit-tasks command - NOT created by /speckit-plan)
```

### Source Code (repository root)

```text
app/                          # Next.js App Router — UI + backend route handlers in one codebase
├── (public)/
│   └── register/             # Public registration form (no auth)
├── (admin)/
│   └── admin/                 # Pending-registration review, team/role assignment, approve/reject
├── (participant)/
│   └── portal/                 # Post-Cognito-sign-in: one-time key reveal + setup instructions
└── api/
    ├── registrations/          # Submit + duplicate-check endpoints
    └── admin/registrations/    # Approve/reject endpoints (admin-authenticated)

src/
├── domain/                   # Registration, Participant, Team, ProvisioningRecord types + validation
├── services/                 # RegistrationService, ApprovalService (calls ProvisioningService), PortalService
├── provisioning/
│   ├── interface.ts            # ProvisioningService contract (contracts/provisioning-interface.md)
│   ├── aws-provisioning.ts      # Real implementation: EC2/SSM/SG mutation — production deployment only
│   └── stub-provisioning.ts     # Safe stub — default for every other deployment; no AWS infra clients
└── data/                      # DynamoDB repositories for the tables in data-model.md

infra/                        # AWS CDK app: Cognito user pool, DynamoDB tables, hosting, IAM roles

tests/
├── unit/                     # Domain + service unit tests
├── contract/                  # ProvisioningService contract tests, run against stub AND real impl
└── e2e/                        # Playwright: the three spec.md user stories, run against the stub
```

**Structure Decision**: Single Next.js web application (frontend UI and backend route handlers in one codebase) plus a separate `infra/` CDK app, matching "Project Type: Web application" in Technical Context above. The `src/provisioning/` split is load-bearing for Constitution Principle II: `interface.ts` is the pluggable contract, `stub-provisioning.ts` has zero imports of any AWS SDK client that can mutate real infrastructure, and only `aws-provisioning.ts` does — so the safety property is enforced by what each file is allowed to import, not by a runtime flag.

## Complexity Tracking

*No Constitution Check violations — this section is not needed.*
