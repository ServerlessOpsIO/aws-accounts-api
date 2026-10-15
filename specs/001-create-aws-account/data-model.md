# Data Model: Account Creation Transactions

Design only; no resources are created.

## Request and configuration

| Field | Rules |
|---|---|
| `account_name` | Required, 1–50 characters, `^[a-z0-9]+(-[a-z0-9]+)*$`; no silent normalization |
| `ou_name` | Required, nonempty, maximum 128 characters; exact approved registered OU name |
| Retry reference | Optional `Idempotency-Key` UUID header, scoped to authenticated consumer |
| Owner | Verified Cognito `client_id` for machine clients or `sub` for users, with distinct identity-type namespaces; never body-supplied |

Reject unknown body properties. The name policy intentionally narrows AWS account
names to make the exact supplied name email-safe. Validate the full derived
`admin+<account_name>@serverlessops.io`; its local part is at most 56 characters.

Deployment configuration contains management role, home Region, product/artifact/
launch path, unique OU name-to-ID map and three SSO values. Snapshot resolved
configuration per transaction so configuration updates cannot change a retry's
provision parameters or token meaning. Never expose SSO settings in status.

## DynamoDB ledger

Use conditional writes and atomic `TransactWriteItems` reservations.

### Transaction

Key: `PK = TXN#<uuid>`, `SK = META`.

- `txn_id`: opaque UUID; basis for execution name and stable provision token.
- `owner`, canonical request fields and immutable resolved configuration.
- `status`: `in_progress`, `succeeded`, `failed`.
- `accepted_at`, `updated_at`, and terminal `completed_at`: UTC RFC 3339.
- Ordered `steps`, initialized before provisioning begins.
- Internal execution ARN, saved immutable input, start confirmation, deadline,
  retry metadata, record/product IDs and provision token.
- Optional public `account_id`, exactly 12 digits; retained on partial failure.
- Optional safe error category/message/step ID.
- `version` for conditional updates.
- `expires_at`: epoch-seconds TTL, absent while active; at least completion plus
  90 days when terminal. Reads enforce expiry because TTL deletion is asynchronous.

### Retry mapping

Key: `PK = OWNER#<namespaced-identity>`, `SK = RETRY#<retry-uuid>`.

Store transaction ID and canonical two-field request. Reserve atomically with
transaction/email records. On conflict, strongly read: identical fields reuse
the winner; changed fields return `409`. Compare field values, not JSON key
order or volatile request IDs/timestamps.

Active retry records have no TTL. Terminal mappings expire no earlier than the
transaction. Replace expired records conditionally rather than waiting for TTL
deletion to unblock reuse.

### Email reservation and repair index

Key: `PK = EMAIL#<derived-email>`, `SK = RESERVATION`.

Record the owning transaction and known provider IDs. Prevent independent
requests from concurrently provisioning the same email. Successful and uncertain
reservations survive transaction expiry; release only after verified no-side-effect
failure or explicit operator reconciliation.

A sparse index on repair state/next-check time supports periodic bounded
reconciliation without full table scans. Conditional updates prevent late
events or competing repair executions from regressing an outcome.

## Milestones and transitions

| Order | ID | Meaning |
|---|---|---|
| 1 | `validate_eligibility` | Approved registered OU and valid product/configuration |
| 2 | `request_account_creation` | Account Factory acknowledges provision request |
| 3 | `wait_for_account_creation` | Provider record reaches terminal outcome |
| 4 | `confirm_environment_enrollment` | Account Factory result and account placement verified |

Each step has `id`, `name`, `status`, optional `started_at`, `completed_at`, and
safe failure details. Completed/failed steps require completion timestamps.

- Transaction: `in_progress → succeeded | failed`; terminal outcomes immutable.
- Step: `not_started → in_progress → completed | failed`.
- Transient failure: `in_progress → awaiting_retry → in_progress`.
- After a failed milestone, remaining steps become `not_performed`.
- Waiting for provider completion remains `in_progress`; completed steps never
  regress. Store account IDs as soon as available, before later work can fail.

## Reservation/start recovery

Persist initialized records before starting the Standard execution. Concurrent
adapters use the winning saved UUID/input. If acknowledgement is lost, verify
that exact execution; never allocate a second transaction. Return `201` only
after confirmed start or replay of an accepted transaction. Native reconciliation
repairs stale reservations and missing terminal ledger writes.
