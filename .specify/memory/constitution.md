# AWS Accounts API Constitution

## Core Principles

### I. State-Machine-First Design
New functionality MUST be implemented as additional AWS Step Functions states whenever
the workflow can meet its requirements that way. Lambda functions MUST NOT be introduced
when state machine steps and direct AWS service integrations suffice; any Lambda addition
must be justified by a requirement that those integrations cannot satisfy.

### II. Idempotent and Resilient Steps
Every state machine step that can cause a side effect MUST be idempotent, including when
an execution or request is retried. Each step MUST handle expected service and input
errors gracefully through appropriate retry, catch, or terminal-failure behavior, without
masking failures or causing unintended repeated side effects.

### III. Error Logging
Every workflow error MUST be logged with enough execution and state context to support
diagnosis. Logging MUST cover errors handled by catch paths as well as unhandled
execution failures, and MUST NOT expose credentials or other sensitive values.

### IV. Step Documentation
Each state MUST have clear, concise documentation describing its purpose and, where
relevant, its inputs, outputs, side effects, idempotency behavior, and error handling.
Documentation MUST be kept in sync with changes to the state.

### V. AWS-Native Architecture
The API and workflow infrastructure MUST be defined with AWS SAM, and workflows MUST be
expressed in Amazon States Language. API Gateway MUST start the state machine directly;
workflow steps MUST call AWS services directly where the required integration is
available and suitable.

## Architecture and AWS Constraints

The API Gateway endpoint is the entry point to the Step Functions workflow. SAM templates
are the source of truth for the API, state machine, and supporting AWS resources.
Workflow changes MUST preserve the API contract and the intended execution behavior.

## Development Workflow and Compliance

Changes to state machines MUST be reviewed for idempotency, error handling and logging,
step documentation, and consistency with the API contract. Reviews MUST verify that new
functionality uses native state machine capabilities where possible and includes a clear
justification for any Lambda function. Validation MUST cover affected workflow definitions
and behavior before deployment; failures and limitations MUST be recorded and addressed.

## Governance

This constitution governs design and review of the AWS Accounts API. Amendments MUST be
proposed through a reviewed change that describes the rationale and impact on existing
principles. Changes that remove or redefine a principle incompatibly require a MAJOR
version increment; adding a principle or materially expanding governance requires a
MINOR increment; clarifications and non-semantic refinements require a PATCH increment.
Every amendment MUST update the last-amended date. The ratification date is the original
adoption date and MUST remain unchanged.

Reviewers MUST assess proposed changes against these principles. Any exception MUST be
explicitly justified and approved in the change review. When this constitution conflicts
with implementation guidance, the constitution takes precedence.

**Version**: 1.0.0 | **Ratified**: TODO(RATIFICATION_DATE): Confirm original adoption date with maintainers. | **Last Amended**: 2026-10-08
