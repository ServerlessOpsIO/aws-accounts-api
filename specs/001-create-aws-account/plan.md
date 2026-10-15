# Control Tower account provisioning

## Architecture

Preserve the existing SAM infrastructure, JSONata ASL workflow, Cognito authorization, and direct API Gateway → Step Functions integration. Provision accounts exclusively through **AWS Control Tower Account Factory in Service Catalog**, never Organizations `CreateAccount`.

## 1. Align the API contract and specification

- Keep the existing `POST /v1/account` endpoint.
- Require non-empty `account_name` and `ou_name`.
- Return **201** with `txn_id` only after Step Functions accepts the execution. Document that acceptance does not mean provisioning succeeded.
- Add **GET /transactions?txn_id=…** for transaction status and progress.
- Define field-specific validation errors, unknown transaction responses, authorization failures, conflicts, throttling, and temporary service failures.
- Reconcile the existing specification: it currently treats account email as consumer input and additionally requires acceptance timestamps, idempotency, and 90-day terminal-history retention. Preserve those requirements unless explicitly superseded; expose acceptance time through transaction details without expanding the requested POST response.

## 2. Establish Account Factory prerequisites

Before implementation, confirm:

- Control Tower home Region in management account `349603509961`.
- Account Factory product, approved provisioning artifact, launch path, portfolio access, and launch-role configuration.
- Actual provisioning parameter names and required OU value format.
- Authoritative OU lookup and rejection rules for missing, ambiguous, or unsupported names.
- Provisioning completion deadline and supported evidence of successful enrollment.

Treat these as deployment/design gates rather than assuming resource identifiers or AWS behavior.

## 3. Configure provisioning inputs

Extend the SAM parameters and component's existing `cfn-parameters.json` with:

- `SSOUserFirstName`, initially Tom.
- `SSOUserLastName`, initially McLaughlin.
- `SSOUserEmail`, initially the supplied address.
- Management-account role ARN and required Account Factory configuration.

Derive `AccountEmail` as `admin+<account_name>@serverlessops.io`.

Validate account names against both Account Factory requirements and the derived email's constraints. Reject incompatible values instead of silently changing the requested name. Consumers must not override configured provisioning identities or privileged product settings.

## 4. Design cross-account access

- Define a separately deployed management-account IAM role trusted only by the service's workflow execution role in account `774305573766`.
- Use Step Functions task credentials to assume that role for supported cross-account AWS integrations.
- Grant narrowly scoped Service Catalog access and only necessary OU/account-read permissions.
- Verify portfolio entitlement, launch constraints, and any required `iam:PassRole` permissions.
- Keep management-account deployment separate from the application stack; document deployment order and trust configuration.

## 5. Replace the placeholder workflow

Define documented business milestones:

1. Validate and enrich provisioning details.
2. Resolve the destination OU and Account Factory configuration.
3. Submit Service Catalog provisioning.
4. Wait and poll the provisioning record.
5. Confirm account creation, intended OU placement, and required enrollment.
6. Persist the final outcome.

Use a stable Service Catalog provisioning token for retries. Apply bounded backoff, polling intervals, an explicit overall deadline, and catch paths. Do not resubmit account creation after an ambiguous timeout without reconciling the existing operation.

Preserve any known account ID when later steps fail; do not automatically delete partially created accounts.

## 6. Track transactions and progress

Use DynamoDB through native AWS integrations for durable transaction records containing:

- Transaction and execution identifiers, requesting consumer, and request details.
- Acceptance/completion timestamps and overall status.
- Ordered business steps, their statuses, and timestamps.
- Service Catalog operation identifiers, resulting account ID, and safe failure details.

Expose overall statuses `in_progress`, `succeeded`, and `failed`; distinguish pending, active, completed, failed, skipped, and blocked steps.

Design the read path to handle an immediate GET before the workflow's first database write. Add reconciliation for execution timeout or external termination, which workflow catch paths alone cannot reliably cover. Retain terminal records for at least 90 days after completion.

## 7. Resolve acceptance and idempotency mechanics

Before implementation, validate a native integration design that satisfies all existing specification requirements together:

- Stable transaction identity and consumer-scoped request identifiers.
- Repeated identical requests returning the original transaction.
- Conflicts for changed payloads using an existing identifier.
- Safe behavior after lost responses and completed executions.
- Immediate transaction lookup without an initialization race.

Do not rely on Step Functions execution-name deduplication alone for the full retention period. Any necessary departure from the direct-integration architecture requires explicit justification and approval.

## 8. Validate and document

During implementation:

- Run the existing `sam validate --lint` and `sam build`.
- Validate ASL definitions and cross-account integration support.
- Exercise acceptance, immediate polling, successful enrollment, provisioning failure, partial creation, duplicate requests, throttling, timeout, external termination, and unauthorized access.
- Verify diagnostic logging covers recovered and terminal errors without exposing secrets.
- Update API documentation and deployment instructions.

This document is a plan only; it does not implement the feature.
