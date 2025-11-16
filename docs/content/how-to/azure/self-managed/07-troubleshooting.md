# Troubleshooting

This guide provides diagnostic procedures and solutions for common issues encountered with HyperShift hosted clusters on Azure.

## Common Issues

### Cluster Creation Failures

#### Authentication Errors with Workload Identities

**Symptom**: Cluster creation fails with "failed to authenticate" or Azure authentication errors in control plane pods.

**Common error messages**:
```
Failed to get managed identity credential: AADSTS70021: No matching federated identity record found
```

**Root causes**:
- Federated credentials not created or misconfigured
- OIDC issuer URL mismatch
- Service account namespace/name mismatch
- Managed identity doesn't exist

**Diagnostic steps**:
```bash
# 1. Verify managed identities exist
az identity list \
    --resource-group ${PERSISTENT_RG_NAME} \
    --query "[?contains(name, '${CLUSTER_NAME}')].{Name:name, ClientId:clientId}" \
    --output table

# 2. Check federated credentials for a specific identity
task azure:list-federated-creds

# Or manually check:
az identity federated-credential list \
    --identity-name "${CLUSTER_NAME}-disk-csi" \
    --resource-group ${PERSISTENT_RG_NAME} \
    --output table

# 3. Verify OIDC issuer is accessible
curl -s "${OIDC_ISSUER_URL}/.well-known/openid-configuration" | jq .

# 4. Verify OIDC issuer URL matches
kubectl get hostedcluster ${CLUSTER_NAME} \
    -n ${CLUSTER_NAMESPACE} \
    -o jsonpath='{.spec.issuerURL}'
```

**Solutions**:

**Solution 1: Recreate federated credentials**
```bash
# Delete and recreate federated credentials
task azure:delete-federated-creds
task azure:federated-creds
```

**Solution 2: Verify configuration matches**
```bash
# Ensure PREFIX and CLUSTER_NAME are identical between:
# - Identity creation (task azure:identities)
# - Federated credential creation (task azure:federated-creds)
# - Cluster creation (task cluster:create)

# Check Taskfile.yml values
grep -E 'PREFIX|CLUSTER_NAME' hack/dev-preview/Taskfile.yml
```

**Solution 3: Verify OIDC issuer accessibility**
```bash
# Test OIDC endpoint
curl -v "${OIDC_ISSUER_URL}/.well-known/openid-configuration"

# Ensure storage account allows public access
az storage account show \
    --name ${OIDC_STORAGE_ACCOUNT_NAME} \
    --resource-group ${PERSISTENT_RG_NAME} \
    --query "allowBlobPublicAccess"
```

#### Precondition Failures

**Symptom**: `task cluster:create` fails with missing precondition errors.

**Error example**:
```
task: precondition not met: Managed resource group myprefix-managed-rg not found
```

**Solutions**:
```bash
# Missing managed resource group
task azure:infra

# Missing workload identities
task azure:identities

# Missing federated credentials
task azure:federated-creds

# Missing network IDs file
# This is created by task azure:infra
ls -l .azure-net-ids

# Missing credential files
ls -l ${AZURE_CREDS} ${PULL_SECRET}
```

#### Azure Quota or Capacity Issues

**Symptom**: Cluster creation fails with Azure quota exceeded errors.

**Error messages**:
```
Operation could not be completed as it results in exceeding approved Total Regional vCPUs quota
```

**Solutions**:
```bash
# Check current quota usage
az vm list-usage \
    --location ${LOCATION} \
    --output table

# Request quota increase through Azure portal:
# Portal → Subscriptions → Usage + quotas → Request increase

# Workaround: Use smaller VM size
# In cluster creation, reduce node count or VM size
NODE_POOL_REPLICAS=1  # Reduce from 2+
```

### Control Plane Issues

#### Control Plane Pods Not Starting

**Symptom**: Control plane pods stuck in `Pending`, `ImagePullBackOff`, or `CrashLoopBackOff`.

