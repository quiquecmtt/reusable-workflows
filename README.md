# Reusable Workflows

A collection of reusable GitHub Actions workflows for Terraform and OpenTofu projects.

## Available Workflows

| Workflow | Description |
|----------|-------------|
| `renovate.yml` | Automated dependency updates with Renovate |
| `terraform-ci.yml` | Terraform linting, validation, security scanning, and docs |
| `opentofu-ci.yml` | OpenTofu linting, validation, security scanning, and docs |

## Usage

### Renovate

Caller workflow (`.github/workflows/renovate.yml`):

```yaml
name: Renovate

on:
  schedule:
    - cron: "5 12 * * 3"  # Wednesdays at 12:05 PM UTC
  workflow_dispatch:
    inputs:
      debug:
        description: "Run Renovate with DEBUG log level"
        required: false
        default: false
        type: boolean

jobs:
  renovate:
    uses: quiquecmtt/reusable-workflows/.github/workflows/renovate.yml@main
    with:
      debug: ${{ inputs.debug || false }}
    secrets:
      RENOVATE_TOKEN: ${{ secrets.RENOVATE_TOKEN }}
```

#### Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `debug` | Run Renovate with DEBUG log level | No | `false` |
| `config_file` | Path to Renovate configuration file | No | `.github/renovate.json` |

#### Secrets

| Secret | Description | Required |
|--------|-------------|----------|
| `RENOVATE_TOKEN` | GitHub token for Renovate | Yes |

---

### Terraform CI

Caller workflow (`.github/workflows/ci.yml`):

```yaml
name: Terraform CI

on:
  push:
    branches: [main]
  pull_request:
  workflow_dispatch:

jobs:
  ci:
    uses: quiquecmtt/reusable-workflows/.github/workflows/terraform-ci.yml@main
    with:
      enable_security_scan: true
      enable_docs: true
      allowed_pr_author: "your-username"  # Optional: restrict PR runs
    permissions:
      contents: write  # Required if enable_docs is true
```

#### Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `runs_on` | Runner to use | No | `ubuntu-latest` |
| `working_directory` | Working directory for Terraform commands | No | `.` |
| `validate_directory` | Directory for init/validate (for modules with provider aliases) | No | Same as `working_directory` |
| `terraform_version` | Terraform version (empty for latest) | No | `""` |
| `enable_security_scan` | Enable Checkov and TFSec scanning | No | `false` |
| `enable_docs` | Enable terraform-docs generation | No | `false` |
| `docs_output_file` | Output file for terraform-docs | No | `README.md` |
| `allowed_pr_author` | GitHub username allowed to run on PRs (empty for all) | No | `""` |

#### Modules with Provider Aliases

For modules that use `configuration_aliases` (e.g., multi-account AWS modules), create a `tests/` directory with a fixture that provides mock providers:

```hcl
# tests/main.tf
provider "aws" {
  alias  = "delegating"
  region = "us-east-1"
  skip_credentials_validation = true
  skip_metadata_api_check     = true
  skip_requesting_account_id  = true
}

module "test" {
  source = "../"
  # ... module inputs
  providers = {
    aws.delegating = aws.delegating
  }
}
```

Then set `validate_directory: "tests"` in your workflow.

#### Jobs

- **lint**: Format check, init, validate, TFLint
- **security**: Checkov and TFSec scans (when enabled)
- **docs**: terraform-docs generation on push to main (when enabled)

---

### OpenTofu CI

Caller workflow (`.github/workflows/ci.yml`):

```yaml
name: OpenTofu CI

on:
  push:
    branches: [main]
  pull_request:
  workflow_dispatch:

jobs:
  ci:
    uses: quiquecmtt/reusable-workflows/.github/workflows/opentofu-ci.yml@main
    with:
      enable_security_scan: true
      enable_docs: true
    permissions:
      contents: write  # Required if enable_docs is true
```

#### Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `runs_on` | Runner to use | No | `ubuntu-latest` |
| `working_directory` | Working directory for OpenTofu commands | No | `.` |
| `validate_directory` | Directory for init/validate (for modules with provider aliases) | No | Same as `working_directory` |
| `opentofu_version` | OpenTofu version (empty for latest) | No | `""` |
| `enable_security_scan` | Enable Checkov and TFSec scanning | No | `false` |
| `enable_docs` | Enable terraform-docs generation | No | `false` |
| `docs_output_file` | Output file for terraform-docs | No | `README.md` |
| `allowed_pr_author` | GitHub username allowed to run on PRs (empty for all) | No | `""` |

## Migration from Existing Workflows

To migrate your existing Terraform modules to use these reusable workflows:

1. Replace your `.github/workflows/renovate.yml` content with the Renovate caller example above
2. Replace your `.github/workflows/ci.yml` content with the Terraform/OpenTofu CI caller example above
3. Update repository references if you fork this to a different organization
4. Ensure you have the required secrets configured (`RENOVATE_TOKEN`)
