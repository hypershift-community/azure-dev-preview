# HyperShift Azure Dev-Preview Automation

Taskfile-based automation for creating HyperShift hosted clusters on self-managed Azure.

## Overview

This directory contains Task automation that demonstrates the exact CLI commands needed to deploy HyperShift on Azure. All commands (`az`, `ccoctl`, `hypershift`, etc.) are visible in the Taskfile - no hidden shell scripts.

## Prerequisites

### Required Tools

Install these CLIs before starting:

- **Task** - Task automation tool ([install guide](https://taskfile.dev/installation/))
- **Azure CLI** (`az`) - Azure command-line interface
- **ccoctl** - Cloud Credential Operator CLI
- **hypershift** - HyperShift CLI (installation instructions below)
- **oc** - OpenShift CLI

### Installing HyperShift CLI

Download from [GitHub Releases](https://github.com/hypershift-community/azure-dev-preview/releases):

```bash
# Linux amd64
curl -LO https://github.com/hypershift-community/azure-dev-preview/releases/latest/download/hypershift-linux-amd64.tar.gz
tar -xzf hypershift-linux-amd64.tar.gz
sudo install -m 0755 hypershift /usr/local/bin/hypershift

# macOS (Intel)
curl -LO https://github.com/hypershift-community/azure-dev-preview/releases/latest/download/hypershift-darwin-amd64.tar.gz
tar -xzf hypershift-darwin-amd64.tar.gz
sudo install -m 0755 hypershift /usr/local/bin/hypershift

# macOS (Apple Silicon)
curl -LO https://github.com/hypershift-community/azure-dev-preview/releases/latest/download/hypershift-darwin-arm64.tar.gz
tar -xzf hypershift-darwin-arm64.tar.gz
sudo install -m 0755 hypershift /usr/local/bin/hypershift

# Verify installation
hypershift version
```

### Required Files

You'll need these files before creating a cluster:

- `azure-credentials.json` - Azure service principal credentials
- `pull-secret.json` - OpenShift pull secret

## Quick Start

### 1. Configure Variables

All configuration is done via Taskfile variables. Set them when running tasks:

```bash
# Option A: Export in your shell
export SUBSCRIPTION_ID="your-subscription-id"
export TENANT_ID="your-tenant-id"
export PREFIX="myprefix"
export OIDC_STORAGE_ACCOUNT_NAME="myprefixoidc$(date +%s)"
export RELEASE_IMAGE="quay.io/openshift-release-dev/ocp-release:4.21.0-x86_64"
export PARENT_DNS_ZONE="example.com"

# Option B: Set inline when calling task
SUBSCRIPTION_ID=xxx TENANT_ID=xxx PREFIX=myprefix task setup
```

**Optional**: For convenience when running manual CLI commands, copy `envrc.example` to `.envrc` and customize it. The Taskfile doesn't use `.envrc` - it's just for your reference.

### 2. Validate Prerequisites

```bash
task prereq:validate
```

This checks:
- All required variables are set
- Required CLIs are installed
- Credential files exist

### 3. Run Complete Setup

```bash
task setup
```

This runs:
1. Generate service account keys
2. Create Azure OIDC issuer
3. Create managed identities
4. Create federated credentials
5. Create networking infrastructure
6. Create HyperShift cluster

### 4. Destroy Cluster

```bash
task cluster:destroy
```

## Available Tasks

Run `task --list` to see all available tasks:

```bash
task --list
```

### Main Workflows

- `task setup` - Complete cluster setup (everything)
- `task prereq:all` - Prerequisites only
- `task azure:all` - Azure setup only
- `task cluster:create` - Create cluster only
- `task cluster:destroy` - Destroy cluster

### Individual Tasks

- `task prereq:validate` - Validate environment
- `task prereq:keys` - Generate service account keys
- `task azure:oidc` - Create OIDC issuer
- `task azure:identities` - Create managed identities
- `task azure:federated-creds` - Create federated credentials
- `task azure:infra` - Create networking infrastructure

## Configuration Variables

See the `vars:` section in `Taskfile.yml` for all available variables and their defaults.

### Required Variables

- `SUBSCRIPTION_ID` - Azure subscription ID
- `TENANT_ID` - Azure tenant ID
- `PREFIX` - Unique naming prefix
- `OIDC_STORAGE_ACCOUNT_NAME` - Globally unique storage account name
- `RELEASE_IMAGE` - OpenShift release image
- `PARENT_DNS_ZONE` - Base DNS domain

### Optional Variables

- `LOCATION` - Azure region (default: eastus)
- `NODE_POOL_REPLICAS` - Worker node count (default: 2)
- `CPO_IMAGE` - Control plane operator override (optional)

## Files Created

The automation creates these files:

- `serviceaccount-signer.private` - Private key for token signing
- `serviceaccount-signer.public` - Public key for token signing
- `workload-identities.json` - Managed identity client IDs
- `.envrc.az_net_ids` - Network resource IDs (temporary)

**Important**: Never commit these files to git! They are in `.gitignore`.

## Documentation

For detailed documentation, see:

- [Understanding HyperShift on Azure](../../docs/content/how-to/azure/self-managed/01-understanding.md)
- [Planning Your Deployment](../../docs/content/how-to/azure/self-managed/02-planning.md)
- [Complete Guide](../../docs/content/how-to/azure/self-managed/)

## Design Philosophy

This automation follows the "executable documentation" approach:

- **Visible CLI commands** - All `az`, `ccoctl`, `hypershift` commands are in the Taskfile
- **Self-documenting** - Variables and commands together in one place
- **No hidden logic** - No shell scripts to maintain
- **Learning-friendly** - Veterans can see exactly what commands are run

The Taskfile serves as both automation and teaching tool.
