# Day 2 Operations

This guide covers common operational tasks for managing HyperShift hosted clusters after initial deployment.

!!! note "Persona: HyperShift Administrator"

    Day 2 operations are performed on running clusters to manage capacity, upgrade versions, and maintain cluster health.

## Prerequisites

- At least one HostedCluster successfully deployed
- oc/oc access to the management cluster
- HyperShift CLI binary available

## Managing NodePools

NodePools define the worker node configuration and count for your hosted cluster. You can add, modify, scale, and delete NodePools as your workload requirements change.

### Understanding NodePools

**What is a NodePool?**
- Defines a group of worker nodes with identical configuration
- Specifies VM size, node count, and Azure-specific settings
- Can have multiple NodePools per cluster with different configurations
- Each NodePool manages its own set of Azure VMs

**Use cases for multiple NodePools**:
- Different VM sizes for different workload types (CPU vs memory optimized)
- Separate pools for different availability zones
- Testing new VM configurations before migrating workloads
- Gradual node upgrades with blue/green approaches

### Adding Additional NodePools

Create a new NodePool with different configuration:

```bash
# Set variables (from your Taskfile or environment)
CLUSTER_NAME="myprefix-hc"
CLUSTER_NAMESPACE="clusters"
AZURE_CREDS="azure-credentials.json"

# Create a new NodePool with larger VMs
hypershift create nodepool azure \
    --cluster-name "${CLUSTER_NAME}" \
    --namespace "${CLUSTER_NAMESPACE}" \
    --name "${CLUSTER_NAME}-large-workers" \
    --node-count 3 \
    --azure-instance-type Standard_D8s_v3 \
    --azure-creds ${AZURE_CREDS}
```

**Common VM sizes**:
- `Standard_D2s_v3`: 2 vCPU, 8GB RAM (default, balanced)
- `Standard_D4s_v3`: 4 vCPU, 16GB RAM (larger workloads)
- `Standard_D8s_v3`: 8 vCPU, 32GB RAM (high-performance)
- `Standard_E4s_v3`: 4 vCPU, 32GB RAM (memory-optimized)
- `Standard_F8s_v2`: 8 vCPU, 16GB RAM (compute-optimized)

See [Azure VM sizes](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes) for complete list.

**Verify NodePool creation**:
```bash
# Check NodePool status
oc get nodepool -n ${CLUSTER_NAMESPACE}

# Watch nodes join the cluster (switch to hosted cluster context)
export KUBECONFIG=${CLUSTER_NAME}-kubeconfig
oc get nodes -w
```

### Configuring NodePool Marketplace Images

For OpenShift 4.20+, NodePools automatically use the release payload defaults. For custom configurations:

**Option 1: Specify VM Generation Only**
```bash
hypershift create nodepool azure \
    --cluster-name "${CLUSTER_NAME}" \
    --namespace "${CLUSTER_NAMESPACE}" \
    --name "${CLUSTER_NAME}-gen1-workers" \
    --node-count 2 \
    --image-generation Gen1 \
    --azure-creds ${AZURE_CREDS}
```

**Option 2: Use Custom Marketplace Image**
```bash
hypershift create nodepool azure \
    --cluster-name "${CLUSTER_NAME}" \
    --namespace "${CLUSTER_NAMESPACE}" \
    --name "${CLUSTER_NAME}-custom-workers" \
    --node-count 2 \
    --marketplace-publisher azureopenshift \
    --marketplace-offer aro4 \
    --marketplace-sku aro_421 \
    --marketplace-version 421.0.20250101 \
    --azure-creds ${AZURE_CREDS}
```

### Scaling NodePools

Adjust the number of worker nodes in a NodePool:

**Scale using oc**:
```bash
# Scale to 5 replicas
oc scale nodepool/${CLUSTER_NAME} \
    -n ${CLUSTER_NAMESPACE} \
    --replicas=5

# Verify scaling
oc get nodepool -n ${CLUSTER_NAMESPACE}
```

