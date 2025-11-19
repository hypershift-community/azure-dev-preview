# Reference

This page provides reference material, appendices, and quick-reference guides for HyperShift on Azure.

## Taskfile Command Reference

### Prerequisites and Foundation

| Command | Purpose | When to Run | Lifecycle |
|---------|---------|-------------|-----------|
| `task prereq:validate` | Validate tools and configuration | Before starting | N/A |
| `task prereq:keys` | Generate SA signing keys | Once per environment | Persistent |
| `task azure:oidc` | Create OIDC issuer | Once per environment | Persistent (shared) |
| `task azure:delete-oidc` | Delete OIDC issuer | Environment teardown | Destructive |

### Per-Cluster Resources

| Command | Purpose | When to Run | Lifecycle |
|---------|---------|-------------|-----------|
| `task azure:identities` | Create managed identities (7) | Once per cluster | Persistent |
| `task azure:federated-creds` | Create federated credentials (14) | Once per cluster | Persistent |
| `task azure:infra` | Provision VNet, NSG, RGs | Before cluster creation | Temporary |
| `task cluster:create` | Deploy HostedCluster | After all prerequisites | Temporary |
| `task cluster:destroy` | Delete HostedCluster | Cluster teardown | Destructive |
| `task azure:delete-infra` | Delete VNet, NSG, RGs | After cluster destroyed | Destructive |
| `task azure:delete-federated-creds` | Delete federated credentials | Permanent removal | Destructive |
| `task azure:delete-identities` | Delete managed identities | Permanent removal | Destructive |

### Information Commands

| Command | Purpose |
|---------|---------|
| `task azure:list-identities` | List managed identities for cluster |
| `task azure:list-federated-creds` | List federated credentials |
| `task --list` | Show all available tasks |

## HyperShift CLI Reference

### Cluster Creation

```bash
hypershift create cluster azure \
    --name <cluster-name> \
    --namespace <namespace> \
    --azure-creds <credentials-file> \
    --location <azure-region> \
    --node-pool-replicas <count> \
    --base-domain <domain> \
    --pull-secret <pull-secret-file> \
    --release-image <ocp-release-image> \
    --resource-group-name <managed-rg> \
    --vnet-id <vnet-id> \
    --subnet-id <subnet-id> \
    --network-security-group-id <nsg-id> \
    --sa-token-issuer-private-key-path <private-key> \
    --oidc-issuer-url <oidc-url> \
    --workload-identities-file <identities-json> \
    [--dns-zone-rg-name <dns-rg>] \
    [--external-dns-domain <dns-domain>] \
    [--control-plane-operator-image <image>] \
    [--marketplace-publisher <publisher>] \
    [--marketplace-offer <offer>] \
    [--marketplace-sku <sku>] \
    [--marketplace-version <version>] \
    [--image-generation Gen1|Gen2] \
    [--assign-service-principal-roles] \
    [--diagnostics-storage-account-type Managed] \
    [--generate-ssh]
```

**Key Flags**:

- `--name`: Cluster name (must match identity creation)
- `--namespace`: Kubernetes namespace for HostedCluster resource
- `--azure-creds`: Service principal credentials JSON file
- `--location`: Azure region (e.g., eastus, westus2)
- `--node-pool-replicas`: Initial worker node count
- `--base-domain`: Base domain for cluster (used for ingress)
- `--pull-secret`: OpenShift pull secret from cloud.redhat.com
- `--release-image`: OpenShift release image URL
- `--resource-group-name`: Azure resource group for cluster resources
- `--vnet-id`: Full Azure resource ID of VNet
- `--subnet-id`: Full Azure resource ID of subnet
- `--network-security-group-id`: Full Azure resource ID of NSG
- `--sa-token-issuer-private-key-path`: Path to SA signing private key
- `--oidc-issuer-url`: OIDC issuer endpoint URL
- `--workload-identities-file`: JSON file with managed identity client IDs
- `--dns-zone-rg-name`: Resource group containing DNS zone (External DNS only)
- `--external-dns-domain`: DNS domain for External DNS (External DNS only)
- `--control-plane-operator-image`: Override CPO image (optional)
- `--assign-service-principal-roles`: Auto-assign Azure roles
- `--generate-ssh`: Generate SSH key for node access
- `--diagnostics-storage-account-type`: Boot diagnostics storage (Managed or Disabled)

