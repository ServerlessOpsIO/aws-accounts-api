# Backstage / aws-accounts-api

API for managing AWS accounts

The full API can be found in the [OpenAPI document](./openapi.yaml).


## Architecture
This is an AWS Step Functions state machine that is started through an API. It is built on top of the following AWS services:
* API Gateway
* Step Functions
* Cognito (See _Authentication and Authorization_ for more)

API Gateway starts the state machine directly, without a Lambda function in between.


## New Project Getting Started
This repository was generated from a template intended to get a new AWS Step Functions state machine with an API up and running quickly. It uses the [AWS Serverless Application Model (SAM)](https://aws.amazon.com/serverless/sam/) to define the infrastructure as code.

### Project layout

- [`template.yaml`](template.yaml): AWS SAM template that defines the state machine, the API, and their supporting resources.
- [`openapi.yaml`](openapi.yaml): OpenAPI document that defines the API. The SAM template also uses it to create the API Gateway resources.
- [`statemachine/CreateAccount.asl.yaml`](statemachine/CreateAccount.asl.yaml): the state machine definition, written in [Amazon States Language (ASL)](https://docs.aws.amazon.com/step-functions/latest/dg/concepts-amazon-states-language.html).
- [`cfn-parameters.json`](cfn-parameters.json): parameter values used when the stack is deployed.
- [`cfn-tags.json`](cfn-tags.json): tags applied to the deployed stack.
- [`.github/workflows/`](.github/workflows/): GitHub Actions workflows that validate, build, and deploy the stack.

### API

`POST https://accounts.platform.serverlessops.io/v1/account` starts an execution of the **CreateAccount** state machine.

The JSON body requires `account_name` and `ou_name`:

```sh
curl -X POST https://accounts.platform.serverlessops.io/v1/account \
  -H "Authorization: $ACCESS_TOKEN" -H 'Content-Type: application/json' \
  -d '{"account_name":"example-dev","ou_name":"Sandbox"}'
```

Successful acceptance returns HTTP `201` with `{"txn_id":"<execution-arn>"}`.
This is asynchronous acceptance, not confirmation that the account already exists.
Account names must start with an alphanumeric character, contain only alphanumeric
characters, underscores or hyphens, and be at most 50 characters long so the generated
email remains valid.

```sh
curl --get https://accounts.platform.serverlessops.io/transactions \
  -H "Authorization: $ACCESS_TOKEN" \
  --data-urlencode "txn_id=$TXN_ID"
```

`GET /transactions` requires the returned `txn_id` query parameter and returns
`txn_id`, `status` (`RUNNING`, `SUCCEEDED`, `FAILED`, `TIMED_OUT`, or `ABORTED`),
and `progress`: the latest 100 Step Functions history events, newest first.
Events identify the current workflow step without returning execution input/output.
Older history and provisioning outputs remain available through Step Functions.
Malformed/missing request fields and invalid transaction IDs return `400`.

#### Authentication and Authorization
This service is configured to use a pre-existing Cognito User Pool. Clients should obtain a JWT from the Cognito token endpoint using the client's clientId and clientSecret. Each endpoint's scope requirements are defined in the [OpenAPI document](./openapi.yaml). Account creation requires the `https://accounts.platform.serverlessops.io/account.write` scope; transaction reads require `https://accounts.platform.serverlessops.io/account.read`. Configure both scopes in the existing Cognito resource server and allow them on the consumer's app client. Transaction reads are service-wide, not isolated per consumer.

### State machine

The **CreateAccount** state machine assumes a role in management account
`349603509961` from service account `774305573766`. It searches all organizational
units (including nested OUs and paginated results) for a unique `ou_name`, then
provisions the **AWS Control Tower Account Factory** Service Catalog product.
The OU must already be enrolled in Control Tower. Missing or ambiguous OU names
fail the transaction before any provisioning request.

Account Factory receives `AccountName`, `ManagedOrganizationalUnit` in
`Name (ou-id)` format, and `AccountEmail` as `admin+<account_name>@serverlessops.io`.
`SSOUserFirstName`, `SSOUserLastName`, and `SSOUserEmail` are configured in
[`cfn-parameters.json`](cfn-parameters.json), initially `Tom`, `McLaughlin`, and
`tom@serverlessops.io`. Consumers cannot override these values.

The JSONata workflow uses the execution name as Service Catalog's idempotency
token, so task retries do not create duplicate products. Separate POST requests
are separate transactions. It polls the provisioning record every 30 seconds,
succeeds only on `SUCCEEDED`, and fails on `FAILED`. Other in-progress states
continue polling, bounded by a two-hour execution timeout. Failures appear in
transaction history and the existing ERROR-level CloudWatch logs.

You can edit the definition visually with [Workflow Studio in the AWS Toolkit for VS Code](https://docs.aws.amazon.com/step-functions/latest/dg/workflow-studio-local.html).

### Permissions

SAM creates the execution role with permission to assume only
`AccountFactoryRoleArn`. API Gateway may start only this state machine and read
only its execution histories. No `organizations:CreateAccount` call is used.

#### Cross-account deployment

1. Deploy the API stack in `774305573766` in the **Control Tower home region**.
   Set repository variables `ACCOUNT_FACTORY_PRODUCT_ID` and
   `ACCOUNT_FACTORY_PROVISIONING_ARTIFACT_ID` to the management account's
   Account Factory product and active artifact IDs. The deployment workflow
   resolves these into `cfn-parameters.json`; that file also contains the
   configurable SSO identity and target role ARN.
2. Read the API stack outputs `CreateAccountExecutionRoleArn` and
   `CreateAccountStateMachineArn`.
3. Deploy [`management-account.yaml`](management-account.yaml) in `349603509961`
   in the same region, using those outputs:

   ```sh
   aws cloudformation deploy --profile management --region "$CONTROL_TOWER_HOME_REGION" \
     --template-file management-account.yaml --stack-name aws-accounts-api-account-factory \
     --capabilities CAPABILITY_NAMED_IAM \
     --parameter-overrides ExecutionRoleArn="$EXECUTION_ROLE_ARN" \
       StateMachineArn="$STATE_MACHINE_ARN" AccountFactoryProductId="$PRODUCT_ID"
   ```

   This creates `aws-accounts-api-account-factory`, trusting only the API execution
   role with the state machine ARN as the STS external ID. The default
   `AccountFactoryRoleArn` already targets this role.
4. Grant this management-account role principal access to the Account Factory
   Service Catalog portfolio. Configure a Service Catalog launch constraint
   with a launch role authorized for Account Factory provisioning (and trusted
   by `servicecatalog.amazonaws.com`). The API's assumed role deliberately
   does not have general CloudFormation, IAM, or account-administration access;
   those provisioning permissions belong to the launch role. Ensure the
   selected product has an unambiguous launch path.

Do not send requests until management-role trust, portfolio access, launch
permissions, and Cognito scopes are configured. Redeploy the management stack if
the API execution role or state machine ARN changes.

### Local validation

Validate the template before you push:

```sh
sam validate --lint
sam validate --lint --template-file management-account.yaml
sam build
```

After deployment, verify a valid request returns `201` and its transaction
progresses through OU lookup, provisioning, and polling to `SUCCEEDED`. A request
missing either required field should return `400`. A nonexistent or ambiguous OU
should be accepted asynchronously but finish `FAILED` without provisioning.
Check a provisioning failure reports `FAILED`, and ensure a transaction ARN from
another state machine cannot be read through this API. These live checks create
real accounts and require authorized AWS credentials; do not run them against
production without approval.
