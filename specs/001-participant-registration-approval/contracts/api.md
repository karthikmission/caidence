# Contract: HTTP Surface

**Feature**: [../spec.md](../spec.md) | **Plan**: [../plan.md](../plan.md)

Request/response shapes for the Next.js route handlers backing the three user stories. Error bodies are `{ "error": string }` throughout; validation errors return 400, auth/authorization failures return 401/403.

## Public (no auth) — User Story 1

### `POST /api/registrations`

Submit a new registration (FR-001, FR-002).

**Request**: `{ "email": string, "githubId": string }`

**Responses**:
- `201 Created` → `{ "registrationId": string, "status": "pending" }`
- `400 Bad Request` → malformed email or empty `githubId`
- `409 Conflict` → a non-rejected registration already exists for this `email` or `githubId`

## Admin (admin-authenticated) — User Story 2

### `GET /api/admin/registrations?status=pending`

List registrations for review (FR-004).

**Response**: `200 OK` → `{ "registrations": Registration[] }` (shape per `data-model.md`)

### `POST /api/admin/registrations/{registrationId}/approve`

Assign team/role and approve; triggers provisioning exactly once (FR-004, FR-005, FR-006, FR-007).

**Request**: `{ "teamId": string, "role": string }`

**Responses**:
- `200 OK` → `{ "status": "approved", "provisioning": ProvisioningResult }` (shape per `contracts/provisioning-interface.md`)
- `404 Not Found` → no such registration
- `409 Conflict` → registration is not in `pending` status (already decided — FR-007/FR-008)

### `POST /api/admin/registrations/{registrationId}/reject`

Reject without provisioning (FR-005, FR-008).

**Responses**:
- `200 OK` → `{ "status": "rejected" }`
- `404 Not Found` → no such registration
- `409 Conflict` → registration is not in `pending` status

## Participant portal (Cognito-authenticated) — User Story 3

### `GET /api/portal/credentials`

Returns the authenticated participant's private key on first call only, and setup instructions on every call (FR-010, FR-011, FR-012, FR-013).

**Authorization**: Caller's Cognito identity email MUST match an approved `Participant.email`; otherwise `403 Forbidden` (covers the Edge Case of a pending/rejected visitor attempting sign-in — FR-013).

**Responses**:
- `200 OK`, first call → `{ "privateKey": string, "setupInstructions": string }`
- `200 OK`, subsequent calls → `{ "privateKey": null, "setupInstructions": string }`
- `403 Forbidden` → caller has no approved `Participant` record