### Cluster Destruction

```bash
hypershift destroy cluster azure \
    --name <cluster-name> \
    --location <azure-region> \
    --resource-group-name <managed-rg> \
    --azure-creds <credentials-file>
```

### NodePool Creation

```bash
hypershift create nodepool azure \
    --cluster-name <cluster-name> \
    --namespace <namespace> \
    --name <nodepool-name> \
    --node-count <count> \
    --azure-instance-type <vm-size> \
    --azure-creds <credentials-file> \
    [--marketplace-publisher <publisher>] \
    [--marketplace-offer <offer>] \
    [--marketplace-sku <sku>] \
    [--marketplace-version <version>] \
    [--image-generation Gen1|Gen2]
```

### Kubeconfig Generation

```bash
hypershift create kubeconfig \
    --name <cluster-name> \
    --namespace <namespace> \
    > <output-file>
```

### Operator Installation

```bash
hypershift install \
    --pull-secret <pull-secret-file> \
    --limit-crd-install Azure \
    [--external-dns-provider azure] \
    [--external-dns-credentials <azure-creds>] \
    [--external-dns-domain-filter <dns-zone>] \
    [--hypershift-image <operator-image>] \
    [--namespace <namespace>] \
    [--development]
```

## Azure CLI Quick Reference

### Identity Management

```bash
# List managed identities
az identity list \
    --resource-group <rg-name> \
    --output table

# Show identity details
az identity show \
    --name <identity-name> \
    --resource-group <rg-name>

# List federated credentials
az identity federated-credential list \
    --identity-name <identity-name> \
    --resource-group <rg-name> \
    --output table

# Show federated credential
az identity federated-credential show \
    --identity-name <identity-name> \
    --name <credential-name> \
    --resource-group <rg-name>
```

### Resource Group Management

```bash
# List resource groups
az group list --output table

# Create resource group
az group create \
    --name <rg-name> \
    --location <location>

# Delete resource group
az group delete \
    --name <rg-name> \
    --yes [--no-wait]
```

### Network Resources

```bash
# Create VNet
az network vnet create \
    --name <vnet-name> \
    --resource-group <rg-name> \
    --address-prefix <cidr> \
    --subnet-name <subnet-name> \
    --subnet-prefixes <subnet-cidr>

# Create NSG
az network nsg create \
    --name <nsg-name> \
    --resource-group <rg-name>

# List NSG rules
az network nsg rule list \
    --nsg-name <nsg-name> \
    --resource-group <rg-name> \
    --output table
```

### Storage Account (OIDC)

```bash
# Show storage account
az storage account show \
    --name <account-name> \
    --resource-group <rg-name>

# Delete storage account (OIDC issuer)
az storage account delete \
    --name <account-name> \
    --resource-group <rg-name>
```

### Virtual Machines

```bash
# List VMs
az vm list \
    --resource-group <rg-name> \
    --output table

# Show VM details with power state
az vm list \
    --resource-group <rg-name> \
    --show-details \
    --query "[].{Name:name, PowerState:powerState}" \
    --output table

# Get boot diagnostics
az vm boot-diagnostics get-boot-log \
    --resource-group <rg-name> \
    --name <vm-name>
```

## oc (OpenShift CLI) Quick Reference

The `oc` CLI is the recommended tool for OpenShift environments. While `oc` works for basic Kubernetes operations, `oc` provides additional OpenShift-specific features like project management, build operations, and route handling.

