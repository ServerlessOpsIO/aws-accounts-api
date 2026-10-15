# Feature Specification: Create AWS Account and Track Progress

**Feature Branch**: `copilot/create-new-aws-account`

**Created**: 2026-10-15

**Status**: Draft

**Input**: User description: "As a consumer of this API I can use it to create a new AWS account in the ServerlessOps environment. By posting data to this API the API will initiate the process of creating an account. After the request is made the consumer should be able to check the progress of the initial transaction. The user should be able to see if the transaction has succeeded or failed, or if the transaction is still in progress, see what steps have completed and what steps still remain."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Request a New AWS Account (Priority: P1)

As an authorized API consumer, I can submit the information needed for a new AWS account and receive a transaction reference without waiting for account creation to finish.

**Why this priority**: Accepting a creation request is the entry point for delivering a new account.

**Independent Test**: Submit a valid account request and verify that it initiates one account-creation transaction and returns a reference while provisioning continues.

**Acceptance Scenarios**:

1. **Given** an authorized consumer and valid account details, **When** the consumer posts an account-creation request, **Then** the service accepts the request, initiates provisioning in the ServerlessOps environment, and returns a unique transaction reference and instructions for checking its status.
2. **Given** missing or invalid required account details, **When** the consumer submits the request, **Then** the service identifies the invalid fields and does not initiate provisioning.
3. **Given** a consumer without account-creation permission, **When** the consumer submits a request, **Then** the service rejects it without initiating provisioning.
4. **Given** an accepted request, **When** the consumer repeats it with the same retry reference and identical details, **Then** the service returns the original transaction reference without initiating another account creation.
5. **Given** a retry reference already associated with a request, **When** the consumer reuses it with different details, **Then** the service reports a conflict and does not change the original transaction.

---

### User Story 2 - Inspect In-Progress Steps (Priority: P1)

As an authorized API consumer, I can use a transaction reference to see whether provisioning is still underway, which steps have completed, and which steps remain.

**Why this priority**: Account creation is asynchronous; consumers need visibility without operator assistance.

**Independent Test**: Use an existing in-progress transaction with known completed and outstanding steps and verify that its status view accurately distinguishes them.

**Acceptance Scenarios**:

1. **Given** a newly accepted transaction, **When** the consumer checks its status, **Then** the service reports it as in progress and lists the applicable provisioning steps, including those not yet started.
2. **Given** a partially completed transaction, **When** the consumer checks its status, **Then** the service identifies completed steps, any active steps, and steps that have not started.
3. **Given** a temporary provisioning issue being retried, **When** the consumer checks its status, **Then** the transaction remains in progress and the affected step is identified as awaiting retry rather than complete.
4. **Given** a transaction that advances between checks, **When** the consumer checks again, **Then** the response reflects the newer progress within the defined visibility target and retains already completed steps.
5. **Given** an unknown transaction reference or a consumer without permission to view a transaction, **When** the consumer checks its status, **Then** the service returns an appropriate error without disclosing unauthorized transaction details.

---

### User Story 3 - Determine the Final Outcome (Priority: P1)

As an authorized API consumer, I can determine whether my account-creation transaction succeeded or failed and obtain either the resulting account identifier or an understandable failure explanation.

**Why this priority**: Consumers must distinguish acceptance from successful provisioning and know when intervention is needed.

**Independent Test**: Query known successful and failed transactions and verify their outcomes, account results, and step-level details.

**Acceptance Scenarios**:

1. **Given** all required provisioning steps have completed successfully, **When** the consumer checks the transaction, **Then** it reports success, the new AWS account identifier, and all required steps as completed.
2. **Given** provisioning cannot continue after its allowed retries or waiting period, **When** the consumer checks the transaction, **Then** it reports failure, the failed step, a safe explanation, completed steps, and steps not performed.
3. **Given** the account was created but a subsequent required step failed, **When** the consumer checks the transaction, **Then** it reports failure and identifies the existing account so the consumer does not assume no account was created.
4. **Given** a transaction has reached a terminal outcome, **When** the consumer queries it repeatedly, **Then** its outcome and recorded results remain available and unchanged during the retention period.

### Edge Cases

