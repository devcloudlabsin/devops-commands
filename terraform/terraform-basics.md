# Terraform — From Basics to Pro (End-to-End Commands, Tools, Testing & Troubleshooting)

This README is a complete operational reference for Terraform (v0.13+; recommendations for v1.x), covering basic → advanced → pro topics with examples, verification steps, and troubleshooting. It includes workflow patterns, state management, modules, testing with Terratest and unit tests, static analysis (tflint, checkov, tfsec), documentation generation (terraform-docs), CI/CD examples, and production best practices.

Table of contents
- Prerequisites & conventions
- Install & verify Terraform and ecosystem tools
- Basic Terraform workflow (init, plan, apply, destroy)
- Configuration language overview (HCL): blocks, arguments, types
- Providers & versions (required_providers, provider blocks, provider installation)
- Variables, outputs, locals, functions, expressions
- Resources & meta-arguments (depends_on, lifecycle, count, for_each)
- Modules (authoring, calling, versioning, composition)
- State (backends, locking, state commands, migration)
- Workspaces & environments
- CLI features: fmt, validate, graph, plan JSON, console
- State operations: import, taint, untaint, mv, rm, refresh
- Provisioners & connection best practices (avoid when possible)
- Secret management & sensitive values
- Policies & policy-as-code (Sentinel, Open Policy Agent)
- Static analysis & security scanning: tflint, checkov, tfsec, terrascan
- Documentation: terraform-docs & auto-generated README
- Testing: unit tests, Terratest (Go), kitchen-terraform, hashicorp/terraform-plugin-testing
- CI/CD examples (GitHub Actions)
- Terraform Cloud / Enterprise features & remote runs
- Best practices & production checklist
- Troubleshooting matrix (common errors & fixes)
- Appendix: useful commands, sample module, sample GitHub Actions workflow

Prerequisites & conventions
- Terraform recommended: v1.x (examples use v1.x features). Check compatibility for your provider versions.
- Store HCL files with extension `.tf` in a directory per Terraform root module.
- Use a remote backend (S3/GCS/Azure/remote) for shared state in production with locking enabled.
- Keep secrets out of plain `*.tfvars` in the repo. Use secret managers (Vault, Azure Key Vault, AWS Secrets Manager) or terraform Cloud variable sets.
- Replace placeholders like <region>, <bucket>, <backend-key>, <module-source>, <workspace> with your values.
- Use explicit provider version constraints via `required_providers` and lock plugin versions via `terraform providers lock` (provider lock file).

Install & verify Terraform and ecosystem tools
- Install Terraform (example for Linux/macOS):
  - Homebrew (macOS/Linux):
    ```
    brew tap hashicorp/tap
    brew install hashicorp/tap/terraform
    ```
  - Script (official):
    ```
    curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
    # or download binary from https://developer.hashicorp.com/terraform/downloads
    ```
- Verify:
  ```
  terraform version
  terraform -v
  ```
- Install common tools:
  - tflint (linting for Terraform)
    ```
    brew install tflint
    tflint --version
    ```
  - terraform-docs (generate README fragments)
    ```
    brew install terraform-docs
    terraform-docs --version
    ```
  - checkov (static security analysis)
    ```
    pip install checkov
    checkov --version
    ```
  - tfsec (security scanner)
    ```
    brew install tfsec
    tfsec --version
    ```
  - terratest (requires Go, see testing section)
  - pre-commit + hooks (see CI section)

Basic Terraform workflow
1. Initialize a working directory:
   ```
   terraform init
   ```
   What it does:
   - Downloads provider plugins.
   - Initializes backend (sets up remote state).
   - Prepares working directory for other commands.
   Verification:
   ```
   ls .terraform
   terraform providers
   ```
   Troubleshooting:
   - "Backend reinitialization required" — run `terraform init -reconfigure` and check backend configuration.

2. Format & validate:
   ```
   terraform fmt -recursive       # formats HCL
   terraform validate             # basic structural checks
   ```
   Notes:
   - `fmt` is safe to run in CI as a pre-check.
   - `validate` does not check provider credentials or remote resources.

