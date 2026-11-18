# Quickstart: Validating Participant Registration & Approval

**Feature**: [spec.md](./spec.md) | **Data model**: [data-model.md](./data-model.md) | **Contracts**: [contracts/](./contracts/)

Validates the three user stories end-to-end against the **stub** `ProvisioningService` — no real AWS infrastructure is touched, consistent with Constitution Principle II. Treat this as the acceptance pass before `/speckit-tasks` → `/speckit-implement` work is considered done.

## Prerequisites

- Local deployment running with `DEPLOYMENT_ENV` unset or set to anything other than `production`, so `StubProvisioningService` is selected (`research.md` Decision 3).
- A seeded `Team` record (any `teamId`, e.g. `team-alpha`) — this feature does not create teams, so one must exist before approval can be tested (spec Assumptions).
- A test Cognito user whose email matches the registration email used below.
- An authenticated admin session for the admin endpoints/UI.

## Scenario 1 — Visitor registers (User Story 1)

1. Submit the public registration form (or `POST /api/registrations`) with a fresh email and GitHub ID.
2. **Expect**: `201 Created`, response status `pending`. The admin registrations list now includes this entry.
3. Re-submit the same email (or same GitHub ID) again.
4. **Expect**: `409 Conflict` — confirms FR-002's duplicate rule.
5. Submit with a malformed email.
6. **Expect**: `400 Bad Request`.

## Scenario 2 — Administrator approves, provisioning fires once (User Story 2)

1. As the admin, call `POST /api/admin/registrations/{registrationId}/approve` with `{ "teamId": "team-alpha", "role": "participant" }`.
2. **Expect**: `200 OK`, `status: "approved"`, and a `provisioning` object whose `implementation` field reads `"stub"` and whose `actions` array shows all three actions (`generate_keypair`, `create_linux_login`, `admit_security_group_ip`) with `outcome: "success"`.
3. Call the same approve endpoint again for the same `registrationId`.
4. **Expect**: `409 Conflict` — confirms FR-007 (no re-provisioning).
5. Inspect the `ProvisioningRecord` rows for this participant (data-model.md) — all four `implementation` values across the audit trail read `"stub"`, never `"aws"`. This is the manual check behind SC-005.
6. Separately, submit and reject a second registration via `POST /api/admin/registrations/{registrationId}/reject`.
7. **Expect**: `200 OK`, `status: "rejected"`, and no `ProvisioningRecord` rows created for it (FR-008).

## Scenario 3 — Participant signs in and retrieves credentials once (User Story 3)

1. Sign in via Cognito as the test user whose email matches the approved registration from Scenario 2.
2. Call `GET /api/portal/credentials`.
3. **Expect**: `200 OK` with a non-null `privateKey` and `setupInstructions` text that references `team-alpha` and the assigned role.
4. Call `GET /api/portal/credentials` again in the same or a new session.
5. **Expect**: `200 OK` with `privateKey: null`, `setupInstructions` still present — confirms FR-011.
6. Sign in as a Cognito user whose email has no approved `Participant` record (e.g., still-pending or rejected).
7. **Expect**: `403 Forbidden` — confirms FR-013.

## Pass criteria

All six expectations above hold. Any deviation means a functional requirement in `spec.md` is not yet satisfied — fix before proceeding to `/speckit-implement` sign-off.