- A consumer loses the acceptance response and retries: the same retry reference must recover the original transaction without duplicate side effects.
- Simultaneous submissions with the same retry reference must resolve to a single transaction.
- A proposed account email is already associated with an AWS account: report a conflict if known before acceptance, otherwise record a terminal provisioning failure; do not silently adopt that account.
- Provisioning reaches an organizational quota, is denied permission, or is rejected by the account provider: report a failed transaction with a safe explanation.
- Provisioning is temporarily unavailable: keep accepted work trackable while retrying, and fail it explicitly when the allowed retry or waiting period is exhausted.
- A transaction is checked before its first provisioning step starts: show in-progress status and all planned steps.
- A required step fails after earlier steps caused side effects: preserve completed-step information, identify any created account, and mark remaining steps as not performed.
- Status retrieval is temporarily unavailable: report a retrieval error rather than falsely reporting provisioning failure.
- A transaction reference is malformed, unknown, expired, or unauthorized: return a clear error without exposing another consumer's information.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The service MUST allow authorized consumers to initiate new AWS account creation in the ServerlessOps environment by posting account details.
- **FR-002**: The service MUST validate a non-empty account name, a syntactically valid account email address, and any consumer-supplied organizational destination against the environment's permitted destinations before accepting a request. Missing required values or invalid values MUST produce field-specific errors without starting provisioning.
- **FR-003**: The service MUST acknowledge accepted requests without waiting for provisioning to finish, returning a unique transaction reference, acceptance time, and a documented way to query progress through this API.
- **FR-004**: The service MUST support a consumer-supplied retry reference. Repeated submissions by the same consumer with the same reference and identical details MUST resolve to the same transaction; conflicting details MUST be rejected. This guarantee MUST hold for concurrent submissions and throughout the transaction retention period.
- **FR-005**: The service MUST create the requested account within the configured ServerlessOps organization and complete its existing required account-provisioning process before reporting success.
- **FR-006**: Authorized consumers MUST be able to query a transaction using its reference and receive its overall status, acceptance time, last update time, completion time when terminal, and step-level progress.
- **FR-007**: Overall transaction status MUST distinguish `in progress`, `succeeded`, and `failed`. Accepted work that has not started and work waiting for a retry MUST remain `in progress`.
- **FR-008**: Each progress view MUST list all applicable business-level provisioning steps in their intended order, using stable step identifiers and understandable names. Each step MUST distinguish `not started`, `in progress`, `awaiting retry`, `completed`, `failed`, and `not performed` where applicable. Completed and failed steps MUST include their completion time.
- **FR-009**: The progress view MUST distinguish validating provisioning eligibility, requesting account creation, waiting for account creation, and confirming required environment enrollment. These are consumer-visible milestones, not a prescribed internal implementation; any additional required environment provisioning milestones MUST also be visible.
- **FR-010**: A transaction MUST report success only after every required step completes, and MUST include the resulting AWS account identifier.
- **FR-011**: A transaction MUST report failure when a required step cannot complete, including when a documented finite retry or waiting limit is exhausted. Its result MUST include the failed step, a consumer-understandable failure category and explanation, and any account identifier already obtained. Remaining steps that cannot run MUST be marked `not performed`.
- **FR-012**: Retries of requests or provisioning steps MUST NOT create duplicate accounts or repeat an already completed side effect. Rechecking status MUST NOT initiate provisioning or change its outcome.
- **FR-013**: Creation and status retrieval MUST enforce the service's existing authentication and authorization model. Status retrieval MUST be limited to consumers explicitly permitted to view that transaction.
- **FR-014**: Consumer-visible errors and progress MUST NOT disclose credentials, secrets, or internal diagnostic payloads. Provisioning errors MUST be recorded with transaction and step context for authorized operators without exposing sensitive values.
- **FR-015**: Progress and terminal outcomes MUST remain queryable while provisioning is active and for at least 90 days after completion. During this period terminal outcomes and completed-step records MUST not regress or disappear.
- **FR-016**: The service MUST distinguish invalid or unavailable transaction references, unauthorized access, and temporary status-retrieval failures from a failed provisioning outcome, without disclosing whether an unauthorized transaction exists.

### Key Entities *(include if feature involves data)*

- **Account Creation Request**: The consumer's requested account name, account email, optional permitted organizational destination, retry reference, and requesting consumer identity.
- **Provisioning Transaction**: One accepted request and its unique reference, overall status, timestamps, ordered step progress, resulting account identifier when known, and terminal failure information when applicable.
- **Provisioning Step**: A stable business milestone within a transaction, with its identifier, name, intended order, status, timestamps, and safe failure information where applicable.
- **AWS Account**: The newly created account identified by its AWS account identifier, requested name and email, and organizational placement in the ServerlessOps environment.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: At least 95% of valid submissions receive a trackable acceptance within 5 seconds under the environment's agreed normal operating load, regardless of total provisioning duration.
- **SC-002**: For every tested in-progress transaction, consumers can identify all completed, active or retrying, and remaining steps from a single status check without using separate administrative tools.
- **SC-003**: At least 95% of recorded provisioning changes become visible to consumers within 30 seconds.
- **SC-004**: Every tested successful transaction returns its new account identifier; every tested terminal failure identifies the failed milestone and a safe reason, including any already-created account.
- **SC-005**: Repeated and concurrent identical submissions with the same retry reference produce exactly one transaction and no more than one new account in all retry acceptance tests.
- **SC-006**: All tested unauthorized creation and status requests are rejected without starting provisioning or exposing protected transaction details.
- **SC-007**: Every tested terminal transaction remains queryable with the same outcome throughout the minimum 90-day retention period.

## Assumptions

- Existing service authentication and permission management are reused; introducing a new identity system is out of scope.
- The ServerlessOps environment already has an account-provisioning facility, a permitted organizational destination or configured default, and established enrollment requirements. This feature uses those requirements rather than defining a new organizational governance baseline.
- Consumers supply an account name and a unique account email. Environment-specific enrollment settings are derived from existing configuration unless explicitly exposed as validated request fields.
- Account creation and required environment enrollment define success. Workload deployment, billing changes, account deletion, cancellation, manual retry of terminal failures, notifications, and a user interface are out of scope.
- The consumer-visible milestones in FR-009 establish the minimum progress vocabulary; planning must map the actual environment's provisioning requirements to these milestones.
- The 5-second acceptance target, 30-second progress visibility target, and 90-day terminal retention are initial service expectations, not promises about how quickly AWS completes account creation.
- Provisioning depends on available organizational capacity, valid environment configuration, and permission to create and enroll accounts. Finite retry and waiting limits must be documented during planning, based on those provider constraints.
- Automatic deletion of a partially provisioned account is not assumed; partial results remain visible for operator recovery.