3. Plan (preview changes):
   ```
   terraform plan -out=tfplan.binary
   # or human readable
   terraform plan -var-file=prod.tfvars
   terraform plan -input=false -var "image_tag=1.2.3"
   ```
   - `-out` stores a binary plan that can be applied later to ensure the same changes.
   Verification:
   ```
   terraform show -json tfplan.binary | jq .   # inspect plan in JSON
   terraform show tfplan.binary                # human readable
   ```
   Common flags:
   - `-target=resource.address` (use sparingly; temporary targeted applies can drift state)
   - `-parallelism=N` (controls concurrent operations)

4. Apply (make changes):
   ```
   terraform apply "tfplan.binary"      # apply previously saved plan
   terraform apply -var-file=prod.tfvars
   terraform apply -auto-approve
   ```
   Notes:
   - Prefer plan+apply for CI to avoid drift between plan and apply.
   - For interactive runs, omit `-auto-approve` to require confirmation.

5. Destroy (tear down):
   ```
   terraform destroy -var-file=prod.tfvars -auto-approve
   terraform plan -destroy -out=destroy.plan
   terraform apply "destroy.plan"
   ```
   Use carefully; prefer remote/state safeguards and approvals in CI.

Configuration language overview (HCL)
- Main blocks: `provider`, `resource`, `module`, `data`, `variable`, `output`, `locals`, `terraform`.
- Example structure:
  ```hcl
  terraform {
    required_version = ">= 1.0.0"
    required_providers {
      aws = {
        source  = "hashicorp/aws"
        version = "~> 4.0"
      }
    }
    backend "s3" {
      bucket = "my-tf-state"
      key    = "prod/infra/terraform.tfstate"
      region = "us-east-1"
    }
  }

  provider "aws" {
    region = var.region
  }

  variable "region" {
    type    = string
    default = "us-east-1"
  }
  ```
- Types: string, number, bool, list, map, object, tuple — use type constraints to catch errors early.

Providers & versions
- required_providers (in terraform block) pins provider source & version.
- Example:
  ```hcl
  terraform {
    required_providers {
      aws = {
        source  = "hashicorp/aws"
        version = ">= 4.0, < 5.0"
      }
    }
  }
  ```
- Provider configuration:
  ```hcl
  provider "aws" {
    region = var.aws_region
    # shared credentials: environment variables, shared credentials file, or assume role
  }
  ```
- Multiple provider instances / aliases:
  ```hcl
  provider "aws" {
    region = "us-east-1"
    alias  = "use1"
  }

  provider "aws" {
    region = "us-west-2"
    alias  = "usw2"
  }

  resource "aws_instance" "example" {
    provider = aws.use1
    ...
  }
  ```

Variables, outputs, locals, functions
- Variables:
  ```hcl
  variable "instance_count" {
    type    = number
    default = 2
    description = "Number of EC2 instances"
  }
  ```
  - Supply via CLI: `-var 'instance_count=3'` or `-var-file=prod.tfvars` or environment var `TF_VAR_instance_count=3`.
- Outputs:
  ```hcl
  output "alb_dns" {
    value       = aws_lb.app.dns_name
    description = "ALB DNS name"
    sensitive   = false
  }
  ```
- Locals:
  ```hcl
  locals {
    common_tags = {
      team = "platform"
      env  = var.environment
    }
  }
  ```
- Functions: `join`, `split`, `lookup`, `try`, `coalesce`, `format`, `toset`, `tomap`, `jsonencode`, `jsondecode`, etc.

Resources & meta-arguments
- Example resource with count and for_each:
  ```hcl
  resource "aws_instance" "web" {
    for_each = var.instances_map
    ami           = each.value.ami
    instance_type = each.value.type
    tags = merge(local.common_tags, { Name = each.key })
  }
  ```
- Meta-arguments:
  - `count` — index-based creation
  - `for_each` — map/set-based creation with stable keys
  - `depends_on` — force ordering (rare; prefer implicit dependencies)
  - `lifecycle`:
    ```hcl
    lifecycle {
      prevent_destroy = true
      create_before_destroy = true   # helpful for replacement resources
      ignore_changes = [ "tags" ]    # ignore certain upstream-imposed changes
    }
    ```