All examples in this guide use `oc` commands.

### Cluster Resources

```bash
# List HostedClusters
oc get hostedclusters -n <namespace>

# Get HostedCluster details
oc get hostedcluster <name> -n <namespace> -o yaml

# Describe HostedCluster (shows events)
oc describe hostedcluster <name> -n <namespace>

# Patch HostedCluster (upgrade)
oc patch hostedcluster/<name> -n <namespace> \
    --type merge \
    --patch '{"spec":{"release":{"image":"<new-image>"}}}'
```

### NodePool Resources

```bash
# List NodePools
oc get nodepools -n <namespace>

# Get NodePool details
oc get nodepool <name> -n <namespace> -o yaml

# Scale NodePool
oc scale nodepool/<name> -n <namespace> --replicas=<count>

# Patch NodePool
oc patch nodepool/<name> -n <namespace> \
    --type merge \
    --patch '{"spec":{"replicas":<count>}}'

# Delete NodePool
oc delete nodepool/<name> -n <namespace>
```

### Control Plane Pods

```bash
# List control plane pods
oc get pods -n clusters-<cluster-name>

# Get pod logs
oc logs -n clusters-<cluster-name> <pod-name> [-f]

# Describe pod
oc describe pod -n clusters-<cluster-name> <pod-name>

# Get pod resource usage
oc top pods -n clusters-<cluster-name>
```

### Machines

```bash
# List machines
oc get machines -n clusters-<cluster-name>

# Describe machine
oc describe machine -n clusters-<cluster-name> <machine-name>

# Delete machine (triggers replacement)
oc delete machine -n clusters-<cluster-name> <machine-name>
```

## Appendices

### Appendix A: Azure Permissions Reference

#### Azure Administrator Permissions

Required for setting up foundation resources:

**Subscription-Level Roles**:
- `Contributor`: Create and manage all types of Azure resources
- `User Access Administrator`: Manage user access to Azure resources (for role assignments)

**Microsoft Graph API Permissions**:
- `Application.ReadWrite.OwnedBy`: Create and manage service principals

#### Service Principal Permissions

For cluster operations (`azure-credentials.json`):

**Subscription-Level Roles**:
- `Contributor`: Manage cluster infrastructure (VMs, networking, storage)
- `User Access Administrator`: Assign roles to managed identities

**Alternative: Resource Group Scoped**:
- `Contributor` on specific resource groups (tighter security)
- `User Access Administrator` on resource groups containing managed identities

#### External DNS Service Principal

For automatic DNS management:

**DNS Zone Level**:
- `DNS Zone Contributor`: Manage DNS records in specific DNS zone

### Appendix B: Workload Identity Service Account Mapping

Complete mapping of Azure managed identities to OpenShift service accounts and their purposes:

| Component | Managed Identity | Service Accounts | Namespace | Purpose |
|-----------|------------------|------------------|-----------|---------|
| **Disk CSI** | `${CLUSTER_NAME}-disk-csi` | `azure-disk-csi-driver-node-sa`<br>`azure-disk-csi-driver-operator`<br>`azure-disk-csi-driver-controller-sa` | `openshift-cluster-csi-drivers` | Provision and attach Azure Disk volumes as persistent storage |
| **File CSI** | `${CLUSTER_NAME}-file-csi` | `azure-file-csi-driver-node-sa`<br>`azure-file-csi-driver-operator`<br>`azure-file-csi-driver-controller-sa` | `openshift-cluster-csi-drivers` | Provision and mount Azure Files as ReadWriteMany volumes |
| **Image Registry** | `${CLUSTER_NAME}-image-registry` | `registry`<br>`cluster-image-registry-operator` | `openshift-image-registry` | Manage image registry storage backend (Azure Blob Storage) |
| **Ingress** | `${CLUSTER_NAME}-ingress` | `ingress-operator` | `openshift-ingress-operator` | Configure Azure Load Balancers for ingress traffic routing |
| **Cloud Provider** | `${CLUSTER_NAME}-cloud-provider` | `azure-cloud-provider` | `openshift-cloud-controller-manager` | Integrate with Azure cloud APIs for node/volume management |
| **NodePool Mgmt** | `${CLUSTER_NAME}-nodepool-mgmt` | `capi-provider` | Control plane namespace | Provision and manage Azure VMs for worker nodes |
| **Network** | `${CLUSTER_NAME}-network` | `cloud-network-config-controller` | `openshift-cloud-network-config-controller` | Configure Azure networking components (routes, load balancer rules) |

