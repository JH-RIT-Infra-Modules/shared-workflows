# Shared Workflows for Terraform Deployments

This repository contains reusable GitHub Actions workflows for deploying Terraform projects to Azure.

## 📋 Overview

These shared workflows provide a standardized CI/CD pipeline for Terraform infrastructure deployments with:

- ✅ **Multi-environment support** (dev, test, stage, prod)
- ✅ **Parallel execution** across environments using matrix strategy
- ✅ **Validation, planning, and deployment** stages
- ✅ **Controlled destruction** with confirmation safeguards
- ✅ **Environment-specific secrets** and configurations

## 📁 Workflows Included

| Workflow | Description |
|----------|-------------|
| `validate.yaml` | Validates Terraform code (linting, formatting, syntax) across all environments |
| `plan-deploy.yaml` | Generates plans and deploys infrastructure to selected environments |
| `destroy.yaml` | Destroys infrastructure in selected environments (requires confirmation) |

## 📖 Documentation

For detailed documentation on workflow configuration, prerequisites, and usage, see:

**[.github/workflows/README.md](.github/workflows/README.md)**

## 🚀 Quick Start

1. Reference these workflows from your Terraform project repository
2. Configure GitHub Environments with Azure credentials (dev, test, stage, prod)
3. Add environment-specific `tfvars` files (optional)
4. Push changes to trigger the CI/CD pipeline

## 📄 License

Internal use only - Johns Hopkins University