**Diagnostic steps**:
```bash
# Check pod status
kubectl get pods -n clusters-${CLUSTER_NAME}

# Describe problematic pod
kubectl describe pod <pod-name> -n clusters-${CLUSTER_NAME}

# Check pod logs
kubectl logs <pod-name> -n clusters-${CLUSTER_NAME}

# Check events
kubectl get events -n clusters-${CLUSTER_NAME} --sort-by='.lastTimestamp'
```

**Common causes and solutions**:

**Insufficient Resources on Management Cluster**:
```bash
# Check node resources
kubectl top nodes

# Check pod resource requests
kubectl describe pod <pod-name> -n clusters-${CLUSTER_NAME} | grep -A 5 Requests

# Each hosted cluster needs approximately:
# - 4 vCPU
# - 8 GB memory

# Solutions:
# - Add more nodes to management cluster
# - Delete unused hosted clusters
# - Reduce resource requests (not recommended for production)
```

**Image Pull Errors**:
```bash
# Check pull secret
kubectl get secret ${CLUSTER_NAME}-pull-secret \
    -n clusters-${CLUSTER_NAME} \
    -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq .

# Verify pull secret is valid at cloud.redhat.com
# Recreate cluster with valid pull secret if needed
```

**etcd Issues**:
```bash
# Check etcd pods
kubectl get pods -n clusters-${CLUSTER_NAME} -l app=etcd

# Check etcd logs
kubectl logs -n clusters-${CLUSTER_NAME} statefulset/etcd

# Check etcd persistent volumes
kubectl get pvc -n clusters-${CLUSTER_NAME}
```

#### Control Plane Performance Issues

**Symptom**: Slow API response times or timeouts.

**Diagnostic steps**:
```bash
# Check control plane pod CPU/memory usage
kubectl top pods -n clusters-${CLUSTER_NAME}

# Check API server logs for slow requests
kubectl logs -n clusters-${CLUSTER_NAME} deployment/kube-apiserver | grep -i "slow\|timeout"

# Check etcd performance
kubectl logs -n clusters-${CLUSTER_NAME} statefulset/etcd | grep -i "slow"
```

**Solutions**:
- Scale up management cluster nodes
- Reduce load on hosted cluster
- Check management cluster storage performance

### Worker Node Issues

#### Worker Nodes Not Joining Cluster

**Symptom**: Azure VMs are created but don't appear in `kubectl get nodes` (from hosted cluster context).

**Diagnostic steps**:
```bash
# 1. Check NodePool status
kubectl get nodepool -n ${CLUSTER_NAMESPACE} -o yaml

# 2. Check Machine resources
kubectl get machines -n clusters-${CLUSTER_NAME}

# 3. Check Azure VMs
az vm list \
    --resource-group ${MANAGED_RG_NAME} \
    --output table

# 4. Check ignition server logs
kubectl logs -n clusters-${CLUSTER_NAME} deployment/ignition-server

# 5. Get VM boot diagnostics (if enabled)
VM_NAME=$(az vm list --resource-group ${MANAGED_RG_NAME} --query "[0].name" -o tsv)
az vm boot-diagnostics get-boot-log \
    --resource-group ${MANAGED_RG_NAME} \
    --name ${VM_NAME}
```

**Common causes and solutions**:

**NSG Blocking Required Ports**:
```bash
# Check NSG rules
az network nsg show \
    --name ${NSG} \
    --resource-group ${NSG_RG_NAME}

# Required inbound ports:
# - 6443 (API server)
# - 22623 (machine config server / ignition)
# - 443 (ingress)
# - 10250-10259 (kubelet, kube-proxy)

# Fix: Update NSG rules
az network nsg rule create \
    --resource-group ${NSG_RG_NAME} \
    --nsg-name ${NSG} \
    --name allow-ignition \
    --priority 1000 \
    --source-address-prefixes '*' \
    --destination-port-ranges 22623 \
    --access Allow \
    --protocol Tcp
```