**Scale using patch**:
```bash
# Scale specific NodePool
oc patch nodepool/${CLUSTER_NAME}-large-workers \
    -n ${CLUSTER_NAMESPACE} \
    --type merge \
    --patch '{"spec":{"replicas":10}}'
```

**Scale using edit**:
```bash
# Edit NodePool interactively
oc edit nodepool/${CLUSTER_NAME} -n ${CLUSTER_NAMESPACE}

# Modify spec.replicas field and save
```

**Monitor scaling progress**:
```bash
# Watch NodePool status
oc get nodepool -n ${CLUSTER_NAMESPACE} -w

# Watch machines being created
oc get machines -n clusters-${CLUSTER_NAME} -w

# In hosted cluster context, watch nodes joining
export KUBECONFIG=${CLUSTER_NAME}-kubeconfig
oc get nodes -w
```

!!! tip "Scaling Best Practices"

    - Scale gradually in production (add/remove 1-2 nodes at a time)
    - Monitor cluster resource usage before scaling down
    - Ensure workloads have PodDisruptionBudgets configured
    - Drain nodes before scaling down to avoid disruption

### Deleting NodePools

Remove a NodePool when no longer needed:

```bash
# Delete a specific NodePool
oc delete nodepool/${CLUSTER_NAME}-large-workers \
    -n ${CLUSTER_NAMESPACE}

# Verify deletion
oc get nodepool -n ${CLUSTER_NAMESPACE}
```

!!! warning "NodePool Deletion"

    Deleting a NodePool:

    - Immediately deletes the NodePool resource
    - Terminates all Azure VMs in that NodePool
    - Evicts all pods running on those nodes
    - Cannot be undone

    **Before deleting**:
    - Drain workloads to other nodes
    - Ensure sufficient capacity remains
    - Verify no critical workloads are pinned to specific nodes

## Upgrading Clusters

Upgrade hosted clusters to new OpenShift versions by changing the release image.

### Understanding HyperShift Upgrades

**How upgrades work**:
1. Update the HostedCluster `spec.release.image` field
2. Control plane components upgrade first (pods on management cluster)
3. NodePools upgrade next (worker nodes replaced or upgraded)
4. Cluster operators coordinate the upgrade process

**Upgrade characteristics**:
- Control plane upgrades are fast (pod restarts)
- Worker node upgrades create new VMs and drain old ones
- Minimal downtime with proper planning
- Can upgrade control plane and NodePools separately

### Upgrading the Control Plane

Update the cluster release image:

```bash
# Variables
CLUSTER_NAME="myprefix-hc"
CLUSTER_NAMESPACE="clusters"
NEW_RELEASE_IMAGE="quay.io/openshift-release-dev/ocp-release:4.21.1-x86_64"

# Patch the HostedCluster
oc patch hostedcluster/${CLUSTER_NAME} \
    -n ${CLUSTER_NAMESPACE} \
    --type merge \
    --patch "{\"spec\":{\"release\":{\"image\":\"${NEW_RELEASE_IMAGE}\"}}}"
```

**Monitor control plane upgrade**:
```bash
# Watch HostedCluster status
oc get hostedcluster ${CLUSTER_NAME} \
    -n ${CLUSTER_NAMESPACE} \
    -w

# Check control plane pod rollouts
oc get pods -n clusters-${CLUSTER_NAME} -w

# In hosted cluster context, check cluster version
export KUBECONFIG=${CLUSTER_NAME}-kubeconfig
oc get clusterversion
```

**Expected progression**:
```
# HostedCluster shows progressing
NAME          VERSION   AVAILABLE   PROGRESSING   MESSAGE
myprefix-hc   4.21.0    True        True          Upgrading to 4.21.1

# After completion
NAME          VERSION   AVAILABLE   PROGRESSING   MESSAGE
myprefix-hc   4.21.1    True        False         The hosted cluster is available
```

### Upgrading NodePools

Upgrade worker nodes to match the control plane version:

