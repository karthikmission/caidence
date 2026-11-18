---

description: "Task list template for feature implementation"
---

# Tasks: Participant Registration & Approval

**Input**: Design documents from `/specs/001-participant-registration-approval/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/, quickstart.md (all present)

**Tests**: Included. `plan.md` commits to a Vitest contract-test suite run against both `ProvisioningService` implementations specifically to verify Constitution Principle II (pluggable provisioning, safe-by-construction stub) and Plus Playwright e2e per `quickstart.md`; these are design-mandated, not incidental.

**Organization**: Tasks are grouped by user story (spec.md priorities P1/P2/P3) to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (US1, US2, US3)
- Exact file paths are included in every description

## Path Conventions

Per `plan.md` Project Structure: single Next.js app (`app/`, `src/`) plus a separate CDK app (`infra/`), tests under `tests/{unit,contract,e2e}/`.

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization and basic structure

- [ ] T001 Initialize the Next.js 14+ (App Router, TypeScript 5.x, Node 20) application at the repository root with the `app/`, `src/`, and `tests/` directories from `plan.md` Project Structure
- [ ] T002 [P] Initialize the `infra/` AWS CDK (TypeScript) app skeleton for the Cognito user pool, DynamoDB tables, and hosting that `research.md` Decisions 2 and 4 call for
- [ ] T003 [P] Configure ESLint + Prettier for the Next.js app
- [ ] T004 [P] Configure Vitest (unit + contract tests) and Playwright (e2e) per `plan.md` Testing

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before ANY user story can be implemented

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

- [ ] T005 Implement the DynamoDB client wrapper and per-table environment-driven table-name configuration in `src/data/dynamo-client.ts` (`research.md` Decision 2)
- [ ] T006 [P] Implement Cognito session/JWT verification middleware in `src/auth/session.ts`, exposing the authenticated caller's email for downstream matching against `Participant.email`
- [ ] T007 [P] Implement shared domain validation utilities — email format check and non-empty GitHub ID check (data-model.md `Registration` validation rules) — in `src/domain/validation.ts`
- [ ] T008 [P] Define the `ProvisioningService` interface (`provision(request) -> ProvisioningResult` and `describeAttempt(participantId) -> ProvisioningRecord[]`, exact shapes per `contracts/provisioning-interface.md`) in `src/provisioning/interface.ts`
- [ ] T009 Implement `StubProvisioningService` in `src/provisioning/stub-provisioning.ts`: MUST NOT import any AWS SDK client capable of mutating EC2, SSM, or security-group state (safety by construction per Constitution Principle II); returns a syntactically valid keypair and records `outcome: success` for all three actions (`generate_keypair`, `create_linux_login`, `admit_security_group_ip`) by default (depends on T008)
- [ ] T010 Implement environment-based `ProvisioningService` selection in `src/provisioning/index.ts` — defaults to `StubProvisioningService` for any value of `DEPLOYMENT_ENV` other than exactly `production` (depends on T008, T009)
- [ ] T011 [P] Implement a logging utility in `src/lib/logger.ts` that structurally cannot be passed a private key, password, or credential field (Constitution Principle I — no secrets ever logged)

**Checkpoint**: Foundation ready — user story implementation can now begin

---

## Phase 3: User Story 1 - Visitor registers for the workshop (Priority: P1) 🎯 MVP

**Goal**: A visitor submits email + GitHub ID through a public form with no sign-in required; duplicates and malformed input are rejected; valid submissions are recorded as `pending`.

**Independent Test**: Submit the public registration form with a valid email and GitHub ID and confirm the submission is recorded with status `pending` and the visitor receives a pending confirmation (spec.md Acceptance Scenario 1).

### Tests for User Story 1

- [ ] T012 [P] [US1] Contract test for `POST /api/registrations` covering 201 (valid), 400 (malformed email / empty GitHub ID), and 409 (duplicate email or GitHub ID) per `contracts/api.md` in `tests/contract/registrations.test.ts`
- [ ] T013 [P] [US1] Unit test for `Registration` validation rules — malformed email rejected, empty GitHub ID rejected, and "an existing registration with this email or this githubId already has status pending or approved" blocks a new submission (data-model.md validation rules) — in `tests/unit/registration-validation.test.ts`

### Implementation for User Story 1

- [ ] T014 [P] [US1] Create the `Registration` domain type and its `pending → approved | rejected` state machine (data-model.md State Transitions: `approved` and `rejected` are terminal) in `src/domain/registration.ts`
- [ ] T015 [US1] Implement `RegistrationsRepository` (DynamoDB) with `create`, `findByEmail`, and `findByGithubId` lookups in `src/data/registrations-repository.ts` (depends on T005, T014)
- [ ] T016 [US1] Implement `RegistrationService.submit()` enforcing: valid email format and non-empty GitHub ID (via `src/domain/validation.ts`), and rejecting submission when a non-rejected (`pending` or `approved`) registration already exists for the same email or the same GitHub ID (data-model.md validation rule) in `src/services/registration-service.ts` (depends on T007, T015)
- [ ] T017 [US1] Implement the `POST /api/registrations` route handler returning `201 { registrationId, status: "pending" }`, `400` on malformed input, `409` on duplicate, per `contracts/api.md` in `app/api/registrations/route.ts` (depends on T016)
- [ ] T018 [US1] Build the public registration form UI (email + GitHub ID fields, client-side validation, pending-confirmation message) in `app/(public)/register/page.tsx` (depends on T017)

**Checkpoint**: User Story 1 is fully functional and independently testable — a visitor can register and see a pending confirmation.

---

## Phase 4: User Story 2 - Administrator reviews and approves a registration (Priority: P2)

**Goal**: An administrator assigns a team and role to a pending registration and approves or rejects it; approval triggers the pluggable `ProvisioningService` exactly once.

**Independent Test**: As an administrator, assign a team and role to a pending registration and approve it; confirm the provisioning actions were invoked exactly once (via `ProvisioningRecord` rows) and the registration's status moves to `approved` (spec.md Acceptance Scenario 1 for US2).

### Tests for User Story 2

- [ ] T019 [P] [US2] Contract test for `ProvisioningService`, run against `StubProvisioningService`, asserting the three behavioral guarantees in `contracts/provisioning-interface.md` (all three actions attempted and recorded individually; stub always succeeds; stub has no AWS client import) in `tests/contract/provisioning-service.test.ts`
- [ ] T020 [P] [US2] Contract test for `GET /api/admin/registrations`, `POST /api/admin/registrations/{id}/approve`, and `.../reject` covering 200/404/409 per `contracts/api.md` in `tests/contract/admin-registrations.test.ts`
- [ ] T021 [P] [US2] Integration test for the approval flow: approve once → exactly one set of `ProvisioningRecord` rows with `implementation: "stub"` and `outcome: "success"` for all three actions; approving the same registration again returns 409 (quickstart.md Scenario 2, FR-007) in `tests/integration/approval-flow.test.ts`

### Implementation for User Story 2

- [ ] T022 [P] [US2] Create the `Team` domain type (`teamId`, `name`, `ec2InstanceRef`, `securityGroupRef`, `setupInstructionsTemplate` per data-model.md) in `src/domain/team.ts`
- [ ] T023 [P] [US2] Create the `Participant` domain type (`participantId`, `registrationId`, `email`, `linuxUsername`, `teamId`, `role`, `privateKeyRevealed` per data-model.md) in `src/domain/participant.ts`
- [ ] T024 [P] [US2] Create the `ProvisioningRecord` domain type with `action` enum (`generate_keypair` \| `create_linux_login` \| `admit_security_group_ip`), `outcome` enum (`success` \| `failure`), and `implementation` enum (`stub` \| `aws`) per data-model.md in `src/domain/provisioning-record.ts`
- [ ] T025 [US2] Implement `TeamsRepository` (DynamoDB, read-only lookup by `teamId`) in `src/data/teams-repository.ts` (depends on T005, T022)
- [ ] T026 [US2] Implement `ParticipantsRepository` (DynamoDB) with `create`, `findByEmail`, and a `revealPrivateKey` operation that sets `privateKeyRevealed = true` and deletes `privateKeyCiphertext` in the same write (data-model.md Security Rule) in `src/data/participants-repository.ts` (depends on T005, T023)
- [ ] T027 [US2] Implement `ProvisioningRecordsRepository` (DynamoDB) with `create` and `listByParticipant` in `src/data/provisioning-records-repository.ts` (depends on T005, T024)
- [ ] T028 [US2] Implement `ApprovalService.approve()`: reject with 409 if `Registration.status !== "pending"`; require both `teamId` and `role` to be set before transitioning to `approved` (FR-004); call `ProvisioningService.provision()` exactly once; persist the resulting `Participant` and `ProvisioningRecord` rows; transition `Registration` to `approved` in `src/services/approval-service.ts` (depends on T009, T010, T015, T025, T026, T027)
- [ ] T029 [US2] Implement `ApprovalService.reject()`: reject with 409 if `Registration.status !== "pending"`; otherwise transition to `rejected` with no `ProvisioningService` call (FR-008) in `src/services/approval-service.ts` (depends on T015)
- [ ] T030 [US2] Implement the `GET /api/admin/registrations`, `POST /api/admin/registrations/{registrationId}/approve`, and `POST /api/admin/registrations/{registrationId}/reject` route handlers per `contracts/api.md` in `app/api/admin/registrations/route.ts` and `app/api/admin/registrations/[registrationId]/approve/route.ts` and `app/api/admin/registrations/[registrationId]/reject/route.ts` (depends on T028, T029)
- [ ] T031 [US2] Build the admin review UI — pending-registrations list, team/role assignment picker, approve/reject actions — in `app/(admin)/admin/page.tsx` (depends on T030)

**Checkpoint**: User Stories 1 AND 2 both work independently — registrations flow through to approval and provisioning.

---

## Phase 5: User Story 3 - Approved participant signs in and retrieves their credentials (Priority: P3)

**Goal**: A Cognito-authenticated, approved participant sees their private key exactly once, plus setup instructions customized to their team and role.

**Independent Test**: Sign in as an approved participant who has not yet viewed their credentials and confirm the private key and team/role-specific setup instructions display; sign in again and confirm the private key no longer displays (spec.md Acceptance Scenarios 1–2 for US3).

**Note**: Functionally, this story needs an approved `Participant` record, which User Story 2 creates — so while its own code (routes, service, UI) is independent, exercising it end-to-end requires US2 to have run first.

### Tests for User Story 3

- [ ] T032 [P] [US3] Contract test for `GET /api/portal/credentials` covering: 200 with non-null `privateKey` on first call, 200 with `privateKey: null` on subsequent calls, and 403 when the caller has no approved `Participant` record, per `contracts/api.md` in `tests/contract/portal-credentials.test.ts`
- [ ] T033 [P] [US3] Integration test confirming `privateKeyCiphertext` is deleted (not merely flagged) immediately after the first reveal, per the data-model.md Security Rule and FR-011, in `tests/integration/key-reveal.test.ts`

### Implementation for User Story 3

- [ ] T034 [US3] Implement participant authorization: match the authenticated Cognito caller's email to an approved `Participant.email`; return 403 otherwise (FR-013, covers a pending/rejected visitor attempting sign-in) in `src/auth/session.ts` (depends on T006, T026)
- [ ] T035 [US3] Implement `PortalService.getCredentials()`: return `{ privateKey, setupInstructions }` on first call, `{ privateKey: null, setupInstructions }` on every subsequent call, and delete `privateKeyCiphertext` immediately after the first reveal (data-model.md Security Rule, FR-011) in `src/services/portal-service.ts` (depends on T026, T034)
- [ ] T036 [US3] Render team/role-specific setup instructions from `Team.setupInstructionsTemplate` (FR-012) in `src/services/portal-service.ts` (depends on T025, T035)
- [ ] T037 [US3] Implement the `GET /api/portal/credentials` route handler per `contracts/api.md` in `app/api/portal/credentials/route.ts` (depends on T035, T036)
- [ ] T038 [US3] Build the participant portal UI — one-time private key display plus setup instructions — in `app/(participant)/portal/page.tsx` (depends on T037)

**Checkpoint**: All user stories are now independently functional — registration, approval/provisioning, and credential retrieval all work end-to-end against the stub.

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: The production-only path and verification work that spans every story

- [ ] T039 [P] Implement `AwsProvisioningService` in `src/provisioning/aws-provisioning.ts` — real EC2 security-group ingress mutation, SSM-based Linux login creation, and SSH keypair generation — reachable only when `DEPLOYMENT_ENV=production` selects it via T010 (Constitution Principle II, `research.md` Decision 3)
- [ ] T040 [P] Extend the shared `ProvisioningService` contract-test suite from T019 to also run against `AwsProvisioningService` with the AWS SDK clients mocked at the boundary, so both implementations are verified against the same contract (`research.md` Decision 5) in `tests/contract/provisioning-service.test.ts`
- [ ] T041 [P] Author the CDK stack(s) in `infra/` provisioning a per-deployment Cognito user pool, DynamoDB tables, and least-privilege IAM roles, with no shared stack/credentials between the production deployment and any other (Constitution Principle III, Security Requirements, `research.md` Decision 4)
- [ ] T042 [P] Write the Playwright e2e spec covering `quickstart.md` Scenarios 1–3 end-to-end against the stub-backed deployment in `tests/e2e/participant-flow.spec.ts`
- [ ] T043 Audit the repository for hardcoded secrets or real infrastructure identifiers (account IDs, ARNs, credentials) per Constitution Principle I
- [ ] T044 Run the full `quickstart.md` validation pass and confirm all six pass-criteria expectations hold

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — start immediately
- **Foundational (Phase 2)**: Depends on Setup — BLOCKS all user stories
- **User Story 1 (Phase 3)**: Depends on Foundational only
- **User Story 2 (Phase 4)**: Depends on Foundational only for its own code; needs US1's `RegistrationsRepository` (T015) to read/update registrations
- **User Story 3 (Phase 5)**: Depends on Foundational only for its own code; needs US2's `Participant` creation (T026, T028) to have real data to serve
- **Polish (Phase 6)**: Depends on all three user stories being complete

### Within Each User Story

- Tests are written before their corresponding implementation tasks
- Domain types before repositories
- Repositories before services
- Services before route handlers
- Route handlers before UI

### Parallel Opportunities

- All Setup tasks marked [P] (T002–T004) run in parallel
- All Foundational tasks marked [P] (T006, T007, T008, T011) run in parallel; T009 and T010 are sequential after T008
- All US1 tests marked [P] (T012, T013) run in parallel; T014 can start alongside them
- All US2 tests marked [P] (T019, T020, T021) run in parallel; T022, T023, T024 (domain types) run in parallel with each other and with the tests
- All US3 tests marked [P] (T032, T033) run in parallel
- All Polish tasks marked [P] (T039–T042) run in parallel

---

## Parallel Example: User Story 1

```bash
# Tests together:
Task: "Contract test for POST /api/registrations in tests/contract/registrations.test.ts"
Task: "Unit test for Registration validation rules in tests/unit/registration-validation.test.ts"

