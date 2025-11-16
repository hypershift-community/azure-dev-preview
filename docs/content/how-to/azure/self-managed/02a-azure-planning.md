# Azure Planning

This guide helps you understand Azure-specific requirements and make key planning decisions for your HyperShift deployment.

## Target Audience

This guide is designed for **OpenShift administrators and SREs** who are new to Azure. We assume you have:

- Strong knowledge of OpenShift/Kubernetes operations
- Limited or no experience with Azure infrastructure
- Understanding of networking, storage, and identity management concepts in Kubernetes

## Prerequisites

### Azure Requirements

If you're new to Azure but experienced with OpenShift/Kubernetes, this section explains Azure-specific concepts you'll need.

#### Azure Subscription & Your Permissions

**What is an Azure Subscription?**

An Azure subscription is the billing and access boundary for Azure resources. Think of it as the top-level container where all your Azure resources live and are billed.

**Your Azure account needs these permissions:**

| Permission | Why You Need It | OpenShift/K8s Equivalent |
|------------|----------------|--------------------------|
| `Contributor` | Create and manage Azure resources (VMs, networks, storage) | cluster-admin for infrastructure |
| `User Access Administrator` | Assign roles to managed identities | Permission to create RoleBindings |
| `Application.ReadWrite.OwnedBy`<br/>(Microsoft Graph API) | Create service principals for cluster operations | Permission to create ServiceAccounts |

**How to verify your permissions:**

```bash
# Check your subscription roles
az role assignment list --assignee $(az ad signed-in-user show --query id -o tsv) \
  --scope /subscriptions/$(az account show --query id -o tsv)

# Look for "Contributor" and "User Access Administrator" in the output
```

**Why both Contributor and User Access Administrator?**

- `Contributor` lets you create resources but NOT assign permissions to them
- `User Access Administrator` lets you assign permissions (roles) to managed identities
- HyperShift needs both: create resources AND grant them permissions to work together

#### Azure Resource Groups

**What is a Resource Group?**

Similar to a Kubernetes namespace, but for Azure resources. All Azure resources must belong to exactly one resource group. Resource groups are a logical container for managing lifecycle and access control.

**You'll need TWO types:**

1. **Persistent Resource Group** (e.g., `hypershift-shared`)
    - **Lifecycle**: Long-lived, survives across multiple cluster deployments
    - **Contains**: OIDC issuer (storage account), managed identities, federated credentials, DNS zones (optional)
    - **Important**: Never delete this - it contains shared infrastructure
2. **Per-Cluster Resource Groups** (created automatically by tasks)
    - **Lifecycle**: Created and deleted with each hosted cluster
    - **Contains**: VNet, NSG, subnets, worker node VMs
    - **Safe to delete** when removing a cluster (using `task cluster:destroy`)

!!! tip "Resource Group Analogy"

    Think of the persistent resource group like a shared namespace for operators and the per-cluster groups like workload-specific namespaces that come and go.

#### Optional: Azure DNS Zone

**What is it?**

Azure DNS is Microsoft's DNS hosting service. A DNS zone hosts the DNS records for a specific domain (like `example.com`).

**When do you need it?**

- Only if using External DNS for automatic DNS management (recommended for production)
- Not needed for development/testing - you can use Azure LoadBalancer DNS instead (e.g., `abc123.eastus.cloudapp.azure.com`)

**What is "DNS zone delegation"?**

When you create a hosted cluster with domain `my-cluster.azure.example.com`:

1. The parent zone (`azure.example.com`) must already exist in Azure DNS
2. HyperShift creates a child zone (`my-cluster.azure.example.com`) automatically
3. "Delegation" means the parent zone's NS records point to the child zone's nameservers
4. This allows the child zone to manage its own DNS records independently

!!! example "DNS Delegation Example"

    If your parent zone is `azure.example.com` in Azure DNS:

    - HyperShift creates child zone: `my-cluster.azure.example.com`
    - Parent zone gets NS record: `my-cluster.azure.example.com` → `[child-ns1, child-ns2, ...]`
    - Cluster API becomes: `api.my-cluster.azure.example.com`
    - Apps become: `*.apps.my-cluster.azure.example.com`

#### Required Tools

Install these tools on your workstation:

