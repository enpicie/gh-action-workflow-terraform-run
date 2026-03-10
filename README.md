# gh-action-workflow-terraform-run

Reusable GitHub Actions composite action to conduct Terraform runs. Accepts project-specific parameters and uses OpenTofu to plan and apply Terraform configurations in AWS via OIDC authentication.

## Prerequisites

- AWS account bootstrapped with the [tf-backend](./tf-backend.yml) stack (S3 state bucket, DynamoDB lock table, OIDC provider) provisioned via this repo
- An IAM role ARN scoped to the resources your project needs — see [aws-tf-iam-roles](https://github.com/enpicie/aws-tf-iam-roles) for available roles
- A Terraform configuration with an empty S3 backend block:

```hcl
terraform {
  backend "s3" {}
}
```

## Usage

```yaml
name: Deploy Infrastructure

on:
  push:
    branches:
      - main
  pull_request:

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: your-org/gh-action-workflow-terraform-run@v1.0.0
        with:
          aws_role_arn: ${{ vars.TF_ROLE_APIGW_LAMBDA }}
          state_key: projects/my-app/prod/infra.tfstate
```

## Inputs

| Name           | Description                                                                | Required | Default     |
| -------------- | -------------------------------------------------------------------------- | -------- | ----------- |
| `aws_role_arn` | IAM role ARN to assume via OIDC — controls what this project can provision | Yes      |             |
| `state_key`    | S3 path for this project's state file (see [State Key](#state-key))        | Yes      |             |
| `aws_region`   | AWS region                                                                 | No       | `us-east-2` |
| `tf_directory` | Path to Terraform configuration directory                                  | No       | `.`         |
| `tf_vars`      | Terraform input variables (see [Passing Variables](#passing-variables))    | No       | `''`        |
| `apply`        | Whether to apply after planning — set to `'false'` for plan-only on PRs    | No       | `'true'`    |

## Passing Variables

Use `tf_vars` to pass values to your Terraform input variables. Each line must be a `TF_VAR_*` environment variable assignment. OpenTofu automatically maps `TF_VAR_foo` to `var.foo` in your Terraform configuration.

```yaml
- uses: your-org/gh-action-workflow-terraform-run@v1.0.0
  with:
    aws_role_arn: ${{ vars.TF_ROLE_APIGW_LAMBDA }}
    state_key: projects/my-app/prod/infra.tfstate
    tf_vars: |
      TF_VAR_app_name=my-app
      TF_VAR_environment=prod
      TF_VAR_instance_class=t3.micro
```

For sensitive values, reference GitHub Actions secrets inline:

```yaml
tf_vars: |
  TF_VAR_app_name=my-app
  TF_VAR_db_password=${{ secrets.DB_PASSWORD }}
```

Each `TF_VAR_*` key must exactly match a declared variable in your Terraform configuration:

```hcl
variable "app_name" {}
variable "environment" {}
variable "db_password" {
  sensitive = true
}
```

## State Key

The `state_key` is the S3 path where this project's Terraform state file is stored. It must be unique per project and consistent across runs. The recommended convention is:

```
projects/{app-name}/{environment}/configname.tfstate
```

Configs should be named for their use case, which should typically be the app name. For example:

```
projects/my-app/prod/myapp.tfstate
projects/my-app/dev/myapp.tfstate
```

## Plan vs Apply

By default the action runs both plan and apply. To run plan-only on pull requests and apply only on merge to main:

```yaml
- uses: your-org/gh-action-workflow-terraform-run@v1.0.0
  with:
    aws_role_arn: ${{ vars.TF_ROLE_APIGW_LAMBDA }}
    state_key: projects/my-app/prod/infra.tfstate
    apply: ${{ github.ref == 'refs/heads/main' && 'true' || 'false' }}
```