**Ignition Server Not Reachable**:
```bash
# Check ignition server service
kubectl get svc -n clusters-${CLUSTER_NAME} ignition-server

# From a node, test connectivity (requires VM access)
# curl -k https://ignition-server.clusters-${CLUSTER_NAME}.svc:443/config/worker
```

**Workload Identity Issues**:
```bash
# Check nodepool-mgmt identity has proper roles
az role assignment list \
    --assignee $(az identity show \
        --name "${CLUSTER_NAME}-nodepool-mgmt" \
        --resource-group ${PERSISTENT_RG_NAME} \
        --query clientId -o tsv) \
    --output table
```

#### Node Status Shows NotReady

**Symptom**: Nodes appear in `kubectl get nodes` but show status `NotReady`.

**Diagnostic steps**:
```bash
# From hosted cluster context
export KUBECONFIG=${CLUSTER_NAME}-kubeconfig

# Check node status
kubectl get nodes -o wide

# Describe the NotReady node
kubectl describe node <node-name>

# Check node conditions
kubectl get node <node-name> -o jsonpath='{.status.conditions}'  | jq .
```

**Common causes**:
- Container runtime issues
- Network plugin not ready
- Insufficient resources on node
- Disk pressure

**Solutions**:
```bash
# Check kubelet logs (requires SSH access to node)
# ssh -i ${CLUSTER_NAME}-ssh-key core@<node-ip>
# sudo journalctl -u kubelet -f

# Delete and recreate the node
kubectl delete node <node-name>
# NodePool controller will create replacement VM
```

### DNS and Networking Issues

#### DNS Records Not Created (External DNS)

**Symptom**: Cannot resolve cluster API or app hostnames via DNS.

**Diagnostic steps**:
```bash
# Check External DNS pod
kubectl get pods -n hypershift -l app=external-dns

# Check External DNS logs
kubectl logs -n hypershift deployment/external-dns

# Check Route resources
kubectl get routes -n clusters-${CLUSTER_NAME}

# Verify DNS zone exists
az network dns zone show \
    --name ${PARENT_DNS_ZONE} \
    --resource-group ${PERSISTENT_RG_NAME}

# Check DNS records
az network dns record-set list \
    --resource-group ${PERSISTENT_RG_NAME} \
    --zone-name ${PARENT_DNS_ZONE} \
    --output table
```

**Common issues**:

**External DNS Authentication Failures**:
```bash
# Check External DNS secret
kubectl get secret azure-config-file -n default

# Verify service principal permissions
az role assignment list \
    --assignee $(jq -r '.aadClientId' < azure_mgmt.json) \
    --output table

# Should have "DNS Zone Contributor" role on DNS zone resource group
```

**External DNS Not Watching Correct Namespace**:
```bash
# Check External DNS configuration
kubectl get deployment external-dns -n hypershift -o yaml | grep -A 10 args

# Ensure it's watching routes in hosted cluster namespaces
```

**Solution**: Recreate External DNS configuration
```bash
# Reinstall HyperShift operator with correct External DNS config
hypershift install \
    --external-dns-provider=azure \
    --external-dns-credentials azure_mgmt.json \
    --pull-secret ${PULL_SECRET} \
    --external-dns-domain-filter ${EXTRN_DNS_ZONE_NAME} \
    --limit-crd-install Azure \
    --render | kubectl apply -f -
```

#### LoadBalancer Service Stuck Pending (Without External DNS)

**Symptom**: API server LoadBalancer service doesn't get external IP.

**Diagnostic steps**:
```bash
# Check service
kubectl get svc -n clusters-${CLUSTER_NAME} kube-apiserver

# Check cloud-controller-manager logs
kubectl logs -n clusters-${CLUSTER_NAME} deployment/cloud-controller-manager

# Check Azure load balancer
az network lb list \
    --resource-group ${MANAGED_RG_NAME} \
    --output table
```

**Solutions**:
- Verify cloud-provider workload identity has correct permissions
- Check Azure subscription quotas for load balancers
- Verify network connectivity from management cluster to Azure APIs

### Upgrade Issues

#### Upgrade Stuck or Fails

