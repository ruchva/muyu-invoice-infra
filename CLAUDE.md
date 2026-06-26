# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Tool Versions

Tool versions are pinned in `mise.toml`. Install them with:

```sh
mise install
```

Required tools: `terraform` 1.15.6, `awscli` 2.35.9, `checkov` 3.2.517, `infracost` 0.10.44.

## Common Commands

```sh
# Core workflow
terraform init
terraform fmt
terraform validate
terraform plan
terraform apply
terraform destroy

# Generate a plan file (required for checkov and infracost)
terraform plan -out tfplan.binary
terraform show -json tfplan.binary > tfplan.json

# Security scan
checkov -f tfplan.json

# Cost estimate (requires INFRACOST_API_KEY env var)
infracost breakdown --path tfplan.json

# Replicate CI checks locally
terraform fmt -check -recursive && terraform init -backend=false -input=false && terraform validate
```

## Architecture

Single flat root module — no nested modules. All AWS resources live in `vpc.tf`, variables in `variables.tf`, and exported values in `outputs.tf`.

**Network layout (us-east-1):**
- VPC: `10.10.0.0/16`
- Public subnets: `10.10.10.0/24` (us-east-1a), `10.10.11.0/24` (us-east-1b)
- Private subnets: `10.10.20.0/24` (us-east-1a), `10.10.21.0/24` (us-east-1b)
- Internet Gateway → public route table → public subnets
- NAT Gateway (Elastic IP, lives in `public1`) → private route table → private subnets

**Key conventions:**
- All resource `Name` tags use the `name_prefix` variable (default: `"muyu"`).
- Default tags (Project, ManagedBy, Repository, Owner) are injected globally via the provider block in `providers.tf` — no need to set them per resource.
- There is no remote state backend. State files (`terraform.tfstate`, `tfplan.binary`, `tfplan.json`) are gitignored and must never be committed.
- CI (`terraform init -backend=false`) confirms the configuration is valid without needing AWS credentials or a real backend.
