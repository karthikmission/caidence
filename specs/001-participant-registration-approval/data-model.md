# Data Model: Participant Registration & Approval

**Feature**: [spec.md](./spec.md) | **Plan**: [plan.md](./plan.md)

Derived from the Key Entities section of `spec.md` and the functional requirements (FR-001–FR-014). No account IDs, ARNs, or other real infrastructure identifiers appear below — those are supplied at deploy time per Constitution Principle I.

## Registration

The visitor-facing record created at submission time (FR-001, FR-002) and owned by the approval workflow (FR-003–FR-005).

| Field | Type | Notes |
|---|---|---|
| `registrationId` | string (ULID) | Primary identifier |
| `email` | string | Validated format; unique among non-rejected registrations (FR-002) |
| `githubId` | string | Validated non-empty; unique among non-rejected registrations (FR-002) |
| `submittedIp` | string | Public IP captured at submission; later admitted to the assigned team's security group on approval (FR-006) |
| `status` | enum: `pending` \| `approved` \| `rejected` | See State Transitions below |
| `assignedTeamId` | string, nullable | Set by an administrator as part of approval (FR-004); null while pending |
| `assignedRole` | string, nullable | Set by an administrator as part of approval (FR-004); null while pending |
| `submittedAt` | timestamp | |
| `decidedAt` | timestamp, nullable | Set when an administrator approves or rejects |
| `decidedBy` | string, nullable | Administrator identifier |

**Validation rules**:
- `email` must be a syntactically valid email address (FR-002).
- `githubId` must be non-empty (FR-002).
- The combination of "an existing registration with this `email` or this `githubId` already has status `pending` or `approved`" MUST block a new submission (FR-002, Edge Case: same email/different GitHub ID or vice versa is treated as a duplicate on whichever field matches).
- `assignedTeamId` and `assignedRole` MUST both be set before a registration can transition to `approved` (FR-004).

**State transitions**:

```text
          (visitor submits)                 (admin approves, team+role set)
 [none] ────────────────────▶ pending ───────────────────────────────────▶ approved
                                 │                                            │
                                 │ (admin rejects)                            │ (terminal — FR-007: no re-provisioning)
                                 ▼
                              rejected ──(visitor may submit a new, independent registration)──▶ pending
```

`approved` and `rejected` are terminal for a given `Registration` record (FR-007, FR-008); a rejected visitor's later attempt creates a new `Registration` row rather than reopening this one (per spec Assumptions).

## Participant

Created at the moment a `Registration` transitions to `approved` (FR-006). Represents the identity that goes on to sign in and retrieve credentials.

| Field | Type | Notes |
|---|---|---|
| `participantId` | string (ULID) | Primary identifier |
| `registrationId` | string | References the originating `Registration` |
| `email` | string | Copied from `Registration` — used to match the Cognito identity at sign-in (FR-010) |
| `linuxUsername` | string | Derived from `githubId` at provisioning time |
| `teamId` | string | Copied from `Registration.assignedTeamId` |
| `role` | string | Copied from `Registration.assignedRole` |
| `privateKeyRevealed` | boolean | Starts `false`; set `true` the first time the portal displays the private key (FR-011) |

**Validation rules**:
- Exactly one `Participant` exists per approved `Registration` (FR-006, FR-007 — provisioning and thus `Participant` creation happens exactly once).
- Once `privateKeyRevealed` is `true`, no code path may return the private key value again (FR-011).

## Team

Pre-existing workshop infrastructure grouping that participants are assigned into (spec Assumptions: teams and their EC2 instance/security group already exist before approval).

| Field | Type | Notes |
|---|---|---|
| `teamId` | string | Primary identifier |
| `name` | string | Display name shown in admin UI and setup instructions |
| `ec2InstanceRef` | string | Opaque reference the `ProvisioningService` resolves to a real instance in production; meaningless to the stub |
| `securityGroupRef` | string | Opaque reference the `ProvisioningService` resolves to a real security group in production; meaningless to the stub |
| `setupInstructionsTemplate` | string | Team-specific portion of the setup instructions shown post-sign-in (FR-012) |

No lifecycle here — this feature only reads `Team` records (to populate the admin's assignment picker and the participant's setup instructions); creating/configuring teams' infrastructure is explicitly out of scope (spec Assumptions).

## SSH Keypair

Generated once per `Participant` at provisioning time (FR-006). Not stored as its own table row with the private key in it — see Security Rule below — but tracked via the `ProvisioningRecord` for audit purposes.

| Field | Type | Notes |
|---|---|---|
| `publicKey` | string | Retained — installed on the team's EC2 instance; safe to store |
| `privateKeyCiphertext` | string | Private key, held only until first reveal, encrypted at rest; never logged (Principle I) |

**Security rule**: `privateKeyCiphertext` MUST be deleted (not just flagged) immediately after the portal marks `Participant.privateKeyRevealed = true`, so FR-011's "not retrievable after first display" is enforced by data deletion, not just an access check.

## Provisioning Record

Audit trail for each attempted provisioning action (FR-014).

| Field | Type | Notes |
|---|---|---|
| `provisioningRecordId` | string (ULID) | Primary identifier |
| `participantId` | string | References the `Participant` this provisioning run was for |
| `action` | enum: `generate_keypair` \| `create_linux_login` \| `admit_security_group_ip` | One row per action attempted |
| `outcome` | enum: `success` \| `failure` | |
| `implementation` | enum: `stub` \| `aws` | Which `ProvisioningService` implementation ran this action — lets an audit confirm a non-production deployment never ran `aws` (supports SC-005) |
| `occurredAt` | timestamp | |
| `errorDetail` | string, nullable | Present only when `outcome = failure`; MUST NOT contain secrets (Principle I) |

## Administrator

Not a data entity this feature creates or manages (spec Assumptions: administrators are a fixed, pre-existing role). Referenced only as `decidedBy` on `Registration` and as the actor in access control for the admin UI/endpoints.

## Relationships

```text
Registration (1) ──creates, on approval──▶ (1) Participant
Participant (many) ──assigned to──▶ (1) Team
Participant (1) ──has──▶ (1) SSH Keypair
Participant (1) ──has many──▶ Provisioning Record
```
