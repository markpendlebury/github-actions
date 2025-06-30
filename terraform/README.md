# Terraform Run Action

A GitHub Action that runs Terraform plan or apply operations against AWS accounts with support for both IAM role assumption and access key authentication.

## Features

- ✅ Terraform plan and apply operations
- ✅ AWS authentication via IAM roles or access keys
- ✅ Flexible S3 backend configuration
- ✅ Change detection with structured outputs
- ✅ Automatic dependency installation (jq)
- ✅ Input validation and error handling
- ✅ Support for Terraform variables via TF_VAR_ environment variables

## Inputs

### Required Inputs

| Input | Description | Default |
|-------|-------------|---------|
| `aws_region` | AWS Region to perform Terraform operations | - |
| `terraform_version` | Terraform version to use | - |
| `terraform_working_dir` | Directory containing Terraform configuration | - |

### Authentication Inputs (at least one method required)

| Input | Description | Required |
|-------|-------------|----------|
| `aws_role` | AWS IAM Role ARN to assume | false |
| `aws_access_key_id` | AWS Access Key ID | false |
| `aws_secret_access_key` | AWS Secret Access Key | false |

### Optional Inputs

| Input | Description | Default |
|-------|-------------|---------|
| `terraform_action` | Action to perform: `plan` or `apply` | `plan` |
| `terraform_state_bucket` | S3 bucket for Terraform state | - |
| `terraform_state_key` | S3 object key/path for state file | - |
| `terraform_state_region` | AWS region for S3 state bucket | - |
| `variables` | Comma-separated list of variables to export as TF_VAR_ environment variables | - |

## Outputs

| Output | Description |
|--------|-------------|
| `terraform_plan_exitcode` | Exit code from terraform plan (0=no changes, 2=changes detected) |
| `terraform_apply_output` | JSON output from terraform apply |
| `has_changes` | Boolean indicating if plan detected changes |

## Usage Examples

### Basic Plan with IAM Role

```yaml
- name: Terraform Plan
  uses: ./path/to/terraform-action
  with:
    aws_role: arn:aws:iam::123456789012:role/TerraformRole
    aws_region: us-east-1
    terraform_version: 1.5.0
    terraform_working_dir: ./infrastructure
    terraform_action: plan
```

### Apply with Access Keys and S3 Backend

```yaml
- name: Terraform Apply
  uses: ./path/to/terraform-action
  with:
    aws_access_key_id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws_secret_access_key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws_region: us-west-2
    terraform_version: 1.5.0
    terraform_working_dir: ./terraform
    terraform_action: apply
    terraform_state_bucket: my-terraform-state-bucket
    terraform_state_key: production/terraform.tfstate
    terraform_state_region: us-west-2
```

### Plan with Terraform Variables

```yaml
- name: Terraform Plan
  uses: ./path/to/terraform-action
  with:
    aws_role: ${{ secrets.AWS_ROLE }}
    aws_region: us-east-1
    terraform_version: 1.5.0
    terraform_working_dir: ./infrastructure
    terraform_action: plan
    variables: "environment=production,instance_type=t3.medium,enable_monitoring=true"
```

### Conditional Apply Based on Plan Changes

```yaml
- name: Terraform Plan
  id: plan
  uses: ./path/to/terraform-action
  with:
    aws_role: ${{ secrets.AWS_ROLE }}
    aws_region: us-east-1
    terraform_version: 1.5.0
    terraform_working_dir: ./terraform
    terraform_action: plan

- name: Terraform Apply
  if: steps.plan.outputs.has_changes == 'true'
  uses: ./path/to/terraform-action
  with:
    aws_role: ${{ secrets.AWS_ROLE }}
    aws_region: us-east-1
    terraform_version: 1.5.0
    terraform_working_dir: ./terraform
    terraform_action: apply
```

### Partial Backend Configuration

Use when you have some backend settings in your Terraform configuration but want to override specific values:

```yaml
- name: Terraform Plan
  uses: ./path/to/terraform-action
  with:
    aws_role: ${{ secrets.AWS_ROLE }}
    aws_region: us-east-1
    terraform_version: 1.5.0
    terraform_working_dir: ./terraform
    # Only override the bucket, keep key and region from terraform config
    terraform_state_bucket: ${{ secrets.STATE_BUCKET }}
```

## Authentication Methods

### IAM Role (Recommended)

Configure OIDC trust relationship and use IAM roles:

```yaml
permissions:
  id-token: write
  contents: read

- name: Terraform Plan
  uses: ./path/to/terraform-action
  with:
    aws_role: arn:aws:iam::123456789012:role/GitHubActionsRole
    # ... other inputs
```

### Access Keys

Store AWS credentials as repository secrets:

```yaml
- name: Terraform Plan
  uses: ./path/to/terraform-action
  with:
    aws_access_key_id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws_secret_access_key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    # ... other inputs
```

## Backend Configuration

The action supports flexible S3 backend configuration. You can provide any combination of:

- `terraform_state_bucket`: S3 bucket name
- `terraform_state_key`: Object key/path within bucket  
- `terraform_state_region`: AWS region for the bucket

This allows you to:
- Override only specific backend parameters
- Keep some settings in your Terraform config, others as secrets
- Use different backend configurations per environment

## Terraform Variables

The action supports passing variables to Terraform via the `variables` input. Variables are:

- Provided as a comma-separated string in `KEY=VALUE` format
- Automatically prefixed with `TF_VAR_` if not already present
- Exported as environment variables before running Terraform commands
- Available to both plan and apply operations

### Variable Examples

```yaml
# Simple variables
variables: "environment=prod,region=us-east-1"

# Mixed variable types
variables: "instance_count=3,enable_ssl=true,app_name=myapp"

# Variables with existing TF_VAR_ prefix (will not be double-prefixed)
variables: "TF_VAR_custom_var=value,normal_var=value2"

# Using secrets for sensitive variables
variables: "db_password=${{ secrets.DB_PASSWORD }},api_key=${{ secrets.API_KEY }}"
```

### Variable Usage in Terraform

In your Terraform configuration, declare variables as usual:

```hcl
variable "environment" {
  description = "Environment name"
  type        = string
}

variable "instance_count" {
  description = "Number of instances"
  type        = number
  default     = 1
}
```

## Error Handling

The action includes comprehensive validation:

- ✅ Ensures either AWS role or access keys are provided
- ✅ Validates access key pairs are complete
- ✅ Automatically installs required dependencies (jq)
- ✅ Provides clear error messages for common issues

