# Implementation Plan: Control Tower Account Creation

**Branch**: `copilot/create-account-via-control-tower-again` | **Date**: 2026-10-15 | **Spec**: [spec.md](spec.md)

**Scope**: Planning only. No application code, deployed API definition,
infrastructure, component parameters, or existing specification is changed.
No implementation or `tasks.md` is generated.

## Summary

Accept `POST /v1/account` with required `account_name` and `ou_name`, return HTTP
`201` with only `txn_id` in the body, and provide
`GET /transactions?txn_id=<uuid>` for authorized status/progress retrieval.
Provision the **AWS Control Tower Account Factory** Service Catalog product in
management account `349603509961`, assuming a role from service account
`774305573766`. Never use Organizations `CreateAccount`.

### Requirement precedence

The latest request overrides FR-002 and the existing specification's entity/
assumption text: consumers supply no email and the OU is required, not optional.
Derive `AccountEmail` as `admin+<account_name>@serverlessops.io`. Return the
acceptance time in status, not the `201` body; provide the status URL in `Location`.
Retain the other existing requirements, including concurrent retry deduplication,
ownership checks, ordered progress, finite retries, partial-failure results and
90-day terminal retention. Reconcile the old specification text before
implementation; this planning change leaves it intact and records the differences.

## Technical Context

- **Language/version**: SAM/CloudFormation YAML, ASL with JSONata, OpenAPI 3.0.3;
  no application runtime.
- **Dependencies**: API Gateway REST, Cognito, Step Functions, DynamoDB, STS,
  Service Catalog, Organizations read APIs, CloudWatch and EventBridge.
- **Storage**: DynamoDB ledger; no active-record TTL; terminal and retry records
  retained at least 90 days after completion.
- **Testing**: Existing `sam validate --lint` and `sam build`; no application
  test runner exists. Future behavior scenarios: [quickstart.md](quickstart.md).
- **Platform**: Service account `774305573766`, management account `349603509961`,
  same configured Control Tower home Region.
- **Type/scope**: Asynchronous API for one organization and approved registered
  OUs; existing health route unchanged.
- **Performance**: Preserve the spec's 95% acceptance within 5 seconds and 95%
  progress visibility within 30 seconds; poll provider records every 15 seconds.
- **Constraints**: SAM/ASL and native integrations, least privilege, no Lambda,
  finite 24-hour provisioning budget, no automatic account deletion.

## Constitution Check

| Principle | Before research | After design |
|---|---|---|
| I. State-machine-first | PASS: native integrations selected | PASS: no Lambda proposed |
| II. Idempotent/resilient | PASS: explicit design requirement | PASS: atomic ledger, stable execution input and provision token, bounded retry/reconciliation |
| III. Error logging | PASS: handled errors included | PASS: sanitized catch-path and unhandled-error logs with transaction/state context |
| IV. State documentation | PASS: required during implementation | PASS: each future ASL state requires a purpose/input/output/side-effect/error-handling `Comment` |
| V. AWS-native | PASS: SAM/ASL retained | PASS: Gateway starts workflows directly; tasks call AWS directly |

No exceptions are needed. The constitution's unconfirmed ratification date is
pre-existing metadata, not a feature-design blocker.

## Project Structure

Planning artifacts in `specs/001-create-aws-account/`:
`plan.md`, `research.md`, `data-model.md`, `quickstart.md`, and
`contracts/account-api.yaml`. The existing `spec.md` remains unchanged.

Future implementation extends `template.yaml`, `openapi.yaml`,
`cfn-parameters.json`, `statemachine/CreateAccount.asl.yaml`, and README.
Small acceptance, status, and reconciliation ASL workflows would live beside the
existing workflow. A separate management-account SAM/CloudFormation template
would define the cross-account IAM role. These source changes are not made here.

## Design

### Acceptance and status

1. Retain Cognito and `account.write`; add `transactions.read`. API Gateway
   validates JSON shape/content type and supplies verified consumer identity
   separately from the body. Reject unknown fields and consumer-supplied email,
   SSO settings, role ARNs or product IDs.
2. Directly invoke a synchronous Express acceptance workflow using
   `StartSyncExecution`. Validate an email-safe account name and exact approved
   OU name. An optional `Idempotency-Key` UUID supplies the spec's retry reference;
   requests without one are independent submissions.
3. Atomically reserve the consumer-scoped retry key, initialized transaction,
   and derived email in DynamoDB. Identical retries reuse the stored transaction;
   conflicting details return `409`. Prevent two independent transactions from
   concurrently provisioning the same email.
4. Start the Standard provisioning workflow with the saved transaction UUID as
   execution name and immutable saved input. Concurrent/repeated starts use
   exactly the same name/input. Recover an ambiguous start by describing that
   execution, not by generating another transaction.
5. Return `201` only after a durable reservation and confirmed start, or a replay
   of an already accepted transaction. Map Express business errors explicitly;
   transport HTTP `200` from `StartSyncExecution` is not proof of success.
   Map transient acceptance failures to sanitized `503`.
6. `GET /transactions?txn_id=...` directly invokes a read-only Express workflow
   that strongly reads the ledger, verifies ownership, and projects safe progress.
   Unknown, expired, and other-consumer IDs receive indistinguishable `404`.
   No arbitrary execution ARN or account ID becomes an AWS resource selector.

The small Express adapters preserve direct Gateway-to-Step Functions integration.
A bare `StartExecution` mapping cannot implement concurrent/terminal retry
recovery, durable ownership checks, and status projection by itself.