| Tool | Purpose | Installation |
|------|---------|--------------|
| Azure CLI (`az`) | Interact with Azure APIs | [Install guide](https://learn.microsoft.com/cli/azure/install-azure-cli) |
| `jq` | Parse JSON output | `brew install jq` or `apt install jq` |
| `ccoctl` | Cloud Credential Operator utility for Azure identity setup | Bundled with OpenShift installer or [download](https://mirror.openshift.com/pub/openshift-v4/clients/ocp/) |
| `task` | Task runner for automation scripts | [Install guide](https://taskfile.dev/installation/) |

### Management Cluster Requirements

#### Cluster Prerequisites
- An existing OpenShift cluster running in Azure
- Cluster must have:
    - Sufficient capacity for hosting control plane pods
    - Network connectivity to Azure APIs
    - Ability to create LoadBalancer services (if not using External DNS)
- Administrative access (`cluster-admin` permissions) to the management cluster

#### Tools
- OpenShift CLI (`oc`)
- HyperShift CLI binary ([download](https://github.com/hypershift-community/azure-dev-preview/releases))

### Required Configuration Files

Understanding which files you need to prepare versus which are generated automatically:

#### Files You Must Create

These files must be prepared before starting the deployment:

| File | Purpose | When Needed | How to Get |
|------|---------|-------------|------------|
| **azure-credentials.json** | Service principal for cluster operations (create/destroy VMs, networks) | Before creating first cluster | See detailed instructions below |
| **pull-secret.json** | Red Hat pull secret for downloading OpenShift images | During operator installation | Download from [cloud.redhat.com](https://console.redhat.com/openshift/downloads) |

#### Files Generated During Deployment

These files are created automatically by the Taskfile automation and should not be manually edited:

| File | Purpose | Created By | Used By |
|------|---------|------------|---------|
| **workload-identities.json** | Managed identity client IDs for cluster components (storage, networking, etc.) | `task azure:identities` | `task cluster:create` |
| **serviceaccount-signer.private**<br/>**serviceaccount-signer.public** | RSA key pair for signing service account tokens | `task prereq:keys` | `task cluster:create` |
| **.azure-net-ids** | Network resource IDs (VNet, NSG, subnet) exported as environment variables | `task azure:infra` | `task cluster:create` |

!!! tip "File Lifecycle"

    - **You create**: `azure-credentials.json`, `pull-secret.json`
    - **Automation creates**: All other files during setup
    - **Don't edit**: Generated files - they're overwritten on each run

!!!info "Azure Credentials File Format"

    The `azure-credentials.json` file should contain service principal credentials:

    ```json
    {
      "subscriptionId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
      "tenantId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
      "clientId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
      "clientSecret": "your-service-principal-secret"
    }
    ```

    **How to create:**

    ```bash
    # Create service principal with required permissions
    SP_DETAILS=$(az ad sp create-for-rbac \
        --name "hypershift-cluster-ops" \
        --role Contributor \
        --scopes "/subscriptions/$(az account show --query id -o tsv)")

    # Extract values and create file
    cat <<EOF > azure-credentials.json
    {
      "subscriptionId": "$(az account show --query id -o tsv)",
      "tenantId": "$(az account show --query tenantId -o tsv)",
      "clientId": "$(echo "$SP_DETAILS" | jq -r '.appId')",
      "clientSecret": "$(echo "$SP_DETAILS" | jq -r '.password')"
    }
    EOF
    ```

## Key Planning Decisions

### DNS Management Strategy

Choose how to handle DNS for your hosted clusters:

#### Option 1: With External DNS (Recommended for Production)

**Best For**: Production environments, multiple clusters, custom domains

**Characteristics**:
- Automatic DNS record management via External DNS operator
- Custom domain names (e.g., `api.my-cluster.example.com`)
- Requires: DNS zones, service principal with DNS permissions, External DNS operator
- Higher initial setup complexity, but simpler ongoing operations

**Example DNS**:
- API Server: `api.my-cluster.azure.example.com`
- Apps: `*.apps.my-cluster.azure.example.com`

**Decision Criteria**:
- Choose this if you need custom, branded domain names
- Choose this if you plan to manage multiple clusters
- Choose this if you want fully automated DNS provisioning

#### Option 2: Without External DNS (Simpler for Dev/Test)

**Best For**: Development, testing, proof-of-concept environments

**Characteristics**:
- Manual DNS management or Azure-provided LoadBalancer DNS
- API server uses Azure LoadBalancer DNS (e.g., `abc123.eastus.cloudapp.azure.com`)
- No DNS zones or service principals needed
- Lower initial setup complexity, but requires manual DNS work for production use

**Example DNS**:
- API Server: `my-cluster-api.eastus.cloudapp.azure.com`
- Apps: `my-cluster-apps.eastus.cloudapp.azure.com`

**Decision Criteria**:
- Choose this for quick testing or POC environments
- Choose this if you don't control your DNS infrastructure
- Choose this if you only need a single cluster temporarily

!!! tip "Recommendation"

    **Start with Option 2 (Without External DNS)** for your first cluster to learn the basics. Once comfortable, you can deploy production clusters with Option 1 (External DNS).

### Resource Group Strategy

Plan your resource group structure to separate long-lived shared resources from cluster-specific resources:

#### Persistent Resource Group
- **Name example**: `hypershift-shared`, `openshift-common`
- **Lifecycle**: Long-lived, not deleted when clusters are removed
- **Contains**:
    - OIDC issuer (storage account)
    - Managed identities (per cluster, but persistent)
    - Federated identity credentials (per cluster, but persistent)
    - (Optional) DNS zones

#### Per-Cluster Resource Groups
- **Name pattern**: `<prefix>-vnet-rg`, `<prefix>-nsg-rg`, `<prefix>-managed-rg`
- **Lifecycle**: Created and deleted with each cluster
- **Contains**:
    - Virtual Network (VNet)
    - Network Security Group (NSG)
    - Subnets
    - Virtual Machines (worker nodes)

!!! important "Understanding Resource Persistence"

    Managed identities and federated credentials are **per-cluster but persistent**. They:

    - Are created once per cluster
    - Remain after cluster deletion
    - Allow cluster recreation without identity recreation
    - Preserve Azure IAM role assignments
    - Support disaster recovery scenarios

    Network infrastructure is **temporary** and deleted with the cluster.

### Network Planning

#### VNet and Subnet Design
- Default: `/16` VNet with `/24` subnet
- Plan for sufficient IP addresses for worker nodes
- Consider network security group (NSG) rules for your environment
- Ensure network connectivity to Azure APIs and management cluster

#### Network Security
- NSG rules are created automatically
- Review and customize based on your security requirements
- Consider private endpoint requirements for production

## Next Steps

Now that you've planned your Azure infrastructure:

- [Workflow & Repository Setup](02b-workflow-planning.md) - Understand how this repository works and how to configure it
- [Azure Foundation Setup](03-azure-foundation.md) - Set up Azure infrastructure using Taskfile automation
