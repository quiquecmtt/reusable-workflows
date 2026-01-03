# Reusable Workflows

A collection of reusable GitHub Actions workflows for Terraform and OpenTofu infrastructure projects.

## Features

- **Terraform & OpenTofu CI** - Format checking, validation, and linting with TFLint
- **Security Scanning** - Optional Checkov and TFSec security analysis
- **Auto Documentation** - Automatic README generation with terraform-docs
- **Dependency Updates** - Automated updates with Renovate
- **Provider Alias Support** - Test fixtures for modules with `configuration_aliases`

## Available Workflows

| Workflow | Description |
|----------|-------------|
| [`renovate.yml`](.github/workflows/renovate.yml) | Automated dependency updates with Renovate |
| [`terraform-ci.yml`](.github/workflows/terraform-ci.yml) | Terraform linting, validation, security scanning, and docs |
| [`opentofu-ci.yml`](.github/workflows/opentofu-ci.yml) | OpenTofu linting, validation, security scanning, and docs |

## Quick Start

### 1. Terraform CI (Basic)

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
  workflow_dispatch:

jobs:
  ci:
    uses: quiquecmtt/reusable-workflows/.github/workflows/terraform-ci.yml@main
```

### 2. Terraform CI (Full Features)

```yaml
# .github/workflows/ci.yml
name: CI

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
    permissions:
      contents: write
```

### 3. Renovate

```yaml
# .github/workflows/renovate.yml
name: Renovate

on:
  schedule:
    - cron: "5 12 * * 3"
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

## Workflow Reference

### Terraform CI

Runs format check, init, validate, and TFLint. Optionally runs security scans and generates documentation.

#### Inputs

| Input | Description | Default |
|-------|-------------|---------|
| `runs_on` | Runner to use | `ubuntu-latest` |
| `working_directory` | Working directory for Terraform commands | `.` |
| `validate_directory` | Directory for init/validate (for modules with provider aliases) | Same as `working_directory` |
| `terraform_version` | Terraform version (empty for latest) | `""` |
| `enable_security_scan` | Enable Checkov and TFSec scanning | `false` |
| `enable_docs` | Enable terraform-docs generation | `false` |
| `docs_output_file` | Output file for terraform-docs | `README.md` |
| `allowed_pr_author` | GitHub username allowed to run on PRs (empty for all) | `""` |

#### Jobs

| Job | Description | Condition |
|-----|-------------|-----------|
| `lint` | Format check, init, validate, TFLint | Always |
| `security` | Checkov and TFSec scans | `enable_security_scan: true` |
| `docs` | terraform-docs generation | `enable_docs: true` (push to main only) |

---

### OpenTofu CI

Same as Terraform CI but uses OpenTofu instead.

#### Inputs

| Input | Description | Default |
|-------|-------------|---------|
| `runs_on` | Runner to use | `ubuntu-latest` |
| `working_directory` | Working directory for OpenTofu commands | `.` |
| `validate_directory` | Directory for init/validate (for modules with provider aliases) | Same as `working_directory` |
| `opentofu_version` | OpenTofu version (empty for latest) | `""` |
| `enable_security_scan` | Enable Checkov and TFSec scanning | `false` |
| `enable_docs` | Enable terraform-docs generation | `false` |
| `docs_output_file` | Output file for terraform-docs | `README.md` |
| `allowed_pr_author` | GitHub username allowed to run on PRs (empty for all) | `""` |

---

### Renovate

Runs Renovate for automated dependency updates.

#### Inputs

| Input | Description | Default |
|-------|-------------|---------|
| `debug` | Run Renovate with DEBUG log level | `false` |
| `config_file` | Path to Renovate configuration file | `.github/renovate.json` |

#### Secrets

| Secret | Description | Required |
|--------|-------------|----------|
| `RENOVATE_TOKEN` | GitHub token for Renovate | Yes |

## Modules with Provider Aliases

For modules that use `configuration_aliases` (e.g., multi-account AWS), validation fails without provider configurations. Create a `tests/` directory with mock providers:

```hcl
# tests/main.tf
terraform {
  required_version = ">= 1.0.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0.0"
    }
  }
}

provider "aws" {
  alias  = "delegating"
  region = "us-east-1"

  skip_credentials_validation = true
  skip_metadata_api_check     = true
  skip_requesting_account_id  = true
}

provider "aws" {
  alias  = "delegated"
  region = "us-east-1"

  skip_credentials_validation = true
  skip_metadata_api_check     = true
  skip_requesting_account_id  = true
}

module "test" {
  source = "../"

  # Module inputs
  delegating_zone = "example.com"
  delegated_zone  = "sub.example.com"

  providers = {
    aws.delegating = aws.delegating
    aws.delegated  = aws.delegated
  }
}
```

Then configure your workflow:

```yaml
jobs:
  ci:
    uses: quiquecmtt/reusable-workflows/.github/workflows/terraform-ci.yml@main
    with:
      validate_directory: "tests"
```

## Auto Documentation Setup

To use automatic documentation generation:

1. Add placeholder markers to your `README.md`:

```markdown
<!-- BEGIN_TF_DOCS -->
<!-- END_TF_DOCS -->
```

2. Enable docs in your workflow:

```yaml
jobs:
  ci:
    uses: quiquecmtt/reusable-workflows/.github/workflows/terraform-ci.yml@main
    with:
      enable_docs: true
    permissions:
      contents: write
```

Documentation will be auto-generated on push to main.

## Migration Guide

1. **Replace workflow files** - Copy the examples above to your `.github/workflows/` directory
2. **Add test fixtures** - If your module uses `configuration_aliases`, create a `tests/` directory
3. **Configure secrets** - Ensure `RENOVATE_TOKEN` is set in your repository secrets
4. **Add doc markers** - Add `<!-- BEGIN_TF_DOCS -->` markers to README.md if using auto-docs
