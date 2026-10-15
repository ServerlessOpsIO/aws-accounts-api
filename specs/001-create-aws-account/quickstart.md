# Validation Guide: Control Tower Account Creation

Future implementation guide only. These endpoints are not implemented by this
planning change. Do not provision production accounts to validate documentation.

## Prerequisites

- Approve the specification overrides in [plan.md](plan.md).
- Obtain an authorized sandbox in the service account, management-account
  deployment access, and the confirmed Control Tower home Region.
- Deploy/verify the management caller role, exact trust, portfolio access,
  launch constraints and existing Control Tower execution roles.
- Discover product, artifact, launch path and uniquely named registered OUs.
  Verify the actual provisioning parameter schema and account-ID/enrollment
  completion signals against that artifact.
- Configure the three SSO values through component `cfn-parameters.json`.
- Obtain two independent Cognito consumers with creation/read scopes.
- Reserve unique email-safe sandbox names and get approval for account costs.

## Existing checks

After implementation, from the repository root:

```sh
sam validate --lint
sam build
```

These existing checks do not prove IAM, VTL mappings, provider behavior or
concurrency. No application test runner currently exists. For this documentation
change, use document/contract review and `git diff --check`; no deployment needed.

## Future API smoke scenario

Set `BASE_URL`, `TOKEN`, an approved `OU_NAME`, a unique email-safe `ACCOUNT_NAME`,
and a UUID `RETRY_KEY`. Never persist credentials in repository files.

```sh
curl -i -X POST "$BASE_URL/v1/account" \
  -H "Authorization: $TOKEN" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $RETRY_KEY" \
  --data "{\"account_name\":\"$ACCOUNT_NAME\",\"ou_name\":\"$OU_NAME\"}"
```

Expect `201`, only `txn_id` in the body, and a `Location` header. Set `TXN_ID` to
the returned UUID and query:

```sh
curl -i "$BASE_URL/transactions?txn_id=$TXN_ID" -H "Authorization: $TOKEN"
```

Expect `200`, timestamps and all ordered milestones, including not-started steps.
Poll through completion. Verify the derived email, resulting account ID, correct
OU and Account Factory enrollment using operator credentials.

## Focused acceptance scenarios

| Scenario | Required observation |
|---|---|
| Missing/empty/wrong-type fields, invalid name, unknown fields | `400` with safe field information; no provisioning |
| Unknown/ambiguous/unregistered OU | `400` when excluded by config; eligibility failure if changed after acceptance |
| No/expired JWT or missing scope | `401`/`403`; no provisioning or protected status |
| Same key/body, reordered JSON, concurrent calls, lost response | Same transaction, one execution and one provider provision operation |
| Same key, different name or OU | `409`, original unchanged |
| Identical retry after completion throughout retention | Original transaction and immutable outcome |
| Different requests for same derived email | No duplicate provisioning or adoption of unrelated account |
| Malformed/missing transaction ID | `400`; no AWS ARN injection |
| Unknown, expired, or other consumer's transaction | Indistinguishable `404` |
| Provider throttling/transient failure | Bounded retry, visible `awaiting_retry`, stable side-effect token |
| Permission/provider terminal failure | Failed milestone, safe reason, later steps not performed |
| Account created, later verification fails | Failed outcome preserves account ID/completed steps |
| Local timeout while provider continues | Safe timeout with known IDs; no automatic deletion/resubmission |
| Aborted workflow, failed final write, missed terminal event | Event/periodic reconciliation repairs ledger without regression |
| Status store temporarily unavailable | `503`, not a fabricated provisioning failure |
| SSO settings change between attempts | New transactions use new config; retries use original snapshot |

Inspect sanitized catch-path and unhandled-error logs for transaction/step
context and absence of JWTs, credentials, SSO payloads and raw SDK causes.
Check active records have no TTL; verify terminal/retry expiry is at least
completion plus 90 days with controlled timestamps, rather than waiting 90 days.
Measure 5-second acceptance and 30-second visibility targets under agreed load.
Document partial accounts for operators; cleanup must not automatically close
accounts or terminate products.
