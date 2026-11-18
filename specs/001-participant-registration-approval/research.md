# Research: Participant Registration & Approval

**Feature**: [spec.md](./spec.md) | **Plan**: [plan.md](./plan.md)

No `[NEEDS CLARIFICATION]` markers remained after `/speckit-specify`. The decisions below record the technology and architecture choices made to fill in the Technical Context, each as a research task resolved with a reasonable default appropriate to the project's scale and the constitution's constraints.

## Decision 1: Application stack — Next.js (TypeScript) on Node.js 20

**Decision**: Build the public registration form, admin review UI, and participant portal as a single Next.js (App Router) application in TypeScript, using route handlers / server actions for the backend logic instead of a separate API service.

**Rationale**: The feature has three UI surfaces (public, admin, participant-post-auth) backed by a small number of state transitions (register → approve/reject → provision → sign-in → reveal). A single full-stack framework avoids standing up and deploying a separate backend service for what is, in scope, a modest workshop tool. TypeScript end-to-end lets the `Registration`/`Participant`/`Team` domain types be shared between UI and backend without duplication or a schema-translation layer.

**Alternatives considered**:
- Separate REST/GraphQL backend (e.g., Express or a Lambda-per-endpoint API) + a separate SPA frontend — rejected as unnecessary operational surface area for this scale; two deployables to keep in sync for no scale benefit.
- A no-code/low-code admin tool for the approval queue — rejected because the approval action must deterministically trigger the pluggable `ProvisioningService`, which is easier to guarantee in application code than to bolt onto a low-code tool.

## Decision 2: Data storage — Amazon DynamoDB

**Decision**: Store `Registration`, `Team`, and `ProvisioningRecord` data in DynamoDB tables rather than a relational database.

**Rationale**: No relational queries beyond simple lookups (by email/GitHub ID for duplicate checks, by team for roster views, by participant for the portal) are required by any functional requirement. DynamoDB avoids a persistent database connection/credential that would otherwise need to be provisioned and kept secret per deployment (Principle I), and its per-deployment table creation via CDK keeps each deployment's data fully isolated (Principle III) with no shared connection string to leak.

**Alternatives considered**:
- Amazon RDS (Postgres) — rejected: adds a shared-credential surface (connection string, security group for DB access) that every independent deployment (including participants' own forks) would need to stand up and protect; no requirement here needs relational joins.
- Plain JSON files / SQLite for a "simple" stub-only deployment — rejected as a special case that would diverge from the production storage model, making the stub less representative of real behavior.

## Decision 3: Provisioning interface and stub safety — structural isolation, not configuration

**Decision**: Define a single `ProvisioningService` interface (documented in `contracts/provisioning-interface.md`) with exactly two implementations: `AwsProvisioningService` (imports `@aws-sdk/client-ec2` and `@aws-sdk/client-ssm`, generates and installs the real SSH keypair, mutates the real security group) and `StubProvisioningService` (imports neither AWS SDK client; generates a syntactically valid but inert keypair in-process and records a simulated `ProvisioningRecord`). The application selects an implementation at startup based on an explicit `DEPLOYMENT_ENV=production` flag, defaulting to the stub for any other or missing value.

**Rationale**: Constitution Principle II requires the stub be unable to reach real infrastructure "by construction, not by configuration." Making the real AWS clients a compile-time import that only `aws-provisioning.ts` contains means a misconfigured environment variable can, at worst, select the stub when production was intended (a visible functional failure — nothing gets provisioned) but can never cause the stub path to accidentally mutate real infrastructure, because the code capable of doing so is simply absent from that module's dependency graph.

**Alternatives considered**:
- A single implementation with an `if (isProduction)` branch before each AWS call — rejected: this is exactly the "safety by configuration" the constitution rules out, since a flipped flag or a copy-pasted code path could reach real infrastructure from a non-production deployment.
- Feature-flagging via a remote config service — rejected as unnecessary complexity and an extra runtime dependency for a decision that is static per deployment.

## Decision 4: Deployment isolation between production and every other deployment

**Decision**: Each deployment (production, and any participant's forked/starter deployment) is its own independent CDK stack instance with its own Cognito user pool, its own DynamoDB tables, and its own IAM roles. No deployment reads another's configuration, secrets, or data.

**Rationale**: Directly satisfies Principle III — a bug in a participant's in-progress fork has no path to the production approval/provisioning flow because there is no shared credential, database, or deployment target to reach through. Combined with Decision 3, a non-production deployment is doubly isolated: it defaults to the stub, and even if it didn't, it has no credentials for production's AWS resources.

**Alternatives considered**:
- A shared multi-tenant deployment with per-workshop-run data partitioning — rejected: this would require careful tenant-isolation logic in application code (a much larger attack surface) to achieve what separate stacks give for free.

## Decision 5: Testing approach

**Decision**: Vitest for unit tests and a shared contract-test suite (same test cases run against both `AwsProvisioningService`, mocked at the AWS SDK boundary, and `StubProvisioningService`) to guarantee both implementations satisfy the same interface behaviorally. Playwright drives the three end-to-end user stories from `spec.md` against a stub-backed deployment.

**Rationale**: Running identical contract tests against both implementations is the practical way to verify the interface in Decision 3 is honored by both sides without ever pointing an automated test at real AWS infrastructure.

**Alternatives considered**:
- Testing only the stub — rejected: would never catch a drift between the stub's and the real implementation's contract (e.g., stub returns success for an action the real implementation can fail).
- Integration-testing the real implementation against live AWS resources in CI — rejected: contradicts the project's own goal of never requiring real infrastructure access outside the single production deployment, and would be slow/costly/flaky for CI.

## Outstanding unknowns

None. All Technical Context fields are resolved; Phase 1 design may proceed.
