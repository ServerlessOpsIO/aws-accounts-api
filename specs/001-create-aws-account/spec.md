# Feature Specification: Create AWS Account and Track Progress

**Feature Branch**: `copilot/create-aws-account-api`

**Created**: 2026-10-08

**Status**: Draft

**Input**: User description: "As a consumer of this API I can use it to create a new AWS account in the ServerlessOps environment. By posting data to this API the API will initiate the process of creating an account. After the request is made the consumer should be able to check the progress of the initial transaction. The user should be able to see if the transaction has succeeded or failed, or if the transaction is still in progress, see what steps have completed and what steps still remain."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Request a New Account (Priority: P1)

As an authorized consumer, I can submit account details and receive confirmation that an account-creation transaction has been accepted, without waiting for the account to finish provisioning.

**Why this priority**: Starting actual account creation is the core value; a request acknowledgement alone must not be mistaken for a created account.

**Independent Test**: Submit a valid request in a controlled ServerlessOps environment and verify that it initiates creation of exactly one account and returns a transaction reference before provisioning finishes.

**Acceptance Scenarios**:

1. **Given** an authorized consumer and valid account details, **When** the consumer posts a creation request, **Then** the system accepts the transaction, returns a unique transaction reference and a way to check it through this API, and initiates account creation asynchronously.
2. **Given** missing or invalid required details, **When** the consumer submits a request, **Then** the system rejects it with field-specific guidance and does not initiate account creation.
3. **Given** a consumer without creation permission, **When** the consumer submits a request, **Then** the system rejects it without initiating a transaction.
4. **Given** an accepted request, **When** the consumer repeats the same request with the same request identity, **Then** the system returns the original transaction reference without creating a second account.
5. **Given** an existing request identity, **When** it is reused with different account details, **Then** the system reports a conflict without starting another transaction.

---

### User Story 2 - Inspect In-Progress Work (Priority: P2)

As an authorized consumer, I can check my transaction through this API and see which account-creation steps have completed, which are active, and which still remain.

**Why this priority**: Account creation takes time, and consumers need visibility without depending on operators or direct access to internal administration tools.

**Independent Test**: Use a controlled in-progress transaction with known completed and pending steps, retrieve its progress, and compare the result with the known state.

**Acceptance Scenarios**:

1. **Given** a newly accepted transaction, **When** the consumer checks its progress, **Then** the system reports it as in progress and lists every planned step, including steps not yet started.
2. **Given** a transaction that has completed some work, **When** the consumer checks its progress, **Then** the response distinguishes completed, active, and not-started steps using stable, understandable names.
3. **Given** a temporary provisioning issue undergoing recovery, **When** the consumer checks progress, **Then** the transaction remains in progress and the affected step is not reported as completed or terminally failed.
4. **Given** a transaction reference that does not exist, **When** an authorized consumer checks it, **Then** the system reports that no accessible transaction was found.
5. **Given** a consumer without permission to view a transaction, **When** the consumer checks it, **Then** the system does not disclose its existence, account details, or progress.

---

### User Story 3 - Understand the Final Outcome (Priority: P3)

As an authorized consumer, I can distinguish a successfully created account from a failed transaction and obtain enough information to use the account reference or seek help with a failure.

**Why this priority**: A terminal outcome closes the transaction and prevents consumers from treating incomplete work as success or polling indefinitely after failure.

**Independent Test**: Retrieve one known successful transaction and one known failed transaction and verify their outcomes, step histories, and result or failure details.

**Acceptance Scenarios**:

1. **Given** all required provisioning steps completed successfully, **When** the consumer checks the transaction, **Then** it reports success, all required steps as completed, the AWS account identifier, and the completion time.
2. **Given** a provisioning step has failed without further recovery, **When** the consumer checks the transaction, **Then** it reports failure, identifies the failed step with a safe explanation, preserves completed steps, and labels unexecuted steps as not started.
3. **Given** account creation succeeded but a later required step failed, **When** the consumer checks the transaction, **Then** it reports failure and includes the known account identifier so the partial result is not mistaken for no account having been created.
4. **Given** a transaction has timed out or been stopped before completion, **When** the consumer checks it, **Then** it reports failure with the corresponding reason and the last known step states.
5. **Given** a terminal transaction within the retention period, **When** the consumer checks it repeatedly, **Then** its terminal outcome and completed steps do not revert to an in-progress state.

### Edge Cases

