# Terraform Tools for AWS — Complete Practical Guide (2025)

This document is a practical, hands‑on reference for the most important tools used with Terraform on AWS in real projects. It covers installation, configuration, example commands, CI/CD integrations, recommended patterns and real-world tips for:

- IaC drift detection
- automatic tagging
- importing existing AWS infra into Terraform
- linting and static analysis
- cost estimation
- documentation generation
- policy as code
- security & compliance scanning
- testing infrastructure

Tools covered:
1. driftctl
2. Terratag
3. Terraformer
4. TFLint
5. Infracost
6. terraform-docs
7. Terratest
8. Open Policy Agent (OPA)
9. tfsec
10. Checkov

---

Table of contents
- Overview & recommended workflow
- Tool-by-tool detailed documentation (install, usage, examples)
  - driftctl
  - Terratag
  - Terraformer
  - TFLint
  - Infracost
  - terraform-docs
  - Terratest
  - OPA
  - tfsec
  - Checkov
- CI/CD examples (GitHub Actions snippets)
- Config and sample files (tflint, infracost, terraform-docs, OPA, tfsec)
- Best practices, troubleshooting, remediation and team processes

---

Overview & recommended workflow
- Local developer / PR pipeline:
  1. terraform fmt
  2. tflint (static lint)
  3. terraform validate
  4. tfsec / Checkov (security & compliance static checks)
  5. terraform plan -out=plan.tfplan
  6. terraform show -json plan.tfplan > plan.json
  7. OPA / rego rules evaluate plan.json
  8. Infracost diff (show cost impact of PR)
  9. Terratest (integration tests) in promotion pipelines
  10. Merge and apply in controlled environment
- Post-apply / continuous monitoring:
  - driftctl scheduled scans (to detect out-of-band changes)
  - tagging enforcement (terratag, pre-commit hooks, or pipeline steps)
  - periodic tfsec/checkov audits for drifted resources or new findings

Notes:
- The pipeline should fail fast on lint/security issues.
- Use plan.json for policy-as-code and automated checks (OPA, custom scripts).
- Report Infracost diffs in PRs as comments so reviewers can see cost impact.

---

1) driftctl — Detect AWS Terraform drift
What it is: driftctl compares Terraform state (and optionally cloud provider state) with the real resources in AWS and reports missing, unmanaged, or changed resources.

Install (Linux example)
```bash
curl -LO https://github.com/snyk/driftctl/releases/latest/download/driftctl_linux_amd64
chmod +x driftctl_linux_amd64
sudo mv driftctl_linux_amd64 /usr/local/bin/driftctl
```

Quick scan
```bash
# runs using default AWS credentials
driftctl scan
# save output
driftctl scan --output json://drift.json
```

Using with a remote backend (S3 + state file)
```bash
driftctl scan --from tfstate+s3://my-bucket/path/to/terraform.tfstate
```

Interpretation
- missing: resource exists in state but not in AWS
- managed: resource exists in AWS but not in Terraform state (unmanaged)
- changed: attribute drift between state and actual

Common remediation flow
1. Inspect drift.json to identify differences.
2. If resource was deleted accidentally: recreate or import replacement.
3. If resource was added manually and should be managed: use terraform import or re-create in code + import state.
4. If resource was intentionally changed manually, reconcile Terraform to match or change Terraform and plan/apply.

driftctl tips & CI
- Run scheduled daily/weekly scans (e.g., GitHub Action or cron) and post findings to Slack or ticketing system.
- Use driftctl CI mode with `--exit-code` options or evaluate output JSON.

Example: only check specific resource types
```bash
driftctl scan --filter "Type==aws_s3_bucket || Type==aws_security_group"
```

---

2) Terratag — Automated tagging of Terraform files
What it is: apply tags en masse to Terraform resource blocks (useful temporarily to add tags in many files or to enforce tag defaults).

Install (macOS Homebrew example)
```bash
brew tap env0/tap
brew install terratag
```

Basic usage
```bash
terratag -dir ./ -tags environment=prod,owner=devops,costcenter=banking
```

Output
- Modifies Terraform files by inserting or merging tags blocks.

Best practices
- Prefer central tagging patterns (modules that accept tags and merge them) over modifying many files directly.
- Use terratag for quick remediation or as a one-time migration; keep tags in modules for long-term maintainability.
- Keep tag keys consistent (lowercase, hyphen or underscore rules).

Example: adding tags only to files in modules/
```bash
terratag -dir ./modules -tags environment=prod
```

---

3) Terraformer — Import existing AWS infra into Terraform
What it is: helps generate Terraform files and state by inspecting existing cloud resources.

Install (Homebrew)
```bash
brew install terraformer
```

