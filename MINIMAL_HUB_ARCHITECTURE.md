# Minimal ACM Hub Architecture: 3 Control Plane + 3 Infrastructure Nodes

## Overview

This repository is configured to deploy a **minimal ACM Hub cluster** with only:
- **3 control plane nodes** (masters)
- **3 infrastructure nodes** (infra)
- **0 worker nodes**

All platform workloads (ACM, GitOps, monitoring, ingress, registry, ACS, Quay) run on the dedicated infrastructure nodes.

## Architecture Rationale

### Why No Worker Nodes?

For a dedicated ACM Hub cluster that **only manages other clusters** (not running user applications):

✅ **Cost Savings**: Eliminates 3 worker nodes (~33% reduction)  
✅ **Clear Separation**: Platform workloads isolated on infra nodes  
✅ **Simplified Management**: Fewer nodes to patch, monitor, and maintain  
✅ **Subscription Optimization**: Infra nodes don't consume worker subscriptions  

**User applications should be deployed on managed clusters, not the hub.**

### Node Breakdown

| Node Type | Count | Purpose | Taints | Labels |
|-----------|-------|---------|--------|--------|
| **Control Plane** | 3 | Kubernetes control plane (API, scheduler, controllers) | `node-role.kubernetes.io/master:NoSchedule` | `node-role.kubernetes.io/master` |
| **Infrastructure** | 3 | Platform services (ACM, GitOps, monitoring, ingress, registry) | `node-role.kubernetes.io/infra:NoSchedule` | `node-role.kubernetes.io/infra` |
| **Workers** | 0 | N/A | N/A | N/A |

## Implementation

### 1. Install-Config: Zero Workers

**File:** `roles/ocp/aws_provision_ocp_ipi/templates/install-config.yaml.j2`

```yaml
compute:
- name: worker
  replicas: 0          # ← No workers provisioned by IPI
  platform: {}
```

The OpenShift IPI installer creates **only 3 control plane nodes**. Infrastructure nodes are added post-install by AutoShift policies.

### 2. AutoShift Cluster Labels

**File:** `playbooks/deploy-autoshift-stack.yaml`

```yaml
hubClusterSets:
  hub:
    labels:
      # Infrastructure Nodes Configuration
      infra-nodes: '3'                                    # Number of infra nodes
      infra-nodes-provider: 'aws'                         # Provider (aws, vmware, test)
      infra-nodes-instance-type: "{{ install_config.infra.instance_type }}"
      infra-nodes-volume-size: '120'                      # Root volume size (GB)
      infra-machineconfigpool: 'true'                     # Create dedicated MCP
      infra-node-config: 'true'                           # Enable kubelet tuning
```

These labels trigger AutoShift's `infra-nodes` policy to:
- Create 3 AWS MachineSets for infrastructure nodes
- Label nodes with `node-role.kubernetes.io/infra`
- Taint nodes with `NoSchedule` to prevent user workloads
- Create a dedicated MachineConfigPool for infra-specific configuration

### 3. Platform Workload Migration (Automatic)

AutoShift automatically migrates these platform components to infra nodes:

| Component | AutoShift Policy | Migration Method |
|-----------|------------------|------------------|
| **Ingress Controller** | `policy-infra-ingress` | Patches `IngressController/default` with nodeSelector + tolerations |
| **Image Registry** | `policy-infra-image-registry` | Patches `imageregistry.operator.openshift.io/cluster` |
| **Monitoring Stack** | `policy-infra-monitoring` | Creates `cluster-monitoring-config` ConfigMap |
| **OpenShift GitOps** | `policy-infra-gitops` | Patches `GitopsService/cluster` with `runOnInfra: true` |

**Reference:** See `AUTOSHIFT_INFRA_NODES_GUIDE.md` for detailed policy documentation.

### 4. Supplemental Policies (This Repository)

Since AutoShift's upstream policies don't expose all configuration options, we provide supplemental policies for:

#### ACS Central (`policies/policy-acs-infra-nodes.yaml`)

Patches the ACS Central CR to pin deployments to infra nodes:

```yaml
apiVersion: platform.stackrox.io/v1alpha1
kind: Central
spec:
  customize:
    deploymentDefaults:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
      tolerations:
        - effect: NoSchedule
          key: node-role.kubernetes.io/infra
```