**Symptom**: Cluster upgrade doesn't complete or control plane pods crash during upgrade.

**Diagnostic steps**:
```bash
# Check HostedCluster upgrade status
kubectl get hostedcluster ${CLUSTER_NAME} \
    -n ${CLUSTER_NAMESPACE} \
    -o jsonpath='{.status.version}'

# Check cluster version operator
kubectl logs -n clusters-${CLUSTER_NAME} deployment/cluster-version-operator

# Check control plane pod status
kubectl get pods -n clusters-${CLUSTER_NAME}

# From hosted cluster, check cluster operators
export KUBECONFIG=${CLUSTER_NAME}-kubeconfig
kubectl get co
```

**Solutions**:
```bash
# If upgrade is stuck, check for:
# - Degraded cluster operators before upgrade
# - Insufficient resources during upgrade
# - Breaking changes in release notes

# Cannot rollback - must fix forward or recreate cluster
```

## Diagnostic Commands Reference

### HostedCluster Health

```bash
# Overall cluster status
kubectl get hostedcluster ${CLUSTER_NAME} -n ${CLUSTER_NAMESPACE}

# Detailed status with conditions
kubectl get hostedcluster ${CLUSTER_NAME} -n ${CLUSTER_NAMESPACE} -o yaml

# Check HostedControlPlane
kubectl get hostedcontrolplane -n clusters-${CLUSTER_NAME}

# Describe for events
kubectl describe hostedcluster ${CLUSTER_NAME} -n ${CLUSTER_NAMESPACE}
```

### NodePool Health

```bash
# NodePool status
kubectl get nodepool -n ${CLUSTER_NAMESPACE}

# Detailed NodePool information
kubectl get nodepool ${CLUSTER_NAME} -n ${CLUSTER_NAMESPACE} -o yaml

# Check Machine resources
kubectl get machines -n clusters-${CLUSTER_NAME}

# Machine details
kubectl describe machine <machine-name> -n clusters-${CLUSTER_NAME}
```

### Control Plane Diagnostics

```bash
# All control plane pods
kubectl get pods -n clusters-${CLUSTER_NAME}

# Pod resource usage
kubectl top pods -n clusters-${CLUSTER_NAME}

# Specific component logs
kubectl logs -n clusters-${CLUSTER_NAME} deployment/kube-apiserver -f
kubectl logs -n clusters-${CLUSTER_NAME} deployment/kube-controller-manager -f
kubectl logs -n clusters-${CLUSTER_NAME} deployment/kube-scheduler -f
kubectl logs -n clusters-${CLUSTER_NAME} statefulset/etcd -f

# Check all resources in control plane namespace
kubectl get all -n clusters-${CLUSTER_NAME}

# Events in control plane namespace
kubectl get events -n clusters-${CLUSTER_NAME} --sort-by='.lastTimestamp'
```

### HyperShift Operator Diagnostics

```bash
# Operator pod status
kubectl get pods -n hypershift

# Operator logs
kubectl logs -n hypershift deployment/operator -f

# Check operator reconciliation
kubectl logs -n hypershift deployment/operator | grep ${CLUSTER_NAME}
```

### Azure Resource Diagnostics

```bash
# List all resource groups for cluster
az group list \
    --query "[?contains(name, '${PREFIX}')].{Name:name, Location:location, State:properties.provisioningState}" \
    --output table

# Check VMs
az vm list \
    --resource-group ${MANAGED_RG_NAME} \
    --output table

# Check VM power state
az vm list \
    --resource-group ${MANAGED_RG_NAME} \
    --show-details \
    --query "[].{Name:name, PowerState:powerState}" \
    --output table

# Check load balancers
az network lb list \
    --resource-group ${MANAGED_RG_NAME} \
    --output table

# Check NSG rules
az network nsg rule list \
    --nsg-name ${NSG} \
    --resource-group ${NSG_RG_NAME} \
    --output table
```

### Workload Identity Diagnostics