**Federated Credential Pattern**:

Each managed identity has **2 federated credentials** for redundancy:
- One for the operator service account
- One for the controller/node service account

Total: 7 managed identities × 2 federated credentials = 14 federated credentials per cluster

### Appendix C: Resource Group Lifecycle

Understanding what gets created and when it gets deleted:

#### Persistent Resource Group

**Name**: Configurable (e.g., `hypershift-shared`, `os4-common`)

**Contents**:
- OIDC issuer (storage account)
- Managed identities (7 per cluster, but stored centrally)
- Federated credentials (14 per cluster)
- DNS zones (if using External DNS)

**Lifecycle**: Long-lived, survives cluster deletion

**Shared Across**: All hosted clusters in the environment

**Deletion**: Manual only, when decommissioning entire environment

#### Per-Cluster Resource Groups

**Pattern**: `${PREFIX}-<type>-rg`

**Types**:
1. **Managed RG** (`${PREFIX}-managed-rg`):
   - Worker VMs
   - Load balancers
   - Managed disks
   - NICs

2. **VNet RG** (`${PREFIX}-vnet-rg`):
   - Virtual network
   - Subnets

3. **NSG RG** (`${PREFIX}-nsg-rg`):
   - Network security group
   - NSG rules

**Lifecycle**: Created with cluster, deleted with cluster

**Deletion**: Via `task azure:delete-infra` or manual `az group delete`

### Appendix D: DNS Configuration Reference

#### DNS Flags and Their Effects

**`--base-domain`** (always required):
- Determines the ingress domain for application routes
- Formula: `apps.<cluster-name>.<base-domain>` or `apps.<base-domain>`
- Example: `--base-domain example.com` → `apps.mycluster.example.com`

**`--external-dns-domain`** (optional, External DNS only):
- Changes service publishing strategy from LoadBalancer to Route
- Sets custom DNS names for control plane services
- Formula: `api-<cluster-name>.<external-dns-domain>`
- Example: `--external-dns-domain azure.example.com` → `api-mycluster.azure.example.com`

**`--dns-zone-rg-name`** (required when using `--external-dns-domain`):
- Resource group containing the Azure DNS zone
- External DNS uses this to create DNS records
- Example: `--dns-zone-rg-name hypershift-shared`

#### Service Publishing Strategies

| Scenario | API Server | OAuth | Konnectivity | Ingress | DNS Management |
|----------|------------|-------|--------------|---------|----------------|
| **Without External DNS** | LoadBalancer | Route | Route | Route | Manual or Azure-provided LB DNS |
| **With External DNS** | Route | Route | Route | Route | Automatic via External DNS operator |

### Appendix E: Glossary

**HyperShift Terms**:

- **HostedCluster**: Custom resource representing a hosted OpenShift cluster with control plane running as pods
- **NodePool**: Custom resource representing a group of worker nodes with identical configuration
- **Control Plane**: Kubernetes components (API server, etcd, controllers) managing the cluster, running on management cluster
- **Data Plane**: Worker nodes where application workloads run, deployed as Azure VMs
- **Management Cluster**: OpenShift cluster hosting HyperShift operator and control plane pods
- **HyperShift Operator**: Kubernetes operator managing lifecycle of HostedClusters and NodePools