Import example
```bash
# Import VPC and related resources in ap-south-1
terraformer import aws --resources=vpc,subnet,route_table,security_group --regions=ap-south-1 --profile my-profile
```

Output
- Generates *.tf files and a terraform.tfstate representing discovered resources.

Workflow to adopt terraformer output
1. Run terraformer in a safe environment (with limited AWS credentials or read-only where appropriate).
2. Inspect generated `.tf` files: they often need cleanup (naming, variables, modules).
3. Move relevant parts into modules and parameterize values.
4. Use `terraform state` commands and `terraform import` for targeted imports when you want more control.

Notes & pitfalls
- Terraformer often produces a lot of provider-specific metadata. Clean and refactor before committing.
- Validate that generated resources align with your account policies, tags, and naming conventions.

---

4) TFLint — Linter for Terraform
What it is: detects mistakes, style or deprecated usages in HCL/Terraform code.

Install
```bash
brew install tflint
# or download binary: https://github.com/terraform-linters/tflint
```

Basic run
```bash
tflint
```

Sample .tflint.hcl to enforce tags and AWS rules
```hcl
plugin "aws" {
  enabled = true
  version = "0.38.0"
}

rule "aws_instance_invalid_type" {
  enabled = true
}

# enforce tag presence by custom rule or external checks (can integrate with tflint rules or custom scripts)
```

CI usage
- Run tflint in PR builds with `tflint --init` then `tflint`.
- Fail the build on any ERROR level finding.

Example detection
- Invalid instance type, typo in attribute names, deprecated arguments.

Tip: enable tflint plugins for provider-specific checks (aws plugin).

---

5) Infracost — Cost estimation before apply
What it is: estimates cloud costs from Terraform plan and provides diffs for PRs.

Install
```bash
brew install infracost
infracost configure  # follow prompts to configure API key and pricing region
```

Basic usage
```bash
infracost breakdown --path .
# For CI, produce a diff
infracost diff --path . --format github-comment
```

Using with plan JSON
```bash
terraform plan -out plan.tfplan
terraform show -json plan.tfplan > plan.json
infracost breakdown --path . --terraform-plan plan.json
```

GitHub Actions integration
- Use the official Infracost GitHub Action to comment on PRs with a cost delta.
- Provide INFRACOST_API_KEY in repo secrets.

Example output (human-friendly):
| Resource | Old | New |
|---|---:|---:|
| aws_db_instance.db | $0 | $780/mo |

Best practices
- Keep pricing region consistent.
- Use module-level tagging to map costs where applicable.
- Show both monthly and hourly where needed.

---

6) terraform-docs — Auto-generate module README
What it is: generates markdown docs for Terraform modules (inputs, outputs, providers, resources).

Install
```bash
brew install terraform-docs
# or use prebuilt binary
```

Usage
```bash
terraform-docs markdown table ./module-dir > ./module-dir/README.md
```

Configuration (example .terraform-docs.yml)
```yaml
formatter: markdown table
settings:
  outputs:
    show_required: true
  inputs:
    show_default: true
```

Integration
- Run as a pre-commit hook or in CI to ensure module READMEs are synchronized with code.
- Optionally run `terraform-docs` in a "doc update" job to auto-commit README changes to a docs branch.

---

7) Terratest — Test Terraform with Go
What it is: testing framework to provision resources and assert behavior (integration tests in Go).

Install
- Terratest is a Go library. You need Go installed and a test file.

Example test (basic)
```go
package test

import (
  "testing"
  "github.com/gruntwork-io/terratest/modules/terraform"
  "github.com/stretchr/testify/assert"
)

func TestS3Bucket(t *testing.T) {
  t.Parallel()
  terraformOptions := &terraform.Options{
    TerraformDir: "../examples/s3",
  }
  defer terraform.Destroy(t, terraformOptions)
  terraform.InitAndApply(t, terraformOptions)

  bucketID := terraform.Output(t, terraformOptions, "bucket_id")
  assert.NotEmpty(t, bucketID)
}
```

Best practices
- Use ephemeral accounts or dedicated test AWS accounts.
- Tear down resources in defer with `terraform.Destroy`.
- Keep integration tests targeted and fast; use smaller test fixtures for CI and larger tests in nightly runs.

Security note
- Do not store long-lived credentials in test code; use environment variables and short-lived tokens.

---

8) Open Policy Agent (OPA) — Policy as Code
What it is: evaluate custom policies against plan.json using Rego rules.

Rego example: block public S3 buckets
```rego
package terraform.deny

deny[msg] {
  input.resource_changes[_].type == "aws_s3_bucket"
  some i
  rc := input.resource_changes[_]
  rc.change.after.acl == "public-read"
  msg = sprintf("Public S3 bucket is not allowed: %v", [rc.address])
}
```

