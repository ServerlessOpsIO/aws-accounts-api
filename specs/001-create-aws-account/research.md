# Research and Decisions: Control Tower Account Creation

Planning conclusions below distinguish design choices from live-environment
facts. No management account, portfolio, product or OU has been inspected.

## 1. Provision through Account Factory

**Decision**: Use Service Catalog `ProvisionProduct`, with the Account Factory
product/artifact/launch path explicitly configured. Poll `DescribeRecord`.

**Rationale**: The organization uses Control Tower and the request explicitly
excludes the ordinary Organizations account-creation API. Account Factory owns
the account provisioning/enrollment operation.

**Alternatives considered**: Organizations `CreateAccount` is prohibited;
invoking AWS CLI through Lambda or a worker adds an unnecessary runtime.

**References**:
- [Create accounts through Account Factory](https://docs.aws.amazon.com/controltower/latest/userguide/account-factory.html)
- [ProvisionProduct](https://docs.aws.amazon.com/servicecatalog/latest/dg/API_ProvisionProduct.html)
- [DescribeRecord](https://docs.aws.amazon.com/servicecatalog/latest/dg/API_DescribeRecord.html)
- [DescribeProvisioningParameters](https://docs.aws.amazon.com/servicecatalog/latest/dg/API_DescribeProvisioningParameters.html)

**Deployment verification**: Discover real product, artifact, launch path and OU
allowed values in the Control Tower home Region. Check
`ManagedOrganizationalUnit` formatting and the result's account-ID output against
the selected artifact. Never guess product identifiers or assume an arbitrary
OU is registered.

## 2. Assume a narrowly scoped management role

**Decision**: Use Step Functions task `Credentials.RoleArn` for management-account
calls. Deploy the target role separately and trust the exact service workflow
role; grant matching source `sts:AssumeRole`.

**Rationale**: The caller and Control Tower are in different accounts; a source
IAM allow policy alone does not grant management-account access.

**Alternatives considered**: Static management credentials are unacceptable.
Deploying the entire API into the management account contradicts its stated
placement and increases exposure. A Lambda STS wrapper is unnecessary.

**References**:
- [Cross-account AWS resource access in Step Functions](https://docs.aws.amazon.com/step-functions/latest/dg/concepts-access-cross-acct-resources.html)
- [Task credentials](https://docs.aws.amazon.com/step-functions/latest/dg/state-task.html)

**Deployment verification**: Use the same Region for the native cross-account
integration; verify source/target role trust, portfolio sharing/access, product
launch constraints and the separate Control Tower/Service Catalog launch roles.
Determine which read/list permissions require wildcard resources and document
those exceptions rather than granting broad administration.

## 3. Keep direct native API integration, with durable acceptance

**Decision**: API Gateway directly invokes synchronous Express adapters for
acceptance/status; acceptance starts the long-running Standard workflow.
DynamoDB atomically records transaction ownership and retry mappings.

**Rationale**: The existing spec requires concurrent deduplication, matching and
conflicting retry behavior even after completion, field-specific validation and
owner-authorized reads. Standard `StartExecution` idempotency alone does not
provide this complete contract. Express adapters use direct service calls and
do not perform account provisioning.

**Alternatives considered**: A bare Gateway `StartExecution` mapping is simpler
but insufficient for the full spec. A Lambda API layer violates the native-first
constraint when native state machines can perform these operations.

**References**:
- [StartExecution idempotency](https://docs.aws.amazon.com/step-functions/latest/apireference/API_StartExecution.html)
- [StartSyncExecution response semantics](https://docs.aws.amazon.com/step-functions/latest/apireference/API_StartSyncExecution.html)
- [DynamoDB transactions](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/transactions.html)

**Design constraint**: Inspect the Express execution status and safe business
output when mapping HTTP responses; transport `200` does not imply success.
Use immutable saved Standard execution input, not volatile API request metadata.
Only acknowledge `201` after confirming the Standard execution start.

## 4. Persist progress rather than projecting raw execution history

**Decision**: Record ordered business milestones and terminal results in DynamoDB;
no active-record TTL and terminal expiry at least completion plus 90 days.
Use native terminal-event and periodic reconciliation.

**Rationale**: Consumers need stable steps, safe errors, ownership boundaries and
retained results, not internal ASL events. Uncaught workflow errors or failed
final writes must not strand accepted work indefinitely.

**Alternatives considered**: CloudWatch logs are diagnostics, not a transaction
API. `DescribeExecution` alone lacks the required business model and authorization
projection; execution history retention is not the API's retention policy.

**References**:
- [Step Functions quotas and retention](https://docs.aws.amazon.com/step-functions/latest/dg/service-quotas.html)
- [Step Functions EventBridge events](https://docs.aws.amazon.com/step-functions/latest/dg/eventbridge-integration.html)
- [DynamoDB TTL deletion behavior](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/TTL.html)

## 5. Stable side-effect identity, bounded retries and completion

**Decision**: Derive a persistent Service Catalog `ProvisionToken` and product
name from `txn_id`; preserve parameters across retries. Use a 24-hour local
deadline, 15-second polling and bounded transient retries. Start serialized
unless supported provider concurrency is confirmed. Treat provider success plus
account/OU verification as the enrollment confirmation gate.

**Rationale**: An uncertain response must not cause a second account. Account
Factory can continue after a local timeout, so neither a retry nor cleanup may
silently create/delete resources. Account existence alone is not enrollment.

**Alternatives considered**: Random per-attempt tokens, unbounded waits, treating
submission as provisioning success, and destructive rollback are unsafe.

**Deployment verification**: Exercise ambiguous provision responses, all provider
terminal statuses, eventual account output, and the enrollment signal in a
sandbox. If the selected product cannot provide sufficient completion evidence,
block rollout and extend the confirmation design before implementation approval.

## 6. Configuration and validation boundaries

**Decision**: Require only `account_name` and `ou_name` in JSON; derive account
email unchanged from the validated name. Put the three requested SSO defaults in
component `cfn-parameters.json` during implementation and pass them through SAM.
Use an exact configured registered-OU map.

**Rationale**: Consumers must not override enrollment identity, role/product
configuration or organizational destination outside the allowlist.

**Alternatives considered**: Consumer-supplied email/SSO contradicts the new
request. Silent account-name normalization risks collisions; reject unsupported
names instead. Fuzzy OU selection risks incorrect placement.

## Resolution summary

No design-time `NEEDS CLARIFICATION` items remain. Product IDs, Region, OU map,
role/portfolio access and provider output verification are explicit operator
deployment inputs/checks. This plan does not claim they were discovered or
validated against either live AWS account.