**Azure Terms**:

- **Workload Identity**: Azure managed identity used by OpenShift components for Azure API authentication
- **Managed Identity**: Azure AD identity without credentials, used for Azure service authentication
- **Federated Credential**: Trust relationship between Azure AD and external identity provider (OIDC)
- **OIDC Issuer**: OpenID Connect endpoint used for service account token validation
- **Service Principal**: Azure AD application identity with credentials for programmatic access
- **Resource Group**: Logical container for Azure resources with shared lifecycle
- **VNet (Virtual Network)**: Isolated network in Azure for resource communication
- **NSG (Network Security Group)**: Firewall rules controlling network traffic
- **Azure Blob Storage**: Object storage service used for OIDC issuer hosting

**OpenShift Terms**:

- **Service Account**: Kubernetes identity for pods to access cluster APIs
- **Cluster Operator**: OpenShift component managing specific cluster functionality
- **Machine**: Kubernetes resource representing a physical or virtual machine
- **ClusterVersion**: Resource tracking OpenShift version and upgrade status
- **Pull Secret**: Credentials for pulling OpenShift container images
- **Ignition**: System configuration tool for initial node setup

### Appendix F: Common Azure VM Sizes

**General Purpose (Dsv3-series)**:
- `Standard_D2s_v3`: 2 vCPU, 8 GB RAM - Default, balanced workloads
- `Standard_D4s_v3`: 4 vCPU, 16 GB RAM - Medium workloads
- `Standard_D8s_v3`: 8 vCPU, 32 GB RAM - Larger applications
- `Standard_D16s_v3`: 16 vCPU, 64 GB RAM - High-performance apps

**Memory Optimized (Esv3-series)**:
- `Standard_E2s_v3`: 2 vCPU, 16 GB RAM - Memory-intensive small
- `Standard_E4s_v3`: 4 vCPU, 32 GB RAM - Databases, caching
- `Standard_E8s_v3`: 8 vCPU, 64 GB RAM - Large databases
- `Standard_E16s_v3`: 16 vCPU, 128 GB RAM - In-memory analytics

**Compute Optimized (Fsv2-series)**:
- `Standard_F2s_v2`: 2 vCPU, 4 GB RAM - Batch processing
- `Standard_F4s_v2`: 4 vCPU, 8 GB RAM - Web servers
- `Standard_F8s_v2`: 8 vCPU, 16 GB RAM - Gaming servers
- `Standard_F16s_v2`: 16 vCPU, 32 GB RAM - CPU-intensive workloads

See [Azure VM sizes documentation](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes) for complete list.

## Reference Links

### HyperShift Documentation

