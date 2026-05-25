# Modular Payment System IaC

Terraform configuration for a modular payment-processing platform on AWS. The
root module composes networking, security, authentication, storage, database,
compute, messaging, analytics, and monitoring modules.

![System Architecture](images/modular_payment_system_architecture.png)

## What This Provisions

| Area | AWS resources |
| --- | --- |
| Networking | VPC, public subnets, private subnets, internet gateway, NAT gateways, route tables |
| Security | Security groups for ECS, Lambda, RDS, EC2/SSM, SageMaker, plus regional AWS WAF |
| Auth | Cognito user pool and user pool client |
| Storage | S3 data bucket with versioning and AES256 server-side encryption |
| Database | Aurora MySQL cluster, Aurora instances, DynamoDB table, private SSM host |
| Containers | ECR repositories, ECS Fargate task definitions and services, public ALB |
| Serverless | Router and processor Lambda functions running Node.js 18.x |
| Messaging | SQS queue and SNS topic |
| Analytics | Kinesis Firehose delivery stream to S3 |
| Monitoring | CloudWatch dashboard and ECS CPU alarm |
| IAM | ECS execution/task roles, Lambda roles/policies, Firehose role/policy, SSM role |

The AI/ML and QuickSight portions are partially scaffolded in comments and
policy files, but no active SageMaker or QuickSight resources are currently
provisioned by the root module.

## Repository Layout

```text
.
├── docs/                         # Additional architecture, deployment, API, and troubleshooting notes
├── images/                       # Architecture diagrams
├── modules/
│   ├── analytics/                # Kinesis Firehose and Firehose IAM role
│   ├── api_gateway/              # REST API Gateway, Cognito authorizer, WAF association
│   ├── cognito/                  # User pool and user pool client
│   ├── database/                 # Aurora MySQL, DynamoDB, SSM host
│   ├── ecr/                      # ECR repositories for banking API and Langflow
│   ├── ecs/                      # ECS cluster, Fargate services, ALB, target group
│   ├── iam_roles/                # ECS execution and task roles
│   ├── lambda/                   # Lambda IAM, functions, source code, package metadata
│   ├── messaging/                # SQS queue and SNS topic
│   ├── monitoring/               # CloudWatch dashboard and alarm
│   ├── s3/                       # Data bucket, versioning, encryption
│   ├── security/                 # Security groups and WAF ACL
│   └── vpc/                      # VPC, subnets, routing, NAT gateways
├── policies/                     # JSON IAM policy documents used by selected modules
├── scripts/                      # Environment-based Terraform helper scripts
├── main.tf                       # Root module composition
├── variables.tf                  # Root input variables
├── outputs.tf                    # Root outputs
├── version.tf                    # Terraform and provider constraints
└── terraform.tfvars.example      # Example variable values
```

`tfplan` is a generated Terraform plan artifact. It is not required for normal
usage and should not be treated as source configuration.

## Prerequisites

- Terraform `>= 1.2`
- AWS provider `~> 6.0`
- AWS CLI credentials with permission to create the listed AWS resources
- Node.js and npm for building the Lambda deployment packages
- `zip` command available locally

## Configuration

Create a local variables file from the example:

```bash
cp terraform.tfvars.example terraform.tfvars
```

Update values for your AWS account and environment:

```hcl
aws_region                 = "ap-southeast-1"
project_name               = "modular-payment-system"
environment                = "dev"
vpc_cidr                   = "10.0.0.0/16"
db_master_username         = "admin"
db_master_password         = "replace-with-a-strong-password"
enable_deletion_protection = true
backup_retention_period    = 7
skip_final_snapshot        = false
instance_class             = "db.r6g.large"
publicly_accessible        = false
```

Important variable notes:

- `db_master_password` is marked sensitive, but Terraform state can still store
  sensitive values. Use protected state storage and avoid committing tfvars or
  plan files.
- `domain_name` and `certificate_arn` are declared in the root variables but
  are not currently wired into API Gateway.
