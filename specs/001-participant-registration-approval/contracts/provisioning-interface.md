# Contract: ProvisioningService

**Feature**: [../spec.md](../spec.md) | **Plan**: [../plan.md](../plan.md) | **Data model**: [../data-model.md](../data-model.md)

This is the pluggable interface Constitution Principle II requires. Every infrastructure-touching action in this feature (FR-006) goes through it. There are exactly two implementations — `AwsProvisioningService` and `StubProvisioningService` — and both MUST satisfy this same contract, verified by a shared contract-test suite (see `research.md` Decision 5).

## Operation: `provision(request) -> ProvisioningResult`

Called exactly once per `Participant`, immediately after a `Registration` is approved (FR-006, FR-007).

**Request**:

| Field | Type | Notes |
|---|---|---|
| `participantId` | string | The `Participant` this provisioning run is for |
| `teamRef` | object `{ ec2InstanceRef: string, securityGroupRef: string }` | From the assigned `Team` record |
| `linuxUsername` | string | Derived username for the OS login |
| `admitIp` | string | The public IP captured at registration (data-model.md `Registration.submittedIp`) |

**Response** (`ProvisioningResult`):

| Field | Type | Notes |
|---|---|---|
| `publicKey` | string | Always returned on success — safe to persist |
| `privateKey` | string | Returned only transiently to the caller for a single encrypted write; never logged, never persisted in plaintext (Principle I) |
| `actions` | array of `{ action, outcome, errorDetail? }` | One entry per `ProvisioningRecord.action` enum value — `generate_keypair`, `create_linux_login`, `admit_security_group_ip` |
| `implementation` | enum `stub` \| `aws` | Echoes which implementation ran, for the audit trail |

**Behavioral guarantees both implementations MUST uphold**:

1. All three actions (`generate_keypair`, `create_linux_login`, `admit_security_group_ip`) are attempted; a failure in one does not silently skip recording the others' outcomes (supports FR-014's audit requirement and the Edge Case of partial provisioning failure).
2. Calling `provision()` twice for the same `participantId` is rejected by the caller before this interface is invoked a second time (FR-007 is enforced by the `ApprovalService`, not by this interface, since the interface itself is stateless per call).
3. `StubProvisioningService` MUST return a syntactically valid keypair and MUST record `outcome: success` for all three actions by default, so that end-to-end flows (including Playwright e2e tests and any participant's forked deployment) work without real infrastructure — but it MUST NOT import, hold, or be able to construct a client capable of reaching real EC2, SSM, or security-group APIs. This is a static code property (no such import exists in that module), not a runtime check.
4. `AwsProvisioningService` MUST NOT be reachable from any deployment that has not explicitly set the production deployment flag (Constitution Principle III; see `research.md` Decision 3 and 4).

## Operation: `describeAttempt(participantId) -> ProvisioningRecord[]`

Read-only. Returns the audit rows for a participant (FR-014), used by the admin UI and by SC-005's audit verification. Both implementations read from the same `ProvisioningRecord` store (data-model.md) — this operation itself never touches real infrastructure.

## Non-goals of this contract

- Team/security-group/EC2 *creation* — teams and their infrastructure already exist (spec Assumptions); this contract only mutates an existing team's security group and creates a login on its existing instance.
- Re-admitting a changed IP after initial provisioning — out of scope per spec Assumptions; not an operation on this interface.
