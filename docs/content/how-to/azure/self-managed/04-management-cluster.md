# Management Cluster Setup

This guide covers installing the HyperShift operator on your management cluster using Taskfile automation. The operator manages the lifecycle of hosted clusters and orchestrates control plane deployments.

!!! note "Persona: HyperShift Administrator"

    This phase requires cluster-admin access to the OpenShift management cluster.

## Overview

Preparing your management cluster involves:

1. **Verification**: Confirm cluster access and capacity
2. **HyperShift Operator Installation**: Deploy the operator that manages HostedCluster resources
3. **External DNS Configuration** (Optional): Enable automatic DNS record management for production
4. **Verification**: Ensure the operator is ready to create hosted clusters

**Why This Matters**: The HyperShift operator is the core component that orchestrates all hosted cluster operations. It watches for HostedCluster custom resources and provisions the corresponding control plane components.

## Prerequisites

Before beginning, ensure you have:

- Cluster-admin access to your OpenShift management cluster
- Completed [Azure Foundation Setup](03-azure-foundation.md)
- Taskfile configuration in `hack/dev-preview/Taskfile.yml`
- Valid OpenShift pull secret from [cloud.redhat.com](https://console.redhat.com/openshift/downloads)
- HyperShift CLI binary ([installation covered by `task prereq:validate`](03-azure-foundation.md#step-1-validate-prerequisites))

Required Taskfile variables:

```yaml
PULL_SECRET: 'pull-secret.json'
HYPERSHIFT_NAMESPACE: 'hypershift'  # Optional, defaults to 'hypershift'
PARENT_DNS_ZONE: 'example.com'      # Required for External DNS option
```

## Installing the HyperShift Operator

### Step 1: Verify Management Cluster Access

Verify you have the required access to the management cluster:

```bash
task mgmt-cluster:verify-access
```

**What this does**:
- Tests oc connectivity to the cluster
- Verifies cluster-admin permissions (`oc auth can-i '*' '*'`)
- Confirms you can deploy resources

**Expected output**:
```
Verifying management cluster access...
Kubernetes control plane is running at https://api.management-cluster.example.com:6443
✓ Cluster-admin permissions confirmed
```

**Under the hood** (see `tasks/mgmt-cluster.yml`):
```bash
oc cluster-info
oc auth can-i '*' '*'
```

### Step 2: Choose Your DNS Strategy

Select one of two installation paths based on your DNS requirements:

#### Option A: With External DNS (Recommended for Production)

**Best for**: Production environments, multiple clusters, custom domains

**Characteristics**:
- Automatic DNS record creation/deletion
- Custom branded domains (e.g., `api.my-cluster.example.com`)
- Requires Azure DNS zone and service principal

**What you'll need**:
- Azure DNS zone configured in `PARENT_DNS_ZONE` variable
- Taskfile will automatically create service principal and credentials

**Proceed to**: [Step 3A](#step-3a-install-with-external-dns-production-mode)

#### Option B: Without External DNS (Simpler for Dev/Test)

**Best for**: Development, testing, proof-of-concept

**Characteristics**:
- Uses Azure LoadBalancer DNS names
- No DNS zones or additional credentials needed
- Simpler setup with fewer moving parts

**What you'll need**:
- Only pull secret (already configured)
- No additional Azure resources

**Proceed to**: [Step 3B](#step-3b-install-without-external-dns-simple-mode)

!!! tip "Recommendation"

    Start with **Option B (Without External DNS)** for your first cluster to learn the basics. Once comfortable, reinstall with Option A for production deployments.

### Step 3A: Install with External DNS (Production Mode)

Install the HyperShift operator with External DNS for automatic DNS management:

```bash
task mgmt-cluster:setup-with-dns
```

**What this does**:
1. Verifies management cluster access
2. Creates Azure service principal with DNS Zone Contributor role
3. Generates `azure_mgmt.json` credentials file
4. Creates Kubernetes secret `azure-config-file` in default namespace
5. Installs HyperShift operator with External DNS configuration
6. Verifies both operator and External DNS pods are running

**Under the hood** (see `tasks/mgmt-cluster.yml`):
```bash
# Create service principal
az ad sp create-for-rbac \
  --name "hypershift-external-dns-${PREFIX}" \
  --role "DNS Zone Contributor" \
  --scopes "/subscriptions/${SUBSCRIPTION_ID}/resourceGroups/${PERSISTENT_RG_NAME}"

# Create Kubernetes secret
oc create secret generic azure-config-file \
  -n default \
  --from-file=azure_mgmt.json

# Install operator with External DNS
hypershift install \
  --external-dns-provider=azure \
  --external-dns-credentials azure_mgmt.json \
  --pull-secret ${PULL_SECRET} \
  --external-dns-domain-filter ${PARENT_DNS_ZONE} \
  --limit-crd-install Azure
```

**Expected output**:
```
Verifying management cluster access...
✓ Cluster-admin permissions confirmed
Creating service principal for External DNS...
✓ Created azure_mgmt.json
Creating azure-config-file secret in default namespace...
✓ Created secret azure-config-file
Installing HyperShift operator with External DNS...
  DNS Zone: example.com
✓ HyperShift operator and External DNS installed in namespace hypershift
Verifying HyperShift operator installation...

Pods in hypershift namespace:
NAME                           READY   STATUS    RESTARTS   AGE
external-dns-xxxxx-xxxxx       1/1     Running   0          30s
operator-xxxxx-xxxxx           1/1     Running   0          30s

✓ HyperShift operator verification complete
✓ External DNS verification complete
```

**Advanced configuration**:

To use a custom operator image or namespace:
```yaml
# In Taskfile.yml
vars:
  HYPERSHIFT_IMAGE: 'quay.io/hypershift/hypershift:custom-tag'
  HYPERSHIFT_NAMESPACE: 'custom-namespace'  # Optional, defaults to 'hypershift'
```

### Step 3B: Install without External DNS (Simple Mode)

Install the HyperShift operator for simpler dev/test deployments:

```bash
task mgmt-cluster:setup
```

**What this does**:
1. Verifies management cluster access
2. Installs HyperShift operator in `hypershift` namespace
3. Limits CRD installation to Azure-specific resources only
4. Verifies operator pod is running

**Under the hood** (see `tasks/mgmt-cluster.yml`):
```bash
hypershift install \
  --pull-secret ${PULL_SECRET} \
  --limit-crd-install Azure
```

**Expected output**:
```
Verifying management cluster access...
✓ Cluster-admin permissions confirmed
Installing HyperShift operator (without External DNS)...
✓ HyperShift operator installed in namespace hypershift
Verifying HyperShift operator installation...

Pods in hypershift namespace:
NAME                     READY   STATUS    RESTARTS   AGE
operator-xxxxx-xxxxx     1/1     Running   0          30s

Operator deployment:
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
operator   1/1     1            1           30s

✓ HyperShift operator verification complete
```

**Advanced configuration**:

To use a custom operator image or namespace:
```yaml
# In Taskfile.yml
vars:
  HYPERSHIFT_IMAGE: 'quay.io/hypershift/hypershift:custom-tag'
  HYPERSHIFT_NAMESPACE: 'custom-namespace'
```

## Verifying the Installation

### Check Operator Status

Verify the operator is running correctly:

```bash
task mgmt-cluster:verify
```

**What this shows**:
- Pods in hypershift namespace
- Operator deployment status
- HyperShift CRDs installed
- Recent operator logs

**Example output**:
```
Verifying HyperShift operator installation...

Pods in hypershift namespace:
NAME                     READY   STATUS    RESTARTS   AGE
operator-xxxxx-xxxxx     1/1     Running   0          2m

Operator deployment:
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
operator   1/1     1            1           2m

HyperShift CRDs:
certificates.hypershift.openshift.io
clustersizingconfigurations.hypershift.openshift.io
hostedclusters.hypershift.openshift.io
hostedcontrolplanes.hypershift.openshift.io
nodepools.hypershift.openshift.io
...

Operator logs (last 20 lines):
2025-01-17T10:30:00.000Z INFO Starting workers
2025-01-17T10:30:00.000Z INFO Started controllers

✓ HyperShift operator verification complete
```

### Check External DNS (if installed)

For installations with External DNS:

```bash
task mgmt-cluster:verify-external-dns
```

**What this shows**:
- External DNS deployment status
- Recent External DNS logs
- Connection to Azure DNS

### Check Management Cluster Capacity

Verify the management cluster has sufficient resources:

```bash
task mgmt-cluster:check-capacity
```

**What this shows**:
- Current node resource usage
- Available capacity for hosted clusters

**Resource requirements**:
Each hosted cluster needs approximately:
- 4 CPU cores
- 8 GB memory

Plan your management cluster capacity accordingly.

## Understanding the Installation

### What Gets Deployed

The installation creates:

**1. Namespace**: `hypershift` (configurable via `HYPERSHIFT_NAMESPACE`)

**2. HyperShift Operator Deployment**:
- Watches for HostedCluster and NodePool resources
- Provisions control plane pods
- Manages cluster lifecycle

**3. CRDs (Custom Resource Definitions)**:
- `hostedclusters.hypershift.openshift.io`
- `nodepools.hypershift.openshift.io`
- `certificates.hypershift.openshift.io`
- Related scheduling and configuration resources

**4. RBAC Resources**:
- ServiceAccounts for operator
- ClusterRoles and ClusterRoleBindings
- Permissions to manage cluster-wide resources

**5. External DNS Deployment** (if enabled):
- Watches services and routes in hosted cluster namespaces
- Creates/updates DNS records in Azure DNS
- Uses service principal credentials from Kubernetes secret

### Operator Responsibilities

The HyperShift operator:
- Watches for HostedCluster custom resources in configured namespaces
- Provisions control plane components as pods on the management cluster
- Manages control plane lifecycle (creation, upgrades, deletion)
- Coordinates with Azure APIs for infrastructure provisioning
- Handles PKI and certificate rotation for control planes

### External DNS Responsibilities (if enabled)

External DNS:
- Watches Kubernetes services and OpenShift routes
- Creates DNS A records in Azure DNS for:
  - API server endpoints
  - OAuth server endpoints
  - Konnectivity server endpoints
  - Application ingress routes
- Removes DNS records when services are deleted
- Uses Azure credentials from `azure-config-file` secret

## Troubleshooting

### Operator Pod Not Starting

**Symptom**: Operator pod stuck in `Pending` or `CrashLoopBackOff`

**Check**:
```bash
oc describe pod -n hypershift -l name=operator
```

**Common causes**:
- Insufficient resources on management cluster
- Image pull errors (invalid pull secret)
- Admission webhook failures

**Solutions**:
- Verify pull secret is valid
- Check management cluster has available CPU/memory
- Review pod events and logs

### External DNS Not Creating Records

**Symptom**: DNS records not appearing in Azure DNS zone

**Check**:
```bash
task mgmt-cluster:verify-external-dns
```

**Common causes**:
- Service principal lacks DNS Zone Contributor permissions
- DNS zone name doesn't match `PARENT_DNS_ZONE`
- Secret `azure-config-file` not created or incorrect

**Solutions**:
```bash
# Verify secret exists
oc get secret azure-config-file -n default

# Check service principal permissions
az role assignment list \
  --assignee $(jq -r '.aadClientId' < azure_mgmt.json) \
  --output table

# Recreate DNS resources if needed
task mgmt-cluster:delete-dns-resources
task mgmt-cluster:setup-with-dns
```

### CRD Installation Issues

**Symptom**: Errors about unknown resource types

**Check**:
```bash
oc get crd | grep hypershift
```

**Solution**:
Reinstall the operator:
```bash
task mgmt-cluster:cleanup
task mgmt-cluster:setup
```

## Uninstalling the Operator

### Before Uninstalling

!!! danger "Delete Hosted Clusters First"

    **Never uninstall the operator while hosted clusters exist!**

    The operator manages the lifecycle of hosted clusters. Removing it while clusters exist will leave orphaned resources.

### Check for Existing Clusters

The uninstall task automatically checks for existing clusters:

```bash
task mgmt-cluster:cleanup
```

If hosted clusters exist, you'll see:
```
❌ WARNING: Found 2 HostedCluster resource(s)

NAMESPACE   NAME          VERSION   ...
clusters    myprefix-hc   4.21.0    ...

Delete all hosted clusters before uninstalling the operator!
```

### Complete Cleanup

To uninstall the operator and remove DNS resources, use the cleanup task that matches your installation:

**For basic installations (without External DNS):**
```bash
task mgmt-cluster:cleanup
```

**For installations with External DNS:**
```bash
task mgmt-cluster:cleanup-with-dns
```

**What gets removed** (via `hypershift install render | oc delete -f -`):
- HyperShift operator deployment and all related resources
- External DNS deployment (if installed)
- ServiceAccounts, Roles, RoleBindings, and Services
- All Kubernetes resources created by the install command
- Kubernetes secret `azure-config-file`
- Local `azure_mgmt.json` file

**What persists**:
- CRDs (Custom Resource Definitions)
- Service principal in Azure AD (must delete manually if not needed)

**To remove service principal** (if no longer needed):
```bash
az ad sp delete --id $(az ad sp list --display-name 'hypershift-external-dns-${PREFIX}' --query '[0].appId' -o tsv)
```

## Next Steps

With the HyperShift operator installed and verified, you're ready to:

- [Create Hosted Cluster](05-hosted-cluster.md) - Deploy your first hosted OpenShift cluster
- Configure cluster-specific settings in Taskfile.yml
- Plan your hosted cluster deployment strategy

**What's Next**: The hosted cluster creation process will use the operator you just installed to provision Azure infrastructure and deploy a complete OpenShift control plane on your management cluster.

## Quick Reference

### Essential Commands

```bash
# Simple installation (dev/test)
task mgmt-cluster:setup

# Production installation with External DNS
task mgmt-cluster:setup-with-dns

# Verify installation
task mgmt-cluster:verify
task mgmt-cluster:verify-external-dns  # If using External DNS

# Check capacity
task mgmt-cluster:check-capacity

# Cleanup (choose based on how you installed)
task mgmt-cluster:cleanup              # For simple installations
task mgmt-cluster:cleanup-with-dns     # For External DNS installations
```

### Task Summary

| Task | Purpose | When to Use |
|------|---------|-------------|
| `task mgmt-cluster:verify-access` | Verify cluster access and permissions | Before installation |
| `task mgmt-cluster:setup` | Install operator without External DNS | Dev/test environments |
| `task mgmt-cluster:setup-with-dns` | Install operator with External DNS | Production environments |
| `task mgmt-cluster:verify` | Verify operator installation | After installation |
| `task mgmt-cluster:verify-external-dns` | Verify External DNS installation | After installation with DNS |
| `task mgmt-cluster:check-capacity` | Check management cluster capacity | Before creating clusters |
| `task mgmt-cluster:cleanup` | Uninstall operator WITHOUT External DNS | Teardown simple installations |
| `task mgmt-cluster:cleanup-with-dns` | Uninstall operator WITH External DNS | Teardown External DNS installations |