- [HyperShift Project Home](https://hypershift.pages.dev)
- [HyperShift GitHub Repository](https://github.com/openshift/hypershift)
- [HyperShift Releases](https://github.com/openshift/hypershift/releases)
- [HyperShift Architecture Documentation](https://hypershift.pages.dev/reference/architecture/)

### Azure Documentation

- [Azure Workload Identity](https://azure.github.io/azure-workload-identity/docs/)
- [Azure Managed Identities](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/)
- [Azure DNS Zones](https://learn.microsoft.com/en-us/azure/dns/dns-zones-records)
- [Azure Virtual Networks](https://learn.microsoft.com/en-us/azure/virtual-network/)
- [Azure VM Sizes](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes)
- [Azure Network Security Groups](https://learn.microsoft.com/en-us/azure/virtual-network/network-security-groups-overview)

### OpenShift Documentation

- [OpenShift Container Platform Documentation](https://docs.openshift.com)
- [Cloud Credential Operator](https://docs.openshift.com/container-platform/latest/authentication/managing_cloud_provider_credentials/about-cloud-credential-operator.html)
- [OpenShift Release Images](https://quay.io/repository/openshift-release-dev/ocp-release)
- [Pull Secrets (cloud.redhat.com)](https://console.redhat.com/openshift/downloads)

### Tools and CLI

- [Azure CLI Installation](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli)
- [OpenShift CLI (oc) Downloads](https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/)
- [oc Installation](https://kubernetes.io/docs/tasks/tools/)
- [jq - Command-line JSON processor](https://jqlang.github.io/jq/)
- [go-task/task - Task runner](https://taskfile.dev)

### Community and Support

- [HyperShift Slack Channel](https://kubernetes.slack.com/archives/C02C4LXBW0T) (#hypershift)
- [OpenShift Community](https://www.openshift.com/community)
- [Red Hat Customer Portal](https://access.redhat.com) (for supported deployments)

## Quick Start Workflow

For quick reference, here's the complete workflow from scratch to running cluster:

```bash
# 1. Prerequisites (once per environment)
task prereq:validate
task prereq:keys
task azure:oidc

# 2. Per-cluster setup
task azure:identities
task azure:federated-creds
task azure:infra

# 3. Create cluster
task cluster:create

# 4. Wait for cluster
oc wait --for=condition=Available \
    hostedcluster/${CLUSTER_NAME} \
    -n ${CLUSTER_NAMESPACE} \
    --timeout=30m

# 5. Access cluster
hypershift create kubeconfig \
    --name ${CLUSTER_NAME} \
    --namespace ${CLUSTER_NAMESPACE} \
    > kubeconfig
export KUBECONFIG=kubeconfig
oc get nodes
oc get co

# 6. Cleanup (when done)
task cluster:destroy
task azure:delete-infra
# Optionally: task azure:delete-identities (permanent)
```

## Configuration File Examples

### Taskfile.yml Variables

```yaml
vars:
  # Cluster identification
  PREFIX: 'myprefix'
  CLUSTER_NAME: '${PREFIX}-hc'
  CLUSTER_NAMESPACE: 'clusters'

  # Azure configuration
  SUBSCRIPTION_ID: 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx'
  TENANT_ID: 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx'
  LOCATION: 'eastus'

  # Resource groups
  PERSISTENT_RG_NAME: 'hypershift-shared'
  MANAGED_RG_NAME: '${PREFIX}-managed-rg'
  VNET_RG_NAME: '${PREFIX}-vnet-rg'
  NSG_RG_NAME: '${PREFIX}-nsg-rg'

  # Networking
  VNET_NAME: '${PREFIX}-vnet'
  VNET_SUBNET1: '${PREFIX}-subnet-1'
  NSG: '${PREFIX}-nsg'

  # OIDC and identities
  OIDC_STORAGE_ACCOUNT_NAME: 'myoidc1234567890'
  OIDC_ISSUER_URL: 'https://${OIDC_STORAGE_ACCOUNT_NAME}.blob.core.windows.net/${OIDC_STORAGE_ACCOUNT_NAME}'
  SA_TOKEN_ISSUER_PRIVATE_KEY_PATH: 'serviceaccount-signer.private'

  # DNS
  PARENT_DNS_ZONE: 'example.com'

  # OpenShift
  RELEASE_IMAGE: 'quay.io/openshift-release-dev/ocp-release:4.21.0-x86_64'
  NODE_POOL_REPLICAS: '2'

  # Credentials
  AZURE_CREDS: 'azure-credentials.json'
  PULL_SECRET: 'pull-secret.json'
```

### azure-credentials.json

```json
{
  "subscriptionId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "tenantId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "clientId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "clientSecret": "your-service-principal-secret"
}
```

### workload-identities.json (generated)

```json
{
  "disk_csi": {
    "client_id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
  },
  "file_csi": {
    "client_id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
  },
  "image_registry": {
    "client_id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
  },
  "ingress": {
    "client_id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
  },
  "cloud_provider": {
    "client_id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
  },
  "nodepool_mgmt": {
    "client_id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
  },
  "network": {
    "client_id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
  }
}
```