Modules (authoring, composition, versioning)
- Use modules to encapsulate reusable infrastructure. Store modules in:
  - Local path: `module "db" { source = "./modules/postgres" }`
  - Git: `source = "git::https://github.com/org/repo.git//modules/postgres?ref=v1.2.0"`
  - Registry: `source = "app.terraform.io/org/my-module/aws/postgres"`
  - Registry (public): `source = "terraform-aws-modules/vpc/aws"`
- Example module usage:
  ```hcl
  module "vpc" {
    source  = "terraform-aws-modules/vpc/aws"
    version = "3.14.2"
    name    = "prod-vpc"
    cidr    = "10.0.0.0/16"
    azs     = ["us-east-1a", "us-east-1b"]
  }
  ```
- Module best practices:
  - Keep modules small, focused, and parameterized.
  - Define `inputs` (variables) and `outputs` clearly and document via `README.md` + `terraform-docs`.
  - Make modules idempotent and safe to re-run.
  - Pin module versions and use semantic versioning.
  - Publish public modules to the Terraform Registry for reuse.
  - Provide example usage in `examples/` directory.
- Example module structure:
  ```
  modules/postgres/
  ├── main.tf
  ├── variables.tf
  ├── outputs.tf
  ├── README.md
  └── examples/
      └── simple/
  ```

State (backends, locking, state commands, migration)
- Choose a backend for production: S3 (AWS), GCS (GCP), azurerm (Azure), remote (Terraform Cloud/Enterprise).
- S3 backend example with DynamoDB locking:
  ```hcl
  terraform {
    backend "s3" {
      bucket         = "my-tf-state"
      key            = "org/env/terraform.tfstate"
      region         = "us-east-1"
      dynamodb_table = "terraform-locks"
      encrypt        = true
    }
  }
  ```
- Initialize after backend config changes:
  ```
  terraform init -reconfigure
  ```
- Important state commands:
  - `terraform state list` — list resources in state
  - `terraform state show <address>`
  - `terraform state mv <from> <to>` — move resource in state (useful for refactoring module paths)
  - `terraform state rm <address>` — remove from state (resource left in cloud)
  - `terraform import <address> <id>` — import existing resource into state
- State migration example (move between backends):
  - Initialize with new backend and run `terraform init -migrate-state -reconfigure`.
- State locking:
  - Ensure locking (DynamoDB/GCS/remote) to avoid concurrent writes.
  - For SSH/CLI users, `terraform init` must be run before `plan`/`apply` to get backend lock.

Workspaces & environments
- Terraform workspaces provide multiple state instances using the same configuration.
  ```
  terraform workspace list
  terraform workspace new staging
  terraform workspace select production
  terraform workspace show
  ```
- Use workspaces for small-scale environment separation (dev/staging/prod), but prefer separate root modules/repos for bigger teams and stronger isolation.
- Example usage pattern:
  ```
  backend_key = "org/${terraform.workspace}/infra/terraform.tfstate"
  ```
- Terraform Cloud workspaces are more powerful (variable sets, VCS integration, run triggers).

CLI features: fmt, validate, graph, plan JSON, console
- Format:
  ```
  terraform fmt -recursive
  ```
- Validate:
  ```
  terraform validate
  ```
- Graph:
  ```
  terraform graph | dot -Tpng > graph.png
  ```
  - Visualize dependency graph of resources.
- Plan JSON:
  ```
  terraform plan -out=tfplan.binary
  terraform show -json tfplan.binary > plan.json
  ```
  - Use plan JSON for automated checks / policy validation / tests.
- Console:
  ```
  terraform console
  > var.foo
  > length(aws_instance.web)
  ```
  - Helpful for evaluating expressions against the current config and state.

State operations: import, taint, untaint, mv, rm, refresh
- Import:
  ```
  terraform import aws_instance.web i-0abcd1234
  ```
  - Update HCL to match the imported resource after import.