```bash
# List managed identities
task azure:list-identities

# Or manually:
az identity list \
    --resource-group ${PERSISTENT_RG_NAME} \
    --query "[?contains(name, '${CLUSTER_NAME}')]" \
    --output table

# List federated credentials
task azure:list-federated-creds

# Check specific component identity
az identity show \
    --name "${CLUSTER_NAME}-disk-csi" \
    --resource-group ${PERSISTENT_RG_NAME}

# Check role assignments
az role assignment list \
    --assignee $(az identity show \
        --name "${CLUSTER_NAME}-disk-csi" \
        --resource-group ${PERSISTENT_RG_NAME} \
        --query principalId -o tsv) \
    --output table
```

## Collecting Debug Information

### Comprehensive Cluster Dump

```bash
#!/bin/bash
# Save to a file: cluster-debug.sh

CLUSTER_NAME="myprefix-hc"
CLUSTER_NAMESPACE="clusters"
OUTPUT_DIR="debug-$(date +%Y%m%d-%H%M%S)"

mkdir -p ${OUTPUT_DIR}

echo "Collecting debug information for ${CLUSTER_NAME}..."

# HostedCluster
kubectl get hostedcluster ${CLUSTER_NAME} -n ${CLUSTER_NAMESPACE} -o yaml > ${OUTPUT_DIR}/hostedcluster.yaml

# NodePools
kubectl get nodepool -n ${CLUSTER_NAMESPACE} -o yaml > ${OUTPUT_DIR}/nodepools.yaml

# Control plane pods
kubectl get pods -n clusters-${CLUSTER_NAME} -o wide > ${OUTPUT_DIR}/control-plane-pods.txt
kubectl get pods -n clusters-${CLUSTER_NAME} -o yaml > ${OUTPUT_DIR}/control-plane-pods.yaml

# Control plane events
kubectl get events -n clusters-${CLUSTER_NAME} --sort-by='.lastTimestamp' > ${OUTPUT_DIR}/control-plane-events.txt

# Operator logs
kubectl logs -n hypershift deployment/operator --tail=1000 > ${OUTPUT_DIR}/operator-logs.txt

# Control plane logs
for deployment in kube-apiserver kube-controller-manager kube-scheduler; do
    kubectl logs -n clusters-${CLUSTER_NAME} deployment/${deployment} --tail=500 > ${OUTPUT_DIR}/${deployment}-logs.txt
done

# Machines
kubectl get machines -n clusters-${CLUSTER_NAME} -o yaml > ${OUTPUT_DIR}/machines.yaml

echo "Debug information collected in ${OUTPUT_DIR}/"
tar -czf ${OUTPUT_DIR}.tar.gz ${OUTPUT_DIR}/
echo "Archive created: ${OUTPUT_DIR}.tar.gz"
```

## Getting Help

### Before Requesting Support

Collect the following information:

1. **Cluster details**:
   - OpenShift version
   - HyperShift operator version
   - Azure region

2. **Error symptoms**:
   - Exact error messages
   - When the issue started
   - Steps to reproduce

3. **Debug information** (use script above):
   - HostedCluster YAML
   - Control plane pod status and logs
   - NodePool status
   - Operator logs

### Support Channels

**For Developer Preview**:
- [HyperShift GitHub Issues](https://github.com/openshift/hypershift/issues)
- [HyperShift Slack](https://kubernetes.slack.com/archives/C02C4LXBW0T) (#hypershift channel)

**For Production (when GA)**:
- Red Hat Support Portal
- OpenShift support cases

### Useful Resources

- [HyperShift Documentation](https://hypershift.pages.dev)
- [Azure Workload Identity Troubleshooting](https://azure.github.io/azure-workload-identity/docs/troubleshooting.html)
- [OpenShift Documentation](https://docs.openshift.com)
- [Azure CLI Documentation](https://learn.microsoft.com/en-us/cli/azure/)

## Next Steps

- [Reference](08-reference.md) - CLI commands and configuration reference
- [Understanding HyperShift](01-understanding.md) - Review architecture concepts
