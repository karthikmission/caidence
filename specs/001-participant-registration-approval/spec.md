# Feature Specification: Participant Registration & Approval

**Feature Branch**: `001-participant-registration-approval`

**Created**: 2026-11-18

**Status**: Draft

**Input**: User description: "Caidence: a public registration-and-approval web app for workshop participants. A visitor registers with their email and GitHub ID. Registration is public, but an administrator must approve it before anything is provisioned. On approval, the system: generates a unique SSH keypair for that participant, creates their Linux login on their assigned team's EC2 instance, and admits their current public IP to that team's security group. The participant then signs in via Amazon Cognito; once authenticated, their private key is shown exactly once, alongside setup instructions customized to their assigned team and role. The step that actually touches real infrastructure (creating the OS login, mutating the security group, issuing the key) must sit behind a pluggable interface, per the project constitution — production uses the real implementation, every other deployment uses a safe stub that cannot reach real infrastructure."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Visitor registers for the workshop (Priority: P1)

A visitor who wants to participate in the workshop submits their email address and GitHub ID through a public registration form, with no account or sign-in required to do so.

**Why this priority**: Registration is the entry point for every other capability in this feature. Nothing else can be tested or delivered without it.

**Independent Test**: Can be fully tested by submitting the public registration form with a valid email and GitHub ID and confirming the submission is recorded with a "pending" status and the visitor receives a confirmation that their request is awaiting approval.

**Acceptance Scenarios**:

1. **Given** the public registration page, **When** a visitor submits a valid email address and GitHub ID, **Then** the system records the registration as pending review and confirms submission to the visitor.
2. **Given** a visitor who already has a pending or approved registration under the same email or GitHub ID, **When** they attempt to register again, **Then** the system rejects the duplicate submission and informs them a registration already exists.
3. **Given** the public registration page, **When** a visitor submits an incomplete or malformed email or GitHub ID, **Then** the system rejects the submission and explains what needs to be corrected.

---

### User Story 2 - Administrator reviews and approves a registration (Priority: P2)

An administrator reviews pending registrations, assigns each one to a team and role, and approves or rejects it. Approving a registration automatically triggers provisioning of that participant's access: a unique SSH keypair, a Linux login on their assigned team's EC2 instance, and their public IP being admitted to that team's security group.

**Why this priority**: This is the gate that turns a public, unauthenticated request into real (or stubbed) provisioned access. It is the core trust boundary of the feature and the direct trigger for every infrastructure-touching action.

**Independent Test**: Can be fully tested by having an administrator open the pending-registrations queue, assign a team and role to one entry, and approve it — then confirming the provisioning actions (keypair generated, login created, IP admitted) were invoked exactly once, and that the registration's status moves to "approved."

**Acceptance Scenarios**:

1. **Given** a pending registration, **When** an administrator assigns a team and role and approves it, **Then** the system provisions a unique SSH keypair, creates the participant's Linux login on the assigned team's EC2 instance, and admits the participant's registered public IP to that team's security group, and marks the registration "approved."
2. **Given** a pending registration, **When** an administrator rejects it, **Then** no provisioning action occurs, the registration is marked "rejected," and the visitor is informed they were not approved.
3. **Given** an approved registration, **When** an administrator views it again, **Then** the system does not re-run any provisioning action.
4. **Given** a non-production deployment, **When** an administrator approves a registration, **Then** the provisioning actions are performed by a safe stub that cannot reach or alter any real infrastructure.

---

### User Story 3 - Approved participant signs in and retrieves their credentials (Priority: P3)

Once approved, a participant signs in through Amazon Cognito. On their first authenticated visit after approval, they see their private key exactly once, together with setup instructions tailored to their assigned team and role.

**Why this priority**: This is how the provisioned access actually reaches the participant. It depends on both prior stories having completed successfully, so it is tested last even though it delivers the feature's end-user value.

**Independent Test**: Can be fully tested by signing in as an approved participant and confirming the private key and team/role-specific setup instructions are displayed exactly once, and that a subsequent sign-in no longer displays the private key.

**Acceptance Scenarios**:

1. **Given** an approved participant who has not yet viewed their credentials, **When** they sign in via Amazon Cognito, **Then** the system displays their private key and setup instructions specific to their assigned team and role.
2. **Given** a participant who has already viewed their private key once, **When** they sign in again, **Then** the system no longer displays the private key, while still showing their setup instructions.
3. **Given** a visitor whose registration is still pending or was rejected, **When** they attempt to sign in, **Then** the system does not grant them access to any credentials or setup instructions.

---

### Edge Cases