- Taint & untaint:
  ```
  terraform taint aws_instance.web
  terraform untaint aws_instance.web
  ```
  - Mark resources for recreation on next `apply`.
- Move state item:
  ```
  terraform state mv module.old.aws_instance.web module.new.aws_instance.web
  ```
  - Useful when refactoring modules or resource addresses.
- Refresh state (reconcile state with real infra):
  ```
  terraform refresh   # deprecated in some contexts; prefer plan/apply
  terraform plan -refresh=true
  ```

Provisioners & connection best practices
- Provisioners (remote-exec, local-exec) run on resource creation and are considered an escape hatch.
  - Example (avoid heavy reliance):
    ```hcl
    provisioner "remote-exec" {
      inline = [
        "sudo apt update",
        "sudo apt -y install nginx",
      ]
      connection {
        type        = "ssh"
        host        = self.public_ip
        user        = "ec2-user"
        private_key = file(var.private_key_path)
      }
    }
    ```
  - Prefer configuration management/immutable images (Packer), or use cloud-init/user-data instead of provisioning at apply-time.

Secret management & sensitive values
- Mark outputs/variables as `sensitive = true` in Terraform 0.14+ to suppress logs.
  ```hcl
  output "db_password" {
    value     = aws_secretsmanager_secret_version.db_secret.secret_string
    sensitive = true
  }
  ```
- Avoid committing secrets in `*.tfvars` to VCS. Use secret stores and fetch them during CI runs or runtime.
- Terraform Cloud variable sets allow encrypted storage of sensitive variables.

Policies & policy-as-code
- Terraform Cloud / Enterprise supports Sentinel policies for policy enforcement.
- Open Policy Agent (OPA) Gatekeeper or Conftest can be used in CI to validate plan JSON with Rego or other rules.
- Example Conftest usage:
  ```
  terraform plan -out=tfplan
  terraform show -json tfplan > plan.json
  conftest test plan.json
  ```

Static analysis & security scanning
- tflint (linting):
  - Install and run:
    ```
    tflint --init    # initialize provider-specific plugins
    tflint
    ```
  - Configure via `.tflint.hcl` to enable/disable rules and set provider configs.
  - Use with GitHub Actions to fail PR on policy violations.
- checkov (static scanning for misconfigurations):
  ```
  checkov -d .          # scans current dir of terraform files
  checkov -f main.tf    # or scan plan JSON: checkov -f plan.json --bc
  ```
- tfsec (security scanning):
  ```
  tfsec .
  ```
- terrascan (another option)
- Best practice: run multiple scanners (tflint for style, checkov/tfsec for security) in CI.

Documentation: terraform-docs & auto-generated README
- `terraform-docs` automatically generates docs for variables, outputs, providers, and resources.
  - Example:
    ```
    terraform-docs markdown table ./ > README.md
    ```
  - Integrate with CI to regenerate module README and fail if docs are out of date:
    - Use `terraform-docs` in `--mode diff` or compare generated output to committed README.
- Example `Makefile` targets:
  ```makefile
  docs:
    terraform-docs markdown table ./ > README.md

  fmt:
    terraform fmt -recursive
  ```

Testing: unit tests, Terratest (Go), kitchen-terraform
- Testing pyramid:
  - Unit/Static analysis: tflint, checkov, tfsec, terraform validate
  - Integration tests: Terratest (Go) or kitchen-terraform
  - End-to-end / staging runs in isolated environments
- Terratest (Go) example:
  - Install Go (>=1.18) and Terratest:
    ```
    go get github.com/gruntwork-io/terratest/modules/terraform
    go test ./test -v
    ```
  - Example test file: test/terraform_test.go
    ```go
    package test

    import (
      "testing"
      "github.com/gruntwork-io/terratest/modules/terraform"
      "github.com/stretchr/testify/assert"
    )

    func TestTerraformAwsExample(t *testing.T) {
      t.Parallel()

      opts := &terraform.Options{
        TerraformDir: "../examples/simple",
        Vars: map[string]interface{}{
          "instance_count": 1,
        },
      }
      defer terraform.Destroy(t, opts)
      terraform.InitAndApply(t, opts)

      // Example output assertion
      ip := terraform.Output(t, opts, "instance_public_ip")
      assert.NotEmpty(t, ip)
    }
    ```
  - Best practices:
    - Keep tests idempotent and fast; use smaller AWS resources or local backends (like localstack) for speed.
    - Use tags to clean up test resources in case of interruption.