# Domain type alongside the tests:
Task: "Create Registration domain type in src/domain/registration.ts"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup
2. Complete Phase 2: Foundational (CRITICAL — blocks all stories)
3. Complete Phase 3: User Story 1
4. **STOP and VALIDATE**: a visitor can register publicly and see a pending confirmation
5. Demo if ready — note that nothing is provisioned yet, since that requires US2

### Incremental Delivery

1. Setup + Foundational → foundation ready
2. Add User Story 1 → validate → demo (registration works, nothing provisioned yet)
3. Add User Story 2 → validate via `tests/integration/approval-flow.test.ts` → demo (approval provisions via the stub, exactly once)
4. Add User Story 3 → validate via `tests/integration/key-reveal.test.ts` → demo (full loop: register → approve → sign in → one-time key reveal)
5. Polish phase brings in the real `AwsProvisioningService`, CDK infra, e2e coverage, and the secrets audit — none of which block the three stories from being demoed against the stub first

### Parallel Team Strategy

With multiple developers, once Foundational is done, US1 and the domain/type portions of US2 (T022–T024) can start in parallel; US2's service/route tasks need US1's `RegistrationsRepository` (T015), and US3 needs US2's `Participant` creation (T026, T028) — so full parallelism across all three stories is limited by those two data dependencies even though each story's route/UI code lives in separate files.