```bash
# Option 1: Patch the NodePool release
oc patch nodepool/${CLUSTER_NAME} \
    -n ${CLUSTER_NAMESPACE} \
    --type merge \
    --patch "{\"spec\":{\"release\":{\"image\":\"${NEW_RELEASE_IMAGE}\"}}}"

# Option 2: Edit NodePool interactively
oc edit nodepool/${CLUSTER_NAME} -n ${CLUSTER_NAMESPACE}
# Update spec.release.image and save
```

**Monitor NodePool upgrade**:
```bash
# Watch NodePool status
oc get nodepool -n ${CLUSTER_NAMESPACE} -w

# Watch machines being replaced
oc get machines -n clusters-${CLUSTER_NAME} -w

# In hosted cluster context, watch node versions
export KUBECONFIG=${CLUSTER_NAME}-kubeconfig
oc get nodes -w
```

**Upgrade strategies**:

**Replace Strategy** (default):
- Creates new VMs with new version
- Drains and deletes old VMs
- Rolling replacement maintains capacity
- Safer but slower

**InPlace Strategy**:
- Upgrades existing VMs in place
- Faster but requires node reboots
- Less common in cloud environments

### Upgrade Best Practices

!!! tip "Pre-Upgrade Checklist"

    Before upgrading:

    - [ ] Review release notes for breaking changes
    - [ ] Test upgrade in non-production cluster first
    - [ ] Ensure cluster operators are healthy: `oc get co`
    - [ ] Verify sufficient capacity in management cluster
    - [ ] Back up critical data and configurations
    - [ ] Notify users of potential disruption
    - [ ] Plan maintenance window if required

**Upgrade sequence**:
1. Upgrade control plane first
2. Wait for control plane to become Available
3. Verify all cluster operators are healthy
4. Upgrade NodePools (one at a time in production)
5. Verify cluster functionality

**Rollback**:
Downgrades are not supported. If issues occur:
- Fix issues in upgraded version
- Or restore from backup/recreate cluster

## Monitoring Cluster Health

### Checking Cluster Status

**From management cluster**:
```bash
# HostedCluster health
oc get hostedcluster ${CLUSTER_NAME} -n ${CLUSTER_NAMESPACE}

# NodePool health
oc get nodepool -n ${CLUSTER_NAMESPACE}

# Control plane pods
oc get pods -n clusters-${CLUSTER_NAME}

# Control plane resource usage
oc top pods -n clusters-${CLUSTER_NAME}
```

**From hosted cluster**:
```bash
# Switch to hosted cluster
export KUBECONFIG=${CLUSTER_NAME}-kubeconfig

# Cluster version and available updates
oc get clusterversion

# Cluster operators
oc get co

# Node health
oc get nodes

# Node resource usage
oc top nodes
```

### Key Health Indicators

**Healthy HostedCluster**:
```yaml
status:
  conditions:
  - type: Available
    status: "True"
  - type: Progressing
    status: "False"
  - type: Degraded
    status: "False"
```

**Healthy NodePool**:
```yaml
status:
  replicas: 3        # Matches desired count
  readyReplicas: 3   # All nodes ready
  conditions:
  - type: Ready
    status: "True"
```

**Healthy Cluster Operators**:
```bash
# All operators should show AVAILABLE=True, DEGRADED=False
oc get co

NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED
authentication                             4.21.0    True        False         False
cloud-credential                           4.21.0    True        False         False
cluster-autoscaler                         4.21.0    True        False         False
...
```

### Cluster Metrics and Logging

**View control plane metrics** (from management cluster):
```bash
# CPU and memory usage
oc top pods -n clusters-${CLUSTER_NAME}

# Node resource allocation on management cluster
oc top nodes
```

**View hosted cluster metrics**:
```bash
# Switch to hosted cluster
export KUBECONFIG=${CLUSTER_NAME}-kubeconfig

# Worker node metrics
oc top nodes

# Pod metrics across cluster
oc top pods --all-namespaces
```

**Access control plane logs**:
```bash
# From management cluster
oc logs -n clusters-${CLUSTER_NAME} deployment/kube-apiserver
oc logs -n clusters-${CLUSTER_NAME} deployment/kube-controller-manager
oc logs -n clusters-${CLUSTER_NAME} statefulset/etcd
```