- Unit testing modules with checkov/kubetest or using plan JSON assertions:
  - Generate plan JSON and assert expected content via small scripts:
    ```
    terraform plan -out=tfplan.binary
    terraform show -json tfplan.binary > plan.json
    # then use jq/python to assert keys/values
    ```
- `kitchen-terraform` (Ruby) / `inspec` for infrastructure tests (especially for OS-level tests).
- `terraform validate` + `terraform fmt` + `tflint` + `terraform-docs` form a basic unit-level check pipeline.

CI/CD examples (GitHub Actions)
- Example PR pipeline (lint, fmt, plan):
  ```yaml
  name: Terraform Pull Request

  on:
    pull_request:
      paths:
        - 'modules/**'
        - 'services/**'
        - '*.tf'
        - '*.tfvars'

  jobs:
    fmt:
      runs-on: ubuntu-latest
      steps:
        - uses: actions/checkout@v4
        - name: Setup Terraform
          uses: hashicorp/setup-terraform@v2
          with:
            terraform_version: 1.4.0
        - name: terraform fmt
          run: terraform fmt -check -recursive

    lint:
      runs-on: ubuntu-latest
      needs: fmt
      steps:
        - uses: actions/checkout@v4
        - name: Setup tflint
          uses: wata727/tflint-action@v4
        - name: Run tflint
          run: tflint --init && tflint

    plan:
      runs-on: ubuntu-latest
      needs: [fmt, lint]
      steps:
        - uses: actions/checkout@v4
        - name: Setup Terraform
          uses: hashicorp/setup-terraform@v2
        - name: Terraform Init
          run: terraform init -input=false
        - name: Terraform Plan
          run: terraform plan -out=tfplan.binary -input=false
        - name: Upload plan artifact
          uses: actions/upload-artifact@v4
          with:
            name: tfplan
            path: tfplan.binary
  ```
- Example apply job should run only on approved merges to `main` and with proper secrets:
  - Use protected branches, require reviews, require CI to pass.
  - Use terraform Cloud remote runs for safer apply workflows.

Terraform Cloud / Enterprise features
- Remote state, remote runs, workspace variables, VCS integration, policies.
- Remote runs mean plan and apply happen in a centralized execution environment.
- Use run triggers, plan-only runs, and policy checks.
- Use variable sets to share sensitive variables across workspaces.

Best practices & production checklist
- Use a remote backend with locking (DynamoDB/GCS/remote).
- Pin provider & module versions.
- Keep modules small and maintain a module registry.
- Keep secrets out of VCS; use secret managers or Terraform Cloud variable sets.
- Run `terraform fmt`, `terraform validate`, `tflint`, `checkov` in CI before plan.
- Use `terraform plan -out` in CI and require manual or automated apply in protected pipelines OR use Terraform Cloud remote runs.
- Use state isolation for environments (separate backends or workspace-per-environment).
- Include lifecycle rules and `prevent_destroy` where accidental deletion would be catastrophic.
- Write tests (Terratest) for critical infrastructure.
- Automate docs generation with `terraform-docs` and include `examples/`.
- Backup state and test state restores regularly.
- Use tags and naming conventions consistently for cost tracking.
- Implement policy as code and run it on plan JSON.

Troubleshooting matrix (common errors & fixes)
- Error: "Error: Failed to install provider"
  - Fix: Run `terraform init -upgrade` or verify network/proxy and required_providers in `terraform` block. Check provider source & version.
- Error: "No valid credential sources found for AWS Provider"
  - Fix: Ensure AWS creds are set via env vars, shared credentials file, or assume role setup. Use `AWS_PROFILE` or `provider` block with credentials.