Run example
```bash
terraform plan -out plan.tfplan
terraform show -json plan.tfplan > plan.json

opa eval --input plan.json --data mypolicy.rego "data.terraform.deny" --format pretty
```

CI usage
- Fail the job if OPA produces any deny messages.
- Maintain a central polices repo and version them.

Tips
- Use plan.json as input to check proposed infrastructure changes (more precise than static HCL checks).
- Combine OPA checks with team-approved exceptions and an allowlist if necessary.

---

9) tfsec — Terraform security scanner
What it is: static analysis tool that finds security issues in Terraform code.

Install
```bash
brew install tfsec
# or use Docker: docker run -it --rm -v $(pwd):/src aquasec/tfsec /src
```

Run
```bash
tfsec .
```

Example detections
- Public S3 buckets
- Unencrypted EBS volumes
- RDS without backup retention
- Security groups allowing 0.0.0.0/0 unnecessarily

CI usage
- Run tfsec in PR pipeline; fail on medium/high severity findings.
- Use `--exclude` or `.tfsecignore` for acceptable exceptions, but record them.

Configuration (example)
```yaml
# tfsec configuration is mainly CLI flags; use tfsec.json to map severity rules if needed
```

---

10) Checkov — Advanced security & compliance scanner
What it is: comprehensive policy engine with many pre-built checks (CIS, PCI-DSS, NIST, etc.)

Install
```bash
pip install checkov
# or use Docker: docker run -v $(pwd):/tf bridgecrew/checkov -d /tf
```

Run
```bash
checkov -d .
```

Example: run specific framework
```bash
checkov -d . --framework AWS
```

Notes
- Checkov and tfsec have some overlap; using one or both depends on coverage and team preference.
- Checkov supports scanning for IaC + generating reports and suppressing by ID with justification.

---

CI/CD Examples

1) GitHub Actions — Lint, Security, Infracost and OPA check (example snippet)
```yaml
name: terraform-pr
on:
  pull_request:
    paths:
      - 'terraform/**'

jobs:
  lint-and-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: 1.5.0

      - name: Install tools
        run: |
          curl -LO https://github.com/terraform-linters/tflint/releases/latest/download/tflint_linux_amd64.zip && unzip tflint_linux_amd64.zip && sudo mv tflint /usr/local/bin/
          curl -LO https://github.com/snyk/driftctl/releases/latest/download/driftctl_linux_amd64 && chmod +x driftctl_linux_amd64 && sudo mv driftctl_linux_amd64 /usr/local/bin/driftctl
          pip install checkov tfsec

      - name: Terraform fmt
        run: terraform fmt -check

      - name: TFLint
        run: tflint --init && tflint

      - name: Terraform Init & Plan
        run: |
          terraform init -backend=false
          terraform plan -out=plan.tfplan
          terraform show -json plan.tfplan > plan.json
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

      - name: OPA policy check
        run: |
          curl -L -o opa https://openpolicyagent.org/downloads/latest/opa_linux_amd64
          chmod +x opa
          ./opa eval --input plan.json --data policies/ "data.terraform.deny" --format pretty
        # fail if any denies are returned (adjust command or wrap in script to exit non-zero)

      - name: tfsec
        run: tfsec .

      - name: Checkov
        run: checkov -d .

      - name: Infracost comment
        uses: infracost/infracost-action@v2
        with:
          path: .
          terraform_plan_json: plan.json
        env:
          INFRACOST_API_KEY: ${{ secrets.INFRACOST_API_KEY }}
```

2) Scheduled driftctl run (GitHub Actions nightly)
```yaml
name: drift-scan
on:
  schedule:
    - cron: '0 2 * * *'  # daily at 02:00 UTC

jobs:
  drift:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install driftctl
        run: |
          curl -LO https://github.com/snyk/driftctl/releases/latest/download/driftctl_linux_amd64
          chmod +x driftctl_linux_amd64
          sudo mv driftctl_linux_amd64 /usr/local/bin/driftctl
      - name: Run driftctl
        run: driftctl scan --from tfstate+s3://my-bucket/path/to/terraform.tfstate --output json://drift.json && cat drift.json
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_DRIFT_SCAN_ACCESS_KEY }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_DRIFT_SCAN_SECRET_KEY }}
```

---

Sample config & snippets

.tflint.hcl (example)
```hcl
plugin "aws" {
  enabled = true
  version = "0.38.0"
}

rule "aws_instance_invalid_type" {
  enabled = true
}

# You can write custom rules via tflint custom rule mechanisms or enforce tagging in pre-commit
```

.infracost.yml (example)
```yaml
version: 0.1
project:
  name: my-terraform
  terraform_version: 1.5.0
```