- A visitor submits the same email with a different GitHub ID, or the same GitHub ID with a different email — is this treated as a duplicate or a second distinct registration?
- A participant's public IP changes between the time they registered and the time they try to connect over SSH, so the admitted IP no longer matches.
- An administrator approves a registration but the underlying provisioning action partially fails (e.g., the Linux login is created but the security group update fails).
- An administrator rejects a registration — can that visitor submit a new registration later, and does the system distinguish a fresh attempt from re-submission by a previously rejected visitor?
- A participant closes the page or loses connectivity during the one-time key reveal before copying it down.
- A team's EC2 instance or security group does not exist or is unreachable at the moment an administrator approves a registration assigned to that team.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The system MUST allow any visitor to submit a registration consisting of an email address and a GitHub ID, without requiring the visitor to sign in or create an account first.
- **FR-002**: The system MUST reject registration submissions with a missing or malformed email address or GitHub ID, and MUST reject a submission that duplicates an existing pending or approved registration's email or GitHub ID.
- **FR-003**: The system MUST hold every new registration in a "pending" status until an administrator makes a decision, and MUST NOT provision any access while a registration is pending.
- **FR-004**: The system MUST provide administrators a view of pending registrations and the ability to assign a team and a role to a registration as part of approving it.
- **FR-005**: The system MUST allow an administrator to approve or reject a pending registration, and MUST record the resulting status ("approved" or "rejected") against that registration.
- **FR-006**: The system MUST, upon approval of a registration, automatically and exactly once: generate a unique SSH keypair for the participant, create a Linux login for that participant on their assigned team's EC2 instance, and admit the participant's public IP (captured at registration) to that team's security group.
- **FR-007**: The system MUST NOT re-run any provisioning action for a registration that has already been approved.
- **FR-008**: The system MUST NOT perform any provisioning action for a registration that is rejected or still pending.
- **FR-009**: The system MUST perform the infrastructure-touching provisioning actions (OS login creation, security group mutation, SSH key issuance) through a capability that can be swapped between a real implementation and a safe stub, and MUST use the safe stub — which cannot reach or alter real infrastructure — in every deployment that is not the designated production deployment.
- **FR-010**: The system MUST require an approved participant to authenticate via Amazon Cognito before they can view anything related to their provisioned credentials or setup instructions.
- **FR-011**: The system MUST display a participant's private key at most once across the lifetime of their registration, and MUST NOT make that private key retrievable through the application after its first display.
- **FR-012**: The system MUST display setup instructions to an authenticated, approved participant that are customized to that participant's assigned team and role.
- **FR-013**: The system MUST prevent a visitor whose registration is pending or rejected from accessing any provisioned credentials, setup instructions, or team-specific information.
- **FR-014**: The system MUST retain an auditable record of each registration's status changes and the outcome of its provisioning actions (success or failure per action).

### Key Entities

- **Registration**: A visitor's request to join the workshop — email address, GitHub ID, public IP captured at submission time, status (pending, approved, rejected), and timestamps for submission and decision.
- **Participant**: The identity that exists once a registration is approved — linked back to its registration, plus an assigned team, an assigned role, and the state of its one-time private-key reveal.
- **Team**: A workshop team that participants are assigned to — associated with one EC2 instance and one security group that participants in that team are provisioned against.
- **SSH Keypair**: The unique key pair generated for a participant on approval — a private key shown to the participant at most once, and whatever reference the system needs to track that it was issued.
- **Provisioning Record**: An audit entry describing what a provisioning action (OS login creation, security group mutation, key issuance) attempted to do and whether it succeeded, for a given participant.
- **Administrator**: The role responsible for reviewing pending registrations, assigning team and role, and deciding approval or rejection.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A visitor can complete registration in under 2 minutes.
- **SC-002**: An administrator can review and decide on a pending registration, including assigning team and role, in under 1 minute.
- **SC-003**: 100% of approved registrations result in a participant who can retrieve working credentials and team/role-specific setup instructions, with no manual steps by the administrator beyond the approval decision itself.
- **SC-004**: 0% of private keys are retrievable through the application a second time after their first display, verified across repeated sign-ins.
- **SC-005**: 0 instances of a non-production deployment successfully mutating real infrastructure, verified by deployment configuration audit.
- **SC-006**: At least 95% of approved participants successfully connect to their assigned environment using only the displayed setup instructions, without contacting an administrator for help.

## Assumptions

- Teams and their underlying EC2 instance and security group already exist by the time an administrator approves a registration into that team; this feature does not cover creating or configuring teams' infrastructure itself.
- An administrator assigns each participant's team and role at the point of approval, rather than the visitor selecting these during registration.
- The public IP admitted to a team's security group is the IP address captured from the visitor's registration submission, since that is the system's most recent and only known IP for that participant at the moment of approval; re-admitting a different IP after a network change is out of scope for this feature.
- A rejected visitor may submit a new registration later; the system treats it as a new, independent registration rather than reopening the rejected one.
- The GitHub ID collected at registration is used for identification and to derive the participant's Linux username, not to pull any data from GitHub itself.
- "Administrator" is a fixed, pre-existing role in this system; this feature does not cover how administrators themselves are created or authenticated.
- Email notifications to visitors/participants about status changes (pending confirmation, approval, rejection) follow standard practice but their exact content and delivery channel are an implementation detail, not specified further here.