- Error: "Lock table not found" (DynamoDB)
  - Fix: Create the DynamoDB table used for locking or adjust backend config. Ensure partition key named `LockID`.
- Error: "Resource still exists" on destroy
  - Fix: Check external dependencies (cloud provider) and remove manually or use `terraform state rm` if resource was removed out-of-band.
- Error: "Address already in use" for port in provisioner
  - Fix: Use unique ports or avoid provisioners; pre-bake images instead.
- Error: "Cycle: ... detected"
  - Fix: Avoid circular dependencies. Introduce `depends_on` only to break cycles if necessary; prefer to refactor resources.
- Error: "The given key does not exist" when reading remote state
  - Fix: Check backend key path. Run `terraform init -reconfigure` to correct backend.

Appendix: useful commands & quick reference
- Init:
  ```
  terraform init
  terraform init -reconfigure
  terraform init -upgrade
  ```
- Format & validate:
  ```
  terraform fmt -recursive
  terraform validate
  ```
- Plan & apply:
  ```
  terraform plan -out=tfplan.binary
  terraform plan -var-file=prod.tfvars
  terraform apply tfplan.binary
  terraform apply -auto-approve
  terraform destroy -auto-approve
  ```
- State:
  ```
  terraform state list
  terraform state show aws_instance.web
  terraform state mv old.address new.address
  terraform import aws_instance.web i-123456
  terraform state rm aws_instance.web
  ```
- Workspaces:
  ```
  terraform workspace list
  terraform workspace select staging
  terraform workspace new test
  ```
- Providers:
  ```
  terraform providers
  terraform providers lock -platform=linux_amd64 -platform=darwin_amd64
  ```
- Advanced:
  ```
  terraform graph | dot -Tsvg > graph.svg
  terraform show -json tfplan.binary > plan.json
  terraform console
  ```
- Static analysis:
  ```
  tflint
  terraform-docs markdown table ./ > README.md
  checkov -d .
  tfsec .
  ```

Sample module (simple AWS VPC) — usage example
- Root: `examples/vpc/main.tf`
  ```hcl
  module "vpc" {
    source  = "git::https://github.com/example-org/terraform-modules.git//vpc?ref=v1.0.0"
    name    = "prod"
    cidr    = "10.0.0.0/16"
    azs     = ["us-east-1a", "us-east-1b"]
    public_subnets  = ["10.0.1.0/24", "10.0.2.0/24"]
  }

  output "vpc_id" {
    value = module.vpc.vpc_id
  }
  ```
- Generate docs:
  ```
  terraform-docs markdown table ./examples/vpc > examples/vpc/README.md
  ```

Sample Terratest skeleton (Go)
- `test/terraform_test.go`
  ```go
  package test

  import (
    "testing"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
  )

  func TestExampleVPC(t *testing.T) {
    t.Parallel()
    opts := &terraform.Options{
      TerraformDir: "../examples/vpc",
    }
    defer terraform.Destroy(t, opts)
    terraform.InitAndApply(t, opts)
    vpcID := terraform.Output(t, opts, "vpc_id")
    assert.NotEmpty(t, vpcID)
  }
  ```
- Run:
  ```
  go test ./test -v
  ```

Pre-commit and Git Hooks (recommended)
- Example `.pre-commit-config.yaml`:
  ```yaml
  repos:
    - repo: https://github.com/antonbabenko/pre-commit-terraform
      rev: v1.82.0
      hooks:
        - id: terraform_fmt
        - id: terraform_validate
        - id: terraform_tflint
        - id: terraform_docs
  ```
- Install:
  ```
  pip install pre-commit
  pre-commit install
  pre-commit run --all-files
  ```

When to use Terraform Cloud / CI remote runs
- Use remote execution for centralized credential handling, collaboration, locking, and policy enforcement. Remote runs keep sensitive provider credentials out of developer machines and restrict who can approve applies.

- Generate GitHub Actions workflows for linting, plan (PR), and apply (merge) including state handling with S3/DynamoDB.

Tell me which of those you'd like next and specify target cloud provider(s) and Terraform version (if you need exact feature coverage).  
