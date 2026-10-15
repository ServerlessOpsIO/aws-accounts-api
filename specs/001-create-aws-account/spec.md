# Feature Specification: Create AWS Account and Track Progress

**Feature Branch**: `copilot/specify-create-aws-account-api`

**Created**: 2026-10-15

**Status**: Draft

**Input**: User description: "As a consumer of this API I can use it to create a new AWS account in the ServerlessOps environment. By posting data to this API the API will initiate the process of creating an account. After the request is made the consumer should be able to check the progress of the initial transaction. The user should be able to see if the transaction has succeeded or failed, or if the transaction is still in progress, see what steps have completed and what steps still remain."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Request a New Account (Priority: P1)

An authorized API consumer submits the information needed for a new AWS account in the ServerlessOps environment and receives confirmation that creation has been accepted, without waiting for provisioning to finish.

**Why this priority**: Initiating account creation is the core capability; acceptance must not be confused with successful account creation.

**Independent Test**: Submit valid account details and verify that one provisioning transaction starts and a reference for subsequent status checks is returned before provisioning finishes.

**Acceptance Scenarios**:

1. **Given** an authorized consumer and valid account details, **When** the consumer posts an account creation request, **Then** the system accepts the request, initiates provisioning, and returns a unique transaction reference and acceptance time.
2. **Given** missing or invalid required information, **When** the consumer submits a request, **Then** the system identifies the invalid fields and does not initiate provisioning.
3. **Given** a consumer without account creation permission, **When** the consumer submits a request, **Then** the system denies the request and creates no account or transaction.
4. **Given** a request that cannot be accepted because the provisioning service is unavailable, **When** the consumer submits it, **Then** the system reports that acceptance failed rather than claiming that account creation started.
5. **Given** a previously accepted request, **When** the consumer retries that same request with the same consumer-supplied request identifier, **Then** the system returns the original transaction reference without initiating another account creation.
6. **Given** an identifier already used for a different request, **When** the consumer submits changed account details with that identifier, **Then** the system reports a conflict and initiates no new provisioning.

---

### User Story 2 - Inspect Provisioning Progress (Priority: P2)

An authorized consumer uses the transaction reference to check progress through the API and distinguish completed work, work currently underway, and work that remains.

**Why this priority**: Account provisioning is asynchronous; consumers need visibility without requiring access to infrastructure administration tools.

**Independent Test**: Query representative transactions at acceptance and at intermediate stages and verify that the reported overall status and step states match their known progress.

**Acceptance Scenarios**:

1. **Given** a newly accepted transaction, **When** the consumer checks its progress, **Then** the system reports it as in progress and lists its applicable steps, including those not yet started.
2. **Given** a partially completed transaction, **When** the consumer checks its progress, **Then** completed steps, any active steps, and remaining steps are individually identifiable in execution order, with dependencies identified where steps can run concurrently.
3. **Given** a transaction waiting on an external provisioning operation, **When** the consumer checks its progress, **Then** the transaction remains in progress and the waiting step is identified without falsely marking it complete.
4. **Given** an unknown transaction reference, **When** the consumer checks it, **Then** the system reports that the transaction was not found rather than returning an empty or successful result.
5. **Given** a consumer without permission to inspect a transaction, **When** the consumer queries it, **Then** the system reveals neither its account information nor its progress.
6. **Given** a transient failure while reading progress, **When** the consumer queries a transaction, **Then** the system reports that progress is temporarily unavailable without changing the transaction's actual outcome.

---

### User Story 3 - Determine the Final Outcome (Priority: P2)

An authorized consumer checks a finished transaction to determine whether the new account was created successfully or why provisioning failed.

**Why this priority**: Consumers must know whether an account is available and what action is needed after a failure.

**Independent Test**: Query one successful transaction and one failed transaction and verify their outcomes, account identifiers where available, and step-level results.

**Acceptance Scenarios**:

1. **Given** all required account provisioning steps have completed successfully, **When** the consumer checks the transaction, **Then** the system reports success, completion time, and the new AWS account identifier, with all required steps marked complete.
2. **Given** a step has failed permanently or the transaction has exceeded its allowed completion window, **When** the consumer checks the transaction, **Then** the system reports failure, completion time, the failed step, and a consumer-safe reason.
3. **Given** provisioning failed after some steps completed, **When** the consumer checks the transaction, **Then** completed work remains visible, blocked steps are distinguished from completed work, and an account identifier is included if an account was already created.
4. **Given** a terminal transaction, **When** the consumer repeats a status check during its retention period, **Then** the final outcome remains unchanged and the check does not initiate additional provisioning.

### Edge Cases

