# Troubleshooting Guide

This guide reflects the Terraform configuration currently in this repository.
Run commands from the repository root unless a command explicitly changes
directory.

## Quick Checks

Start with these checks before debugging a specific AWS service:

```bash
terraform fmt -check -recursive
terraform init -backend=false
terraform validate
```

If validation fails because Lambda zip files are missing, build the packages
first:

```bash
cd modules/lambda/src/router
npm ci
npm run zip

cd ../processor
npm ci
npm run zip

cd ../../../..
terraform validate
```

## Terraform Init Or Validate Fails

### Module not installed

Symptom:

```text
Error: Module not installed
```

Fix:

```bash
terraform init
```

For local validation without configuring or contacting a remote backend:

```bash
terraform init -backend=false
```

### Missing Lambda zip files

Symptom:

```text
Call to function "filebase64sha256" failed:
open modules/lambda/src/router/router.zip: no such file or directory
```

or:

```text
open modules/lambda/src/processor/processor.zip: no such file or directory
```

Cause: the Lambda module references local zip artifacts, and `*.zip` is ignored
by git.

Fix:

```bash
cd modules/lambda/src/router
npm ci
npm run zip

cd ../processor
npm ci
npm run zip

cd ../../../..
terraform validate
```

### Missing required variables

Symptom:

```text
Error: No value for required variable
```

The most likely required value is `db_master_password`.

Fix with a tfvars file:

```bash
cp terraform.tfvars.example terraform.tfvars
terraform plan -var-file="terraform.tfvars"
```

Or fix with an environment variable:

```bash
export TF_VAR_db_master_password="replace-with-a-strong-password"
terraform plan
```

## Script Failures

The scripts in `scripts/` expect this directory layout:

```text
environments/
└── dev/
    └── terraform.tfvars
```

If you see:

```text
Error: Environment directory 'environments/dev' does not exist.
```

create the environment directory and tfvars file:

```bash
mkdir -p environments/dev
cp terraform.tfvars.example environments/dev/terraform.tfvars
```

Then edit the values and run:

```bash
./scripts/validate.sh dev
./scripts/plan.sh dev
./scripts/deploy.sh dev
```

If you are not using per-environment folders, use the direct Terraform workflow
instead:

```bash
terraform init
terraform plan -var-file="terraform.tfvars"
```

## AWS Permission Errors

Symptom:

```text
AccessDenied
UnauthorizedOperation
User is not authorized to perform
```

Checks:

- Confirm AWS credentials are active: `aws sts get-caller-identity`.
- Confirm the selected region matches `var.aws_region`.
- Confirm the caller can create VPC, EC2, IAM, RDS, DynamoDB, S3, ECR, ECS,
  Lambda, API Gateway, Cognito, WAFv2, Kinesis Firehose, CloudWatch, SNS, and
  SQS resources.
- IAM resources require permissions such as `iam:CreateRole`,
  `iam:AttachRolePolicy`, `iam:PutRolePolicy`, and `iam:PassRole`.

## API Gateway Errors

### 401 or 403 responses

Checks:

- The API method uses `COGNITO_USER_POOLS` authorization.
- Obtain a valid Cognito token from the created user pool and send it in the
  `Authorization` header.
- Check whether the WAF ACL is blocking the request.
- Confirm the deployed stage name matches `var.environment`.

Useful commands:

```bash
terraform output api_gateway_url
terraform output cognito_user_pool_id
terraform output cognito_client_id
```

### 500 responses or wrong backend routing

The API Gateway module currently defines a greedy `ANY /{proxy+}` method and
declares integrations for both the ECS ALB and the router Lambda. If requests
route unexpectedly, split the API into explicit resources/methods or keep only
one integration for the proxy method.

Check CloudWatch logs for:

- API Gateway execution logs, if enabled.
- Router Lambda log group: `/aws/lambda/<project-name>-router`.
- ECS task logs: `/ecs/<project-name>-banking-api` and
  `/ecs/<project-name>-langflow`.

## Lambda Issues

### Router Lambda returns 400

The router Lambda expects a JSON body with:

```json
{
  "action": "process_payment",
  "payload": {
    "transactionId": "txn-123",
    "amount": 100,
    "currency": "USD",
    "recipient": "account123"
  }
}
```

Any other `action` returns `400`.

### Router Lambda returns 500

Checks:

- `SQS_QUEUE_URL` is set in the Lambda environment.
- The Lambda IAM policy allows SQS access to the queue.
- The Lambda function has network access through the configured private
  subnets and security group.
- Inspect `/aws/lambda/<project-name>-router` in CloudWatch Logs.

### SQS messages are not processed

The processor Lambda code expects an SQS event, but the current Terraform code
does not define an `aws_lambda_event_source_mapping` from the SQS queue to the
processor Lambda. Add that mapping or invoke the processor Lambda through
another event source.

## ECS And ALB Issues

### ALB has no healthy targets

Checks:

- ECS services currently do not attach a `load_balancer` block to the target
  group. Without that, tasks may not register as targets.
- The target group is configured for port `80`, while the containers expose
  `8080` for `banking-api` and `7860` for `langflow`.
- The health check path is `/health`; the container behind the target group
  must respond with HTTP `200` on that path.
- The ECS task definitions reference log groups under `/ecs/...`, but this
  module does not currently create those log groups.

Useful checks:

```bash
terraform output ecr_banking_api_repository_url
terraform output ecr_langflow_repository_url
```

Push valid container images before expecting ECS tasks to start successfully.

## Database Connection Issues

The database module provisions Aurora MySQL, not PostgreSQL. Use MySQL port
`3306`.

Checks:

- Use `terraform output aurora_cluster_endpoint` for the writer endpoint.
- Use `terraform output aurora_reader_endpoint` for the reader endpoint.
- Confirm the RDS security group allows inbound TCP `3306` from the ECS,
  Lambda, or SSM security group.
- Confirm the client runs from a network path that can reach private subnets.
- Confirm `db_master_username` and `db_master_password` match the values used
  at cluster creation.
- `publicly_accessible` is declared as a variable, but the current RDS cluster
  instance setting is commented out. Do not expect public database access.

For private troubleshooting, use the SSM host created in the database module if
the instance is reachable through Session Manager.

## S3, Firehose, And Analytics Issues

Checks:

- The S3 bucket name is output as `s3_bucket_name`.
- The Firehose IAM policy uses wildcard bucket ARNs matching `*-data-bucket`.
  If you change the bucket naming convention, update the policy.
- Firehose writes to S3 with `UNCOMPRESSED` output and a 300 second buffering
  interval, so objects may not appear immediately.
- QuickSight resources are commented out and are not provisioned.

## State And Plan File Issues

The root configuration does not define a remote backend. Local state and plan
files can contain sensitive values.

Recommendations:

- Do not commit `terraform.tfvars`, `*.tfstate`, or generated plan files.
- Configure a remote S3 backend with locking before team or production use.
- Prefer timestamped plan files in `plans/` only for local debugging.

## Useful Commands

```bash
terraform fmt -check -recursive
terraform validate
terraform plan -var-file="terraform.tfvars"
terraform state list
terraform output
aws sts get-caller-identity
```