#### ACM MultiClusterHub (`policies/policy-acm-infra-nodes.yaml`)

Patches the MCH CR to run all ACM components on infra nodes:

```yaml
apiVersion: operator.open-cluster-management.io/v1
kind: MultiClusterHub
spec:
  nodeSelector:
    node-role.kubernetes.io/infra: ""
  tolerations:
    - effect: NoSchedule
      key: node-role.kubernetes.io/infra
```

**Why supplemental policies?**
- Applied in **sync-wave "2"** (after AutoShift's wave "1")
- Depend on base operator installation being compliant
- Use `complianceType: musthave` for additive merging (no conflicts)

## Deployment

### Prerequisites

```bash
ansible-playbook playbooks/ocp/install-ocp-prerequisites.yaml --ask-become-pass
```

### Full Stack Deployment

```bash
ansible-playbook playbooks/deploy-autoshift-stack.yaml -i <inventory>
```

**Phases:**
1. **Provision OCP Cluster** (IPI): 3 control plane nodes, 0 workers (~30-40 min)
2. **Install OpenShift GitOps** (Helm OCI)
3. **Install ACM** (Helm OCI, ~10-15 min)
4. **Deploy AutoShift Application** (ArgoCD)
   - AutoShift reads cluster labels
   - Deploys infra-nodes policies
   - Creates infrastructure MachineSets (~5-10 min)
   - Migrates platform workloads (~2-5 min)
5. **Apply Supplemental Policies** (ACS, ACM)

**Total time:** ~45-60 minutes

## Verification

### 1. Node Count

```bash
oc get nodes
```

**Expected:**
```
NAME                         STATUS   ROLES                  AGE   VERSION
ip-10-0-x-x.ec2.internal     Ready    control-plane,master   45m   v1.29.x
ip-10-0-x-x.ec2.internal     Ready    control-plane,master   45m   v1.29.x
ip-10-0-x-x.ec2.internal     Ready    control-plane,master   45m   v1.29.x
ip-10-0-x-x.ec2.internal     Ready    infra                  20m   v1.29.x
ip-10-0-x-x.ec2.internal     Ready    infra                  20m   v1.29.x
ip-10-0-x-x.ec2.internal     Ready    infra                  20m   v1.29.x
```

### 2. Verify MachineSets

```bash
oc get machinesets -n openshift-machine-api
```

**Expected:**
```
NAME                            DESIRED   CURRENT   READY   AVAILABLE
cluster-name-infra-us-east-2a   1         1         1       1
cluster-name-infra-us-east-2b   1         1         1       1
cluster-name-infra-us-east-2c   1         1         1       1
```

### 3. Verify Platform Workloads on Infra Nodes

```bash
# Ingress router
oc get pods -n openshift-ingress -o wide | grep router

# Image registry
oc get pods -n openshift-image-registry -o wide

# Monitoring
oc get pods -n openshift-monitoring -o wide | grep -E "prometheus|alertmanager"

# OpenShift GitOps
oc get pods -n openshift-gitops -o wide

# ACM
oc get pods -n open-cluster-management -o wide

# ACS
oc get pods -n stackrox -o wide
```

All pods should show `NODE` columns with `infra` in the hostname.

### 4. Verify ACM Policies

```bash
oc get policies -n policies-autoshift | grep infra
```

**Expected:**
```
policy-infra-nodes-aws          enforce    Compliant
policy-infra-machineconfigpool  enforce    Compliant
policy-infra-node-config        enforce    Compliant
policy-infra-ingress            enforce    Compliant
policy-infra-image-registry     enforce    Compliant
policy-infra-monitoring         enforce    Compliant
policy-infra-gitops             enforce    Compliant
policy-acs-infra-nodes          enforce    Compliant
policy-acm-infra-nodes          enforce    Compliant
```

## Cost Analysis

### AWS Cost Comparison (m5.2xlarge, us-east-2, 24/7)

| Architecture | Node Count | Monthly Cost | Savings |
|--------------|------------|--------------|---------|
| **Standard Hub** (3 CP + 3 Worker + 3 Infra) | 9 nodes | ~$1,944/month | — |
| **Minimal Hub** (3 CP + 3 Infra) | 6 nodes | ~$1,296/month | **-$648/month (-33%)** |

*Assumes m5.2xlarge instances ($0.384/hour). Control plane nodes are required by OpenShift.*

**Annual savings:** ~$7,776 per hub cluster

## Troubleshooting

### Infra Nodes Not Provisioning

**Check policy status:**
```bash
oc get policy policy-infra-nodes-aws -n policies-autoshift -o yaml
```

**Common issues:**
- Cluster labels missing: `autoshift.io/infra-nodes: '3'`
- AWS IAM permissions insufficient
- AWS quota limits reached

**Resolution:**
```bash
# Check cluster labels
oc get managedcluster local-cluster -o jsonpath='{.metadata.labels}' | jq

# Check AWS quota
aws service-quotas get-service-quota --service-code ec2 --quota-code L-1216C47A
```

### Workloads Still on Control Plane

**Symptoms:** Platform pods running on master nodes instead of infra nodes

**Causes:**
- Infra nodes not Ready yet
- Policies not compliant
- Timing issue: policies applied before infra nodes existed

**Resolution:**
```bash
# Force policy re-evaluation
oc annotate policy policy-infra-ingress -n policies-autoshift \
  policy.open-cluster-management.io/trigger-update=$(date +%s) --overwrite

# Manually restart pods (after infra nodes are Ready)
oc delete pod -n openshift-ingress -l ingresscontroller.operator.openshift.io/deployment-ingresscontroller=default
```

### ACM/ACS Not on Infra Nodes

**Verify supplemental policies:**
```bash
oc get policy policy-acm-infra-nodes -n policies-autoshift -o yaml
oc get policy policy-acs-infra-nodes -n policies-autoshift -o yaml
```

**Check if CRs were patched:**
```bash
# ACM
oc get mch multiclusterhub -n open-cluster-management -o yaml | grep -A5 nodeSelector

# ACS
oc get central stackrox-central-services -n stackrox -o yaml | grep -A10 nodeSelector
```

## When to Use This Architecture

✅ **Recommended for:**
- Dedicated ACM Hub clusters (cluster management only)
- GitOps/Policy distribution hubs
- Development/sandbox environments
- Cost-sensitive deployments

❌ **Not suitable for:**
- Hubs running user applications
- Multi-tenant hubs with unpredictable workloads
- Environments requiring strict platform/application isolation

## Advanced Configuration

### Customizing Infra Node Size

Update inventory variable:

```yaml
install_config:
  infra:
    replicas: 3
    instance_type: "m5.4xlarge"  # Larger for heavy monitoring
```

Update cluster label:

```yaml
hubClusterSets:
  hub:
    labels:
      infra-nodes-instance-type: "m5.4xlarge"
```

### Adjusting Kubelet Reservations

```yaml
hubClusterSets:
  hub:
    labels:
      infra-system-reserved-memory: '4Gi'   # Default: 2Gi
      infra-kube-reserved-memory: '2Gi'     # Default: 1Gi
      infra-max-pods: '300'                 # Default: 250
```

See `AUTOSHIFT_INFRA_NODES_GUIDE.md` for all available tuning parameters.

### Adding Storage Nodes (Optional)

If you need ODF for ACM Observability:

```yaml
- name: Create ODF Storage Node Pool
  ansible.builtin.include_role:
    name: ocp/aws_create_machineset
  vars:
    pool_replicas: 3
    pool_instance_type: "m5.4xlarge"
    pool_labels:
      cluster.ocs.openshift.io/openshift-storage: ""
```

Then enable AutoShift's ODF policy:

```yaml
hubClusterSets:
  hub:
    labels:
      odf: 'true'
      odf-channel: stable-4.20
```

## References

- **AutoShift Documentation:** `AUTOSHIFT_INFRA_NODES_GUIDE.md` (in this repo)
- **AutoShift Repository:** `/home/sscaling/Repos/autoshiftv2`
- **Red Hat Docs:** [Managing Infrastructure MachineSets](https://docs.openshift.com/container-platform/latest/machine_management/creating-infrastructure-machinesets.html)
- **ACM Documentation:** [Red Hat Advanced Cluster Management](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/)