- Duplicate submissions, request retries after a lost acceptance response, and internal provisioning retries must not create multiple accounts for the same request identifier.
- An account email already in use or an organizational quota or policy restriction must produce a validation error when detectable before acceptance, or a clear terminal transaction failure otherwise.
- Provisioning may create an AWS account before a later enrollment step fails; the transaction must report the partial result rather than imply the account does not exist.
- Immediately checking a newly accepted transaction must return its initial progress, not an unexplained missing transaction.
- A transient external error must not be presented as a permanent failure while recovery is still underway.
- A permanently stalled or externally stopped transaction must become failed rather than remain in progress indefinitely.
- Conditional steps that do not apply must be distinguishable from unfinished or failed work.
- Malformed, unknown, expired, and unauthorized references must never expose another transaction's details.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The system MUST allow consumers with account creation permission to initiate creation of a new AWS account in the ServerlessOps environment by posting account details.
- **FR-002**: The system MUST validate a non-empty account name, a syntactically valid account email, and any required organization-specific provisioning inputs not supplied by approved environment defaults. Invalid requests MUST receive field-specific errors without starting provisioning.
- **FR-003**: An accepted request MUST initiate an asynchronous provisioning transaction and return its unique reference, acceptance time, and enough information to check progress through this API. Acceptance MUST NOT be represented as completed account creation.
- **FR-004**: The system MUST support a consumer-supplied request identifier. Repeating the same request with the same identifier within the transaction retention period MUST return the original transaction; reusing it with different details MUST report a conflict. Internal retries MUST NOT repeat account-creating side effects.
- **FR-005**: Authorized consumers MUST be able to retrieve a transaction's overall status using its reference through this API without separate access to AWS administration tools.
- **FR-006**: Every accepted transaction MUST expose exactly one overall status: in progress, succeeded, or failed. Accepted, queued, waiting, and retrying work MUST be classified as in progress; unrecoverable errors, timeout, or external termination MUST be classified as failed.
- **FR-007**: Progress MUST include a stable identifier, consumer-readable name, execution order or dependencies, and status for every applicable provisioning step. Step statuses MUST distinguish not started, in progress, completed, failed, skipped, and blocked; blocked means a prerequisite failed, and skipped means the step does not apply.
- **FR-008**: Progress MUST identify completed steps and remaining work, including waiting or retrying work, without treating a step as completed before its intended result is confirmed. Steps MUST include start and finish times when those events have occurred.
- **FR-009**: The system MUST report success only when the account exists in the intended ServerlessOps environment and all required provisioning steps have succeeded. A successful result MUST include the AWS account identifier and transaction completion time.
- **FR-010**: A failed result MUST identify the failed step or transaction-level failure, a consumer-safe reason, completion time, and any partial result, including an AWS account identifier when known. Steps prevented by that failure MUST be marked blocked.
- **FR-011**: The system MUST recover from temporary provisioning errors where safe and transition unrecoverable, timed-out, or externally terminated transactions to failure. Failure details MUST remain associated with the original transaction.
- **FR-012**: Progress checks MUST NOT alter provisioning or create additional accounts. Terminal outcomes MUST remain stable and queryable for at least 90 days after completion.
- **FR-013**: Creation and status retrieval MUST enforce the service's authorization policies. Invalid, unknown, expired, and inaccessible references MUST produce consumer-safe errors without exposing transaction details to unauthorized consumers.
- **FR-014**: If progress cannot be retrieved temporarily, the system MUST report a retrieval error rather than fabricate progress or change the transaction's outcome.
- **FR-015**: Every provisioning error, including recovered errors, MUST retain diagnostic information associated with its transaction and step for authorized operators. Consumer-visible results and diagnostic records MUST NOT disclose credentials or secrets.
- **FR-016**: Consumer-visible step descriptions MUST explain the business purpose of the work and remain consistent with the actual provisioning process.

### Key Entities *(include if feature involves data)*

- **Account Creation Request**: Requested account name, account email, any required environment-specific details, requesting consumer, and optional consumer-supplied request identifier.
- **Provisioning Transaction**: One accepted creation attempt, uniquely identified and associated with its request, overall status, acceptance and completion times, step progress, failure information, and resulting account identifier when available.
- **Provisioning Step**: An identifiable business activity within a transaction, with a descriptive name, order or dependencies, applicability, status, timestamps, and safe failure information where relevant.
- **AWS Account**: The resulting account identified by its AWS account identifier, name, email, and intended placement in the ServerlessOps environment.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Under normal operating conditions, at least 95% of valid submissions receive an acceptance confirmation and usable transaction reference within 5 seconds, without waiting for account provisioning to finish.
- **SC-002**: In acceptance testing, 100% of sampled in-progress transactions allow consumers to identify completed steps, active or waiting steps, and remaining steps using only this API.
- **SC-003**: Under normal operating conditions, at least 95% of progress checks return a result within 3 seconds, and confirmed step or outcome changes become visible within 30 seconds.
- **SC-004**: In acceptance testing, 100% of successful transactions identify the created account, and 100% of failed transactions identify the failure and any known partial account creation.
- **SC-005**: In retry and duplicate-submission tests, zero duplicate accounts are created for a single request identifier, including after a lost acceptance response.
- **SC-006**: All unauthorized creation and progress-access test cases are denied without exposing protected transaction details or initiating provisioning.
- **SC-007**: Consumers can retrieve unchanged final outcomes throughout the minimum 90-day retention period without requiring operator assistance.

## Assumptions

- Existing service authentication and authorization will be reused. This feature does not introduce user registration or a new permission-management model.
- Account creation uses the existing ServerlessOps account onboarding process, currently described as an AWS Control Tower Service Catalog product. Availability, permissions, capacity, and approved provisioning defaults for that process are dependencies.
- Account name and account email are consumer inputs. Organizational placement and any additional required onboarding information may be supplied by approved environment configuration; missing required information must be identified before acceptance.
- Business milestones are validation of provisioning details, submission for account provisioning, waiting for account creation and enrollment, and confirmation of readiness. Planning must align their exact names and conditional steps with the existing onboarding process.
- Success means the account is created and enrolled according to that onboarding process, not that application workloads have been deployed.
- The initial scope excludes account deletion, account modification, consumer-requested cancellation, automatic rollback of created accounts, and consumer-triggered restart of a failed transaction.
- The configured provisioning completion window follows the existing onboarding process's documented limit; planning must establish that limit before implementation. The responsiveness targets above do not impose an account creation duration on AWS.
- A minimum 90-day terminal-history retention period and the responsiveness targets above are proposed defaults for this specification.
- Consumers may poll for progress. Push notifications, a graphical interface, and separate AWS administrative access are not required.