## Modifying Cluster Configuration

### Changing Cluster Networking

Update cluster network settings:

```bash
# Edit HostedCluster
oc edit hostedcluster/${CLUSTER_NAME} -n ${CLUSTER_NAMESPACE}

# Modify spec.networking fields
# Note: Some changes require cluster recreation
```

!!! warning "Network Configuration Changes"

    Many networking changes cannot be applied to running clusters and require recreation:

    - Service network CIDR
    - Pod network CIDR
    - Network type (OVNKubernetes vs others)

    Changes that can be applied:
    - DNS configuration
    - Proxy settings

### Managing Cluster Credentials

**Rotate kubeadmin password**:
```bash
# From management cluster
oc delete secret kubeadmin -n clusters-${CLUSTER_NAME}

# Controller will regenerate with new password
# Retrieve new password
oc get secret kubeadmin \
    -n clusters-${CLUSTER_NAME} \
    -o jsonpath='{.data.password}' | base64 -d
```

**Add cluster administrators**:
```bash
# From hosted cluster
export KUBECONFIG=${CLUSTER_NAME}-kubeconfig

# Grant cluster-admin to user
oc create clusterrolebinding admin-user \
    --clusterrole=cluster-admin \
    --user=admin@example.com
```

## NodePool Advanced Configuration

### Configuring Node Labels and Taints

**Add labels to NodePool**:
```bash
oc patch nodepool/${CLUSTER_NAME} \
    -n ${CLUSTER_NAMESPACE} \
    --type merge \
    --patch '{"spec":{"nodeLabels":{"workload-type":"database"}}}'
```

**Add taints to NodePool**:
```bash
oc edit nodepool/${CLUSTER_NAME} -n ${CLUSTER_NAMESPACE}

# Add to spec:
spec:
  taints:
  - key: dedicated
    value: database
    effect: NoSchedule
```

**Verify labels and taints**:
```bash
# From hosted cluster
export KUBECONFIG=${CLUSTER_NAME}-kubeconfig
oc get nodes --show-labels
oc describe node <node-name> | grep Taints
```

### Configuring Root Volume Size

Increase root volume size for NodePool VMs:

```bash
oc patch nodepool/${CLUSTER_NAME} \
    -n ${CLUSTER_NAMESPACE} \
    --type merge \
    --patch '{"spec":{"platform":{"azure":{"diskSizeGB":256}}}}'
```

!!! note "Volume Size Changes"

    - Only affects new nodes created after the change
    - Existing nodes keep their current disk size
    - Scale down and up to replace nodes with new disk size

## Next Steps

- [Troubleshooting](07-troubleshooting.md) - Diagnose and resolve common issues
- [Reference](08-reference.md) - CLI commands and configuration reference

## Quick Reference

### Common Operations

```bash
# Add NodePool
hypershift create nodepool azure \
    --cluster-name ${CLUSTER_NAME} \
    --namespace ${CLUSTER_NAMESPACE} \
    --name ${CLUSTER_NAME}-new-pool \
    --node-count 3 \
    --azure-instance-type Standard_D4s_v3 \
    --azure-creds ${AZURE_CREDS}

# Scale NodePool
oc scale nodepool/${CLUSTER_NAME} \
    -n ${CLUSTER_NAMESPACE} \
    --replicas=5

# Upgrade cluster
NEW_VERSION="quay.io/openshift-release-dev/ocp-release:4.21.1-x86_64"
oc patch hostedcluster/${CLUSTER_NAME} \
    -n ${CLUSTER_NAMESPACE} \
    --type merge \
    --patch "{\"spec\":{\"release\":{\"image\":\"${NEW_VERSION}\"}}}"

# Delete NodePool
oc delete nodepool/${CLUSTER_NAME}-new-pool \
    -n ${CLUSTER_NAMESPACE}

# Check cluster health
oc get hostedcluster,nodepool -n ${CLUSTER_NAMESPACE}
oc get co  # From hosted cluster context
```