- An account email is already in use, or concurrent requests target the same email: reject or fail the conflicting transaction with a clear explanation, without provisioning a duplicate account.
- The consumer loses the acceptance response: retrying with the same request identity retrieves the original transaction.
- A provisioning dependency is temporarily unavailable or throttled: recover where appropriate without repeating account-creating side effects; expose failure when recovery is exhausted.
- The system cannot accept a request: return an explicit error rather than a transaction reference that cannot be queried.
- An immediately queried accepted transaction has not begun its first step: expose it as in progress with all steps not started.
- A progress snapshot is read during a step transition: return a coherent snapshot, not success with incomplete required steps.
- A dependency outcome is uncertain: do not claim success or start replacement account creation; expose the uncertainty in the active step until resolved or terminally failed.
- A failure occurs after an account exists: preserve its identifier and explain the incomplete work; automatic account deletion is outside this feature.
- A progress check fails temporarily: distinguish inability to retrieve progress from failure of the account-creation transaction itself.
- A reference is malformed or outside the retention period: return a safe invalid-reference or no-accessible-transaction response without exposing internal details.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The system MUST allow consumers with creation permission to submit account-creation details by posting to this API and initiate creation of a real AWS account in the ServerlessOps environment, rather than completing a placeholder transaction.
- **FR-002**: The system MUST validate required account details before acceptance, including a non-empty account name, a valid account email address, and any additional details required by the environment's configured provisioning process that do not have configured defaults. It MUST explain missing or invalid fields and disallowed destinations without starting provisioning.
- **FR-003**: The system MUST acknowledge accepted work without waiting for provisioning to finish, providing a unique transaction reference and a consumer-accessible means of retrieving progress through this API.
- **FR-004**: The system MUST accept a consumer-supplied request identity. Repeating the same identity and details within the retention period MUST return the original transaction; reusing that identity with different details MUST fail as a conflict. Request identities MUST be scoped to the authorized caller.
- **FR-005**: The system MUST prevent retries, recovery, and concurrent requests for the same account email from creating duplicate accounts.
- **FR-006**: Consumers with transaction-read permission MUST be able to retrieve a transaction's overall outcome as in progress, succeeded, or failed using its reference, without needing access to internal administration tools.
- **FR-007**: Each progress response MUST list the complete ordered set of required steps for that transaction using stable identifiers and user-understandable names, with each step marked not started, in progress, completed, or failed. The response MUST include the transaction start time and the snapshot observation time.
- **FR-008**: A step MUST be marked completed only after its intended result is confirmed. Temporary recoverable errors MUST leave the affected step and overall transaction in progress; retries MUST not erase previously completed steps.
- **FR-009**: The system MUST report overall success only when the account is confirmed to exist in the intended ServerlessOps destination and every required provisioning step has completed. Success MUST include the AWS account identifier and transaction completion time.
- **FR-010**: Unrecoverable errors, exhausted recovery, timeouts, and stopped work MUST produce a failed transaction with a safe reason, the failed or interrupted step where known, completion time, preserved completed steps, and explicit not-started states for remaining unexecuted steps.
- **FR-011**: If an account identifier is known before a later failure, the failed transaction MUST expose that identifier to authorized readers and clearly identify the incomplete work.
- **FR-012**: Creation and progress access MUST use the existing authorization model. Only consumers granted permission for the requested operation and transaction MUST receive account or transaction details; denied progress lookups MUST not disclose whether the transaction exists.
- **FR-013**: Malformed, unknown, or expired references and temporary progress-retrieval failures MUST produce clear, safe errors. A lookup failure MUST NOT be represented as a provisioning failure.
- **FR-014**: Transaction progress, outcomes, and duplicate-request protection MUST remain available throughout execution and for at least 30 days after termination. Terminal outcomes MUST remain stable throughout that period.
- **FR-015**: Every provisioning error MUST be recorded with enough transaction and step context for operators to diagnose it. Consumer-visible responses and diagnostic records MUST NOT expose credentials or sensitive provisioning payloads.

### Key Entities *(include if feature involves data)*

- **Account Creation Request**: Consumer intent to create an account; includes request identity, requesting caller, account name, account email, and environment-required provisioning details or applicable defaults.
- **Account Creation Transaction**: Accepted work associated with one request; includes transaction reference, access permissions, overall outcome, start and completion times, planned steps, account identifier when known, and safe failure details.
- **Provisioning Step**: A named unit of required account-creation work within a transaction; includes stable identity, position in the required sequence, current state, and safe failure or recovery context.
- **AWS Account**: The created account associated with the transaction; includes its AWS account identifier and intended ServerlessOps destination.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: In controlled acceptance testing, at least 95% of valid requests return acceptance and a usable transaction reference within 5 seconds, without waiting for account provisioning.
- **SC-002**: In every tested in-progress snapshot, consumers can identify all required steps and distinguish completed work, active work, and work not yet started from a single progress check.
- **SC-003**: In controlled acceptance testing, at least 95% of confirmed step or terminal-outcome changes become visible to consumers within 30 seconds.
- **SC-004**: Every tested successful transaction identifies exactly one real account in the intended environment with all required steps completed; every tested terminal failure includes a safe reason and the last known step states.
- **SC-005**: Repeated submission of the same request identity, concurrent conflicting submissions, and recovery from transient failures create zero duplicate accounts in all acceptance scenarios.
- **SC-006**: Every tested unauthorized request is denied without disclosing transaction or account details, and every tested invalid creation request is rejected before provisioning begins.
- **SC-007**: Both successful and failed transactions remain retrievable with stable terminal outcomes for at least 30 days after termination.

## Assumptions

- Consumers are existing authorized ServerlessOps clients; new user registration, credential issuance, and a user interface are outside scope.
- The existing ServerlessOps account provisioning product is the source of required account details and required provisioning steps. Account name and email are always supplied; destination and any additional required details may be supplied by consumers or configured defaults. The allowed destinations and mandatory fields must be documented for consumers before release.
- ServerlessOps operators provide a working provisioning environment, permitted destinations, provisioning access, and sufficient account quota. The existing provisioning process supplies the mandatory environment enrollment and baseline configuration; new optional configuration products are outside scope.
- Success means account creation and completion of the existing environment's mandatory provisioning steps, not deployment of application workloads.
- Consumers poll for progress; notifications, cancellation requests, manual resume operations, account deletion, and account updates are outside scope.
- A 30-day post-termination retention period and the 5-second acceptance and 30-second progress-visibility targets are proposed defaults, not claims about current system capabilities. Total provisioning duration depends on external account provisioning and has no fixed completion-time guarantee.
- Failed provisioning does not automatically delete an account already created. Operators handle remediation using the reported partial result and diagnostic context.
