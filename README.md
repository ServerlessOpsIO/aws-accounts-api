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

The JSON request body is passed to the state machine as its input.

The state machine is a `STANDARD` workflow, so API Gateway starts it asynchronously and returns `202` with the execution ARN and start time. Use the execution ARN to check the execution's status, for example with `aws stepfunctions describe-execution`.

#### Authentication and Authorization
This service is configured to use a pre-existing Cognito User Pool. Clients should obtain a JWT from the Cognito token endpoint using the client's clientId and clientSecret. Each endpoint's scope requirements are defined in the [OpenAPI document](./openapi.yaml). The state machine endpoint requires the `https://accounts.platform.serverlessops.io/account.write` scope.

### State machine

The **CreateAccount** state machine starts with a single placeholder `Pass` state followed by a `Succeed` state. Replace these states with the steps of your workflow.

The definition uses the JSONata query language. The `Domain`, `System`, `Component`, and `CodeBranch` stack parameters are passed into the definition as substitutions, so you can reference them as `${Domain}`, `${System}`, and so on. Add more entries to `DefinitionSubstitutions` in `template.yaml` to pass in resource ARNs, such as Lambda functions or SNS topics.

You can edit the definition visually with [Workflow Studio in the AWS Toolkit for VS Code](https://docs.aws.amazon.com/step-functions/latest/dg/workflow-studio-local.html).

### Permissions

The state machine runs with an IAM role that AWS SAM creates. When you add states that call other AWS services, grant the matching permissions under `Policies` in `template.yaml`. Use [AWS SAM policy templates](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-policy-templates.html) where possible.

API Gateway uses the `RestApiIamRole` role to start executions. It only allows starting this state machine.

### Local validation

Validate the template before you push:

```sh
sam validate --lint
```