### Provisioning

Initialize all milestones before starting: `validate_eligibility`,
`request_account_creation`, `wait_for_account_creation`, and
`confirm_environment_enrollment`.

1. Resolve `ou_name` through a deployment-approved exact name-to-ID map; reject
   ambiguous names. Recheck eligibility and registered-OU configuration before
   the side effect. Resolve the configured Account Factory product, pinned
   artifact and launch path, and verify its actual provisioning parameters.
2. Use task `Credentials.RoleArn` to assume the management role. Call
   `servicecatalog:provisionProduct` with:
   - `AccountName`: request `account_name`.
   - `AccountEmail`: `admin+<account_name>@serverlessops.io`.
   - `ManagedOrganizationalUnit`: `<ou_name> (<ou_id>)`, verified against the
     deployed product's allowed values.
   - `SSOUserFirstName`, `SSOUserLastName`, `SSOUserEmail`: configured values.
3. Use a stable provisioned-product name and `ProvisionToken` derived from
   `txn_id`. Save provider record/product IDs immediately. Retrying an ambiguous
   call must use the same token/parameters and reconcile provider records;
   never blindly start another provision operation.
4. Poll `DescribeRecord` with Wait/Choice states. Explicitly handle every provider
   terminal/nonterminal status; persist milestone changes and any available
   account ID, including partial results. Polling is `in_progress`, not a retry.
5. Confirm successful Account Factory provisioning, a healthy provisioned product,
   an active Organizations account and the requested registered-OU parent.
   Account existence alone does not prove enrollment. Validate Account Factory's
   completion signal and account-ID output in a sandbox before rollout.
6. Report success only after all milestones complete. On failure retain completed
   steps/account ID, identify the failed step safely, and mark remaining steps
   `not_performed`. Do not terminate products or close accounts automatically.

### Resilience and retention

- Transient API failures: exponential backoff with jitter, initially 2 seconds,
  maximum 60 seconds, at most 6 retries per call within a shared 24-hour deadline.
  Permanent validation/permission failures do not loop indefinitely.
- Start serialized Account Factory provisioning unless operators confirm a
  supported higher concurrency. Queue waiting counts toward the finite deadline.
- A provider operation may outlive the deadline. Report terminal
  `provisioning_timeout` with known IDs and operator-recovery guidance; do not
  resubmit or automatically release its email reservation.
- Use conditional/versioned writes for monotonic steps and immutable terminal
  outcomes. Keep retry mappings at least as long as their transactions.
- EventBridge terminal-execution events trigger native reconciliation for
  uncaught errors, aborts, timeouts and failed final ledger writes. A periodic
  repair pass handles missed events, stale acceptance reservations and active
  records past deadline. Verify execution/provider state before finalizing;
  late events must never regress terminal outcomes.
- Log sanitized handled/unhandled errors with transaction, milestone and attempt
  context. Disable raw execution-data logging; never expose SDK causes, SSO
  payloads, request bodies, JWTs or credentials. Alarm on stale progress,
  throttling, failed reconciliation, acceptance latency and provisioning failures.

## Configuration and Deployment

Future component `cfn-parameters.json` must expose these SAM parameters:

| Parameter | Initial value or purpose |
|---|---|
| `SSOUserFirstName` | `Tom` |
| `SSOUserLastName` | `McLaughlin` |
| `SSOUserEmail` | `tom@serverlessops.io` |
| `AccountFactoryRoleArn` | Management-account caller role ARN |
| `AccountFactoryProductId` | Discovered Account Factory product ID |
| `AccountFactoryProvisioningArtifactId` | Approved pinned artifact |
| `AccountFactoryLaunchPathId` | Approved portfolio launch path |
| `AllowedOrganizationalUnits` | Unique registered OU names mapped to IDs |

Pass SSO values from SAM to ASL through definition substitutions. They are never
accepted from the consumer, and SSO email does not determine AccountEmail.

Deploy the target role separately in `349603509961`; the service stack cannot
create a role in another account merely by declaring it. The service workflow
role receives only `sts:AssumeRole` on that ARN; the management role trusts that
exact service role, not the whole service account. Bootstrap roles in separate
deployments to avoid circular trust dependencies.

Grant the management caller narrowly scoped Service Catalog provision/describe
and Organizations read permissions, documenting any unavoidable wildcard list/
read actions. Do not grant Organizations `CreateAccount`. Verify portfolio
access and the separate Service Catalog launch role/Control Tower execution
roles; assuming a caller role alone does not establish these prerequisites.

## Implementation Sequence and Verification

1. Reconcile specification overrides and approve Region, registered OU map,
   product/artifact/path, trust policy and sandbox completion evidence.
2. Define management role deployment, component parameters, ledger, workflow
   roles and reconciliation infrastructure using SAM.
3. Implement acceptance/status contracts and native adapters.
4. Replace provisioning placeholders with documented Account Factory states,
   progress persistence, bounded errors and partial-result handling.
5. Execute [quickstart.md](quickstart.md), including concurrent retries,
   cross-consumer denial, partial failure and lost-event reconciliation.
6. Update README alongside implementation; preserve health and unrelated routes.

All design choices are specified. Live identifiers, product schema and sandbox
proof are deployment prerequisites, not guessed repository values; failed
verification blocks rollout.

## Complexity Tracking

No constitutional violations. The ledger and native API adapters are needed for
the existing spec's idempotency, ownership and retention guarantees. No generic
service framework or Lambda layer is introduced.