- `publicly_accessible` is declared for database instances, but the current
  RDS cluster instance block has that setting commented out.

## Build Lambda Packages

The Lambda functions reference local zip files:

- `modules/lambda/src/router/router.zip`
- `modules/lambda/src/processor/processor.zip`

These zip files are ignored by git, so build them before running
`terraform validate`, `terraform plan`, or `terraform apply`:

```bash
cd modules/lambda/src/router
npm ci
npm run zip

cd ../processor
npm ci
npm run zip

cd ../../../..
```

The current Lambda source uses `aws-sdk` v2 through `require("aws-sdk")`, so the
dependency must be installed before packaging.

## Direct Terraform Workflow

Use this flow when working from the root module with `terraform.tfvars`:

```bash
terraform init
terraform fmt -recursive
terraform validate
terraform plan -var-file="terraform.tfvars" -out="tfplan"
terraform apply "tfplan"
```

Destroy the stack when needed:

```bash
terraform destroy -var-file="terraform.tfvars"
```

Inspect state and outputs:

```bash
terraform state list
terraform output
terraform output api_gateway_url
```

## Scripted Environment Workflow

The scripts in `scripts/` expect environment-specific variable files in this
layout:

```text
environments/
├── dev/
│   └── terraform.tfvars
├── staging/
│   └── terraform.tfvars
└── prod/
    └── terraform.tfvars
```

After creating the relevant directory and tfvars file, run:

```bash
./scripts/validate.sh dev
./scripts/plan.sh dev
./scripts/deploy.sh dev
./scripts/destroy.sh dev
```

The scripts write logs to `logs/` and timestamped plan files to `plans/`.

## Runtime Flow

1. API Gateway exposes a Cognito-protected REST API stage named after
   `var.environment` and associates the stage with the regional WAF ACL.
2. The API Gateway module currently defines a greedy `ANY /{proxy+}` resource
   and includes integrations for both the ECS ALB and the router Lambda.
3. The router Lambda accepts a JSON body with `action = "process_payment"` and
   sends the `payload` to SQS.
4. The processor Lambda is intended to process SQS records and write
   transaction data to DynamoDB and S3.
5. ECS defines two Fargate services: `banking-api` on container port `8080` and
   `langflow` on container port `7860`.
6. Aurora MySQL stores relational data, DynamoDB stores NoSQL transaction data,
   and S3 stores transaction objects and Firehose output.

## Root Outputs

The root module exposes:

- `vpc_id`
- `api_gateway_url`
- `cognito_user_pool_id`
- `cognito_client_id`
- `aurora_cluster_endpoint`
- `aurora_reader_endpoint`
- `dynamodb_table_name`
- `s3_bucket_name`
- `ecr_banking_api_repository_url`
- `ecr_langflow_repository_url`
- `waf_acl_arn`

## Current Implementation Notes

Review these before using the stack outside a development account:

- Lambda zip files must exist locally before Terraform validation and planning.
- API Gateway currently has two integrations attached to the same proxy method;
  split routes or choose one integration model before production use.
- ECS services do not currently attach a `load_balancer` block to the target
  group, and the target group is configured for port `80` while containers
  expose `8080` and `7860`.
- Several IAM policies intentionally use wildcard resources. Tighten policies
  to resource-specific ARNs for production.
- Security group egress is broad in several modules, and the ECS security group
  allows inbound HTTP from `0.0.0.0/0`.
- The root configuration does not define a remote backend. Configure an S3
  backend with state locking before team or production use.
- CloudWatch log groups are created for Lambda, but ECS task definitions
  reference log group names without creating those groups in this module.
- The documentation in `docs/` may still contain older references. This README
  reflects the Terraform files currently in the repository.

## Useful Documentation

- [Architecture notes](docs/architecture.md)
- [Deployment guide](docs/deployment.md)
- [API documentation](docs/api_documentation.md)
- [Troubleshooting guide](docs/troubleshooting.md)

## License

This project is licensed under the MIT License. See [LICENSE](LICENSE).