terraform-docs config (.terraform-docs.yml)
```yaml
formatter: markdown table
settings:
  inputs:
    show_default: true
  outputs:
    show_required: true
```

OPA folder structure
- policies/
  - deny_public_s3.rego
  - require_tagging.rego

tfsec usage with ignore (example .tfsecignore)
```
# ignore a false positive with reason
TFSEC:AWS012 - false positive - resource uses custom encryption in separate module
```

---

Importing existing AWS resources — workflow examples

A) Use terraformer to generate TF skeletons for a VPC
```bash
terraformer import aws --resources=vpc,subnet,route_table,security_group --regions=ap-south-1 --profile import-profile
# Review generated files, move resources into modules, clean up attributes that should be variables
```

B) Targeted import using terraform import
1. Create HCL resource definition in code that matches the resource type
2. Run import:
```bash
terraform init
terraform import aws_instance.my_instance i-0123456789abcdef0
```
3. Run `terraform plan` and fix the differences by adjusting the HCL if needed.

---

Tagging strategy & Terratag example policy

- Prefer module-level `default_tags` or tag merging pattern. Example in module:
```hcl
variable "tags" { type = map(string) default = {} }

locals {
  common_tags = merge({Owner = "devops", Environment = "prod"}, var.tags)
}

resource "aws_instance" "example" {
  ami = var.ami
  tags = local.common_tags
}
```

- If you must backfill tags across many repos, use Terratag, then update modules.

Terratag backfill example:
```bash
terratag -dir ./ -tags owner=platform,environment=prod,costcenter=1234
```

---

Security & Compliance

- Use tfsec and Checkov in CI. Consider one as the primary scanner and the other as a secondary if you need broader coverage.
- Maintain a suppression policy: suppressions should include a justification and expiration (tracked in a central place).
- Use IAM least privilege for both CI credentials and any tools that perform write actions.

---

Testing and promotion

- Fast tests in PR (unit-like: tflint, tfsec).
- Integration tests in merge-stage or nightly using Terratest.
- Blue/green or canary patterns for infrastructure changes when possible.
- Use feature branches and promotion pipelines to move from dev -> staging -> prod.

---

Troubleshooting & remediation examples

1) Terraform plan fails after manual deletion in AWS
- Run driftctl to confirm missing resource.
- If resource was deleted accidentally and should be created by Terraform: `terraform apply` will recreate (if code still defines it).
- If resource was deleted intentionally: remove the resource from state (`terraform state rm`), or update code to remove resource and apply.

2) Resource created manually in AWS and should be managed by Terraform
- Add HCL for the resource (prefer using modules).
- Run `terraform import <address> <resource-id>` to import into state.
- Run `terraform plan` and reconcile differences.

3) Handling large imported state from terraformer
- Refactor generated code into modules.
- Parameterize values.
- Review provider and resource meta-attributes such as tags, descriptions, and generated names.

---

Example: Full PR reviewer checklist (automated and manual)
Automated (CI gates):
- terraform fmt check
- tflint no errors
- tfsec / Checkov no critical/high alerts
- Infracost posted in PR
- OPA policies pass on plan.json

Manual:
- Approve cost increase if > threshold
- Confirm tagging/navigation/readme updated
- Confirm test coverage for new network/exposed resources

---

Appendix: Useful command snippets

- Generate plan JSON:
```bash
terraform plan -out plan.tfplan
terraform show -json plan.tfplan > plan.json
```

- Run tfsec safely in Docker:
```bash
docker run --rm -v $(pwd):/src aquasec/tfsec /src
```

- Import a single resource:
```bash
terraform import aws_s3_bucket.example my-bucket-name
```

- Create Infracost diff for a PR locally:
```bash
infracost breakdown --path . --terraform-plan plan.json
```

- Run OPA against plan.json and fail if denies exist (bash):
```bash
if ./opa eval --input plan.json --data policies/ "data.terraform.deny" --format pretty | grep -q "deny"; then
  echo "OPA denies found"; exit 1
fi
```

---

Final notes & recommended next steps
- Start by adding tflint, tfsec/Checkov and terraform-docs to the developer pre-commit/CI for immediate feedback.
- Add Infracost to PR checks to surface cost changes early.
- Implement scheduled driftctl scans and create a process for triaging findings.
- Use Terraformer only as a first pass when migrating legacy resources — expect manual refactoring.
- Iterate: run static checks early, enforce policies via OPA, and automate reporting for visibility.

This document aims to be a single reference for tooling decisions and CI integration patterns. Copy the CI snippets and config templates into your repository, update paths/secrets, and adapt policy rule examples to your organization’s requirements.

Change-log
- 2026-01-05: Initial detailed guide created covering 10 key tools, CI examples and recommended workflows.
