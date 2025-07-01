# Terraform State Migration Action

A GitHub Action designed to assist with Terraform state migrations from monolith to component/de-coupled repository design using [tfmigrate](https://github.com/minamijoyo/tfmigrate). It supports plan action to assist with configuration validation prior to running an apply action.

## Purpose

This action is designed to:
- Safely migrate Terraform resources between states
- Move resources between different Terraform configurations
- Refactor Terraform code while preserving existing infrastructure
- Handle complex state migrations in a controlled, repeatable manner

## Features

- **Component Migration**: Supports migrating resources from a single monolith structure into sub component directories
- **AWS Integration**: Configures AWS credentials for S3 backend access
- **Multi-version Support**: Supports specific Terraform and tfmigrate versions
- **Flexible Backend Configuration**: Allows backend substitution to allow dynamic backend configuration creation
- **Plan and Apply Modes**: Supports "plan" mode to test configuration before making changes to state

## Usage

### Basic Example

```yaml
name: Terraform Migration
on:
  workflow_dispatch:

jobs:
  migrate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Terraform Migration
        uses: ./
        with:
          aws_access_key_id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws_secret_access_key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws_region: "us-east-1"
          terraform_version: "1.5.0"
          terraform_working_dir: "./terraform"
          tfmigrate_file: "./migrations/move_resources.hcl"
          tfmigrate_action: "plan"
```

### Complete Example with S3 Backend

```yaml
name: Terraform Migration
on:
  workflow_dispatch:

jobs:
  migrate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Terraform Migration
        uses: ./
        with:
          aws_access_key_id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws_secret_access_key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws_region: "us-east-1"
          terraform_version: "1.5.0"
          terraform_state_bucket: "my-terraform-state"
          terraform_state_key: "prod/terraform.tfstate"
          terraform_state_region: "us-east-1"
          terraform_working_dir: "./terraform"
          tfmigrate_version: "0.3.17"
          tfmigrate_file: "./migrations/move_resources.hcl"
          tfmigrate_action: "apply"
          component_directory: "./terraform/components/new-component"
```

## Inputs

### Required Inputs

| Input | Description |
|-------|-------------|
| `aws_access_key_id` | AWS Access Key ID for authentication |
| `aws_secret_access_key` | AWS Secret Access Key for authentication |
| `aws_region` | AWS Region to perform Terraform operations |
| `terraform_version` | Terraform version to install and use |
| `terraform_working_dir` | The Terraform directory to run operations in |

### Optional Inputs

| Input | Description | Default |
|-------|-------------|---------|
| `terraform_state_bucket` | AWS S3 bucket containing the Terraform state file | - |
| `terraform_state_key` | Key/path for the state file within the S3 bucket | - |
| `terraform_state_region` | AWS region for the S3 state bucket | - |
| `tfmigrate_version` | Version of tfmigrate to use | `latest` |
| `tfmigrate_file` | Path to the tfmigrate HCL file for state migrations | - |
| `tfmigrate_action` | tfmigrate action to perform: "plan" or "apply" | `plan` |
| `component_directory` | Component directory to migrate resources to | - |

## Workflow Steps

The action performs the following steps in order:

1. **Configure AWS Credentials**: Sets up AWS authentication using provided credentials
2. **Setup Terraform**: Installs the specified version of Terraform
3. **Install tfmigrate**: Downloads and installs tfmigrate (supports latest or specific version)
4. **Configure Backend**: Creates backend configuration if S3 parameters are provided
5. **Initialize Terraform**: Runs `terraform init` in the working directory
6. **Initialize Component** (optional): Runs `terraform init` in component directory if specified
7. **Execute Migration**: Runs either `tfmigrate plan` or `tfmigrate apply` based on the action parameter

## Prerequisites

### tfmigrate Configuration File

You need to create a tfmigrate HCL configuration file that defines your migration. Example:

```hcl
migration "state" "move_to_component_one" {
  from = "."
  to   = "component_one"
  
  actions = [
    "mv aws_instance.example aws_instance.moved_example",
    "mv aws_security_group.example aws_security_group.moved_example",
  ]
}
```

### Repository Structure

```
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   └── components/
│       └── new-component/
│           ├── main.tf
│           └── variables.tf
├── migrations/
│   └── move_resources.hcl
└── .github/
    └── workflows/
        └── terraform-migration.yml
```

## Security Considerations

- Store AWS credentials as GitHub repository secrets
- Use least-privilege IAM policies for the AWS credentials
- Always run `plan` action first to preview changes before applying
- Consider using OpenID Connect (OIDC) for AWS authentication instead of long-lived credentials (TODO)

## Best Practices

1. **Test First**: Always use `tfmigrate_action: "plan"` to preview changes
2. **Version Control**: Pin specific versions of Terraform and tfmigrate for reproducibility
3. **Backup State**: Ensure your Terraform state is backed up before running migrations
4. **Small Migrations**: Break large migrations into smaller, manageable chunks
5. **Documentation**: Document your migration strategy and any manual steps required

## Troubleshooting

### Common Issues

- **tfmigrate planned diff**: This can occur when the source and or destination configuration contains resources that aren't part of the migration. Remember to **MOVE** the configuration for the resources you are migration, don't copy them

## References

- [tfmigrate Documentation](https://github.com/minamijoyo/tfmigrate)
- [Terraform State Management](https://www.terraform.io/docs/language/state/index.html)
- [AWS Actions Configure Credentials](https://github.com/aws-actions/configure-aws-credentials)