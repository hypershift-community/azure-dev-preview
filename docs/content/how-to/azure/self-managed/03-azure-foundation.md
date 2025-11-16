# Azure Foundation Setup

This guide covers the one-time Azure infrastructure setup required before creating hosted clusters. These resources are **shared across all hosted clusters** in your environment.

!!! important "One-Time Setup"

    This setup is performed **ONCE** and shared across all hosted clusters. You do NOT repeat this for each cluster.

## Overview

The Azure foundation consists of:

1. **Service Account Signing Keys**: RSA key pair for signing service account tokens
2. **OIDC Issuer**: Azure Blob Storage container providing OpenID Connect endpoint
3. **Persistent Resource Group**: Container for shared resources

## Prerequisites

Before beginning, ensure you have:

- Azure CLI (`az`) authenticated to your subscription
- Cloud Credential Operator tool (`ccoctl`) installed
- [go-task/task](https://taskfile.dev) installed
- Taskfile configuration completed in `hack/dev-preview/Taskfile.yml`

Required Taskfile variables:
```yaml
SUBSCRIPTION_ID: 'your-subscription-id'
TENANT_ID: 'your-tenant-id'
LOCATION: 'eastus'
PERSISTENT_RG_NAME: 'hypershift-shared'
OIDC_STORAGE_ACCOUNT_NAME: 'unique-storage-name'
```

## Step 1: Validate Prerequisites

Validate that all required tools and variables are set:

```bash
task prereq:validate
```

This checks for:
- Required environment variables
- Azure CLI installation and authentication
- ccoctl installation
- Required credential files

## Step 2: Generate Service Account Signing Keys

Generate the RSA key pair used to sign service account tokens for workload identity:

```bash
task prereq:keys
```

**What this does:**

- Creates `serviceaccount-signer.private` (4096-bit RSA private key)
- Creates `serviceaccount-signer.public` (corresponding public key)
- These keys are used to sign JWTs for workload identity federation

**Under the hood** (see `tasks/prereq.yml`):

```bash
ccoctl azure create-key-pair --output-dir .
```

**Output:**

```
Generating service account token issuer key pair...
✓ Generated RSA key pair
  Private key: serviceaccount-signer.private
  Public key: serviceaccount-signer.public
```

!!! warning "Key Security"

    The private key must be kept secure. It's used to sign service account tokens that authenticate with Azure.

    - Store securely (e.g., secrets management system)
    - Do not commit to version control
    - Back up safely

## Step 3: Create OIDC Issuer

Create the Azure Blob Storage account that serves as the OIDC issuer:

```bash
task azure:oidc
```

**What this does:**

- Creates a storage account in the persistent resource group
- Uploads the public signing key to the storage account
- Configures the OIDC discovery endpoint
- Sets up the required blob container and files

**Under the hood** (see `tasks/azure.yml`):

```bash
ccoctl azure create-oidc-issuer \
  --oidc-resource-group-name ${PERSISTENT_RG_NAME} \
  --tenant-id ${TENANT_ID} \
  --region ${LOCATION} \
  --name ${OIDC_STORAGE_ACCOUNT_NAME} \
  --subscription-id ${SUBSCRIPTION_ID} \
  --public-key-file serviceaccount-signer.public
```

**Output:**

```
Creating OIDC issuer (one-time setup)...
✓ OIDC issuer created at https://yourstorageaccount.blob.core.windows.net/yourstorageaccount
```

### What Gets Created

The OIDC issuer consists of:

1. **Azure Storage Account**: Named `${OIDC_STORAGE_ACCOUNT_NAME}`
2. **Blob Container**: Same name as storage account
3. **OIDC Discovery Files**:
   - `.well-known/openid-configuration`: OIDC discovery document
   - `openid/v1/jwks`: JSON Web Key Set (public keys)

### Verification

Verify the OIDC issuer is accessible:

```bash
# Check storage account exists
az storage account show \
  --name ${OIDC_STORAGE_ACCOUNT_NAME} \
  --resource-group ${PERSISTENT_RG_NAME}

# Test OIDC endpoint (should return JSON)
curl https://${OIDC_STORAGE_ACCOUNT_NAME}.blob.core.windows.net/${OIDC_STORAGE_ACCOUNT_NAME}/.well-known/openid-configuration
```

## Understanding OIDC Issuer Scope

The OIDC issuer is **shared infrastructure**:

### What It's Shared Across
- All hosted clusters using this management cluster
- All clusters in the same environment/region
- Multiple teams using the same Azure subscription

### Why It's Shared
- **Cost efficiency**: One storage account vs many
- **Simplified management**: Single OIDC endpoint to manage
- **Consistency**: Same authentication mechanism across clusters

### When to Create Multiple Issuers
- Different Azure subscriptions
- Compliance requirements for isolation
- Different security zones or environments

!!! tip "Production Recommendation"

    Use one OIDC issuer per:

    - **Environment** (dev, staging, prod) - For change control
    - **Security zone** - For compliance requirements
    - **Azure subscription** - For billing separation

## Resource Lifecycle

Understanding what happens to these resources:

| Resource | Lifecycle | Deletion |
|----------|-----------|----------|
| Service Account Keys | One-time, manual | Manual only (keep for disaster recovery) |
| OIDC Issuer | One-time, persistent | Manual via `task azure:delete-oidc` |
| Persistent Resource Group | Long-lived | Manual only |

!!! danger "Deleting the OIDC Issuer"

    Deleting the OIDC issuer will **break authentication for ALL clusters** using it. Only delete when:

    - All clusters using it have been decommissioned
    - You're performing a complete environment teardown

    Delete with: `task azure:delete-oidc`

## Troubleshooting

### Storage Account Name Already Taken

Storage account names are globally unique across all of Azure.

**Error:**
```
The storage account named 'myoidc' is already taken.
```

**Solution:**
Add a random suffix to your storage account name:
```yaml
OIDC_STORAGE_ACCOUNT_NAME: 'myoidc1234567890'
```

### Authentication Errors

**Error:**
```
ERROR: Please run 'az login' to setup account.
```

**Solution:**
```bash
az login
az account set --subscription ${SUBSCRIPTION_ID}
```

### Insufficient Permissions

**Error:**
```
The client does not have authorization to perform action 'Microsoft.Storage/storageAccounts/write'
```

**Solution:**
Ensure your Azure account has:
- `Contributor` role on the subscription or resource group
- `User Access Administrator` for managed identity operations

## Next Steps

With the Azure foundation in place, proceed to:

- [Management Cluster Setup](04-management-cluster.md) - Install HyperShift operator
- Or jump to [Create Hosted Cluster](05-hosted-cluster.md) if your management cluster is ready
