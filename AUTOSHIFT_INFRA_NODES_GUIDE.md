# AutoShift Infrastructure Nodes Policy Guide

Based on analysis of the autoshiftv2 repository, this document explains how the infrastructure node policies work and how they can be used to move OpenShift platform components to/from infrastructure nodes.

## Overview

The **infra-nodes** policy in AutoShift provides automated management of infrastructure nodes and **automatic migration of OpenShift platform components** to those nodes. This is a key subscription optimization strategy since infrastructure nodes **do not consume OpenShift worker node subscriptions**.

## What Are Infrastructure Nodes?

Infrastructure nodes are dedicated OpenShift nodes that run platform services, separating them from user application workloads. This provides:
- **Cost savings**: Infra nodes don't count against worker node subscriptions
- **Performance isolation**: Platform services don't compete with applications for resources
- **Operational clarity**: Clear separation between platform and application workloads

## AutoShift Infra Node Architecture

### Policy Location
```
autoshiftv2/policies/stable/infra-nodes/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── policy-infra-nodes-aws.yaml           # Create AWS infra MachineSets
│   ├── policy-infra-nodes-vmware.yaml        # Create VMware infra MachineSets
│   ├── policy-infra-nodes-test.yaml          # Test/validation
│   ├── policy-infra-machineconfigpool.yaml   # Create infra MachineConfigPool
│   ├── policy-infra-node-config.yaml         # Kubelet tuning for infra nodes
│   │
│   ├── policy-infra-monitoring.yaml          # ← MIGRATE monitoring to infra
│   ├── policy-infra-gitops.yaml              # ← MIGRATE gitops to infra
│   ├── policy-infra-ingress.yaml             # ← MIGRATE ingress to infra
│   ├── policy-infra-image-registry.yaml      # ← MIGRATE registry to infra
│   │
│   ├── policy-remove-infra-monitoring.yaml   # ← REMOVE from infra (back to workers)
│   ├── policy-remove-infra-gitops.yaml       # ← REMOVE from infra
│   ├── policy-remove-infra-ingress.yaml      # ← REMOVE from infra
│   └── policy-remove-infra-image-registry.yaml  # ← REMOVE from infra
```

## Components That Can Be Migrated

AutoShift provides built-in policies to migrate these OpenShift platform components to infrastructure nodes:

### 1. Monitoring Stack (`policy-infra-monitoring.yaml`)

**Components Migrated:**
- Prometheus (prometheusK8s)
- Alertmanager (alertmanagerMain)
- Prometheus Operator
- Kubernetes Metrics Adapter (k8sPrometheusAdapter)
- Kube State Metrics
- Telemetry Client
- OpenShift State Metrics
- Thanos Querier
- Monitoring Console Plugin

**How It Works:**
- Creates/updates the `cluster-monitoring-config` ConfigMap in `openshift-monitoring` namespace
- Adds `nodeSelector` and `tolerations` to each component
- All monitoring pods automatically reschedule to infra nodes

**Configuration Applied:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |+
    prometheusK8s:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
      tolerations:
      - key: node-role.kubernetes.io/infra
        effect: NoSchedule
      - key: node-role.kubernetes.io/infra
        effect: NoExecute
    # ... (repeated for all monitoring components)
```

### 2. OpenShift GitOps (`policy-infra-gitops.yaml`)

**Components Migrated:**
- ArgoCD application controller
- ArgoCD server
- ArgoCD repo server
- ArgoCD Redis
- All OpenShift GitOps workloads

**How It Works:**
- Patches the `GitopsService` CR (default name: `cluster`)
- Sets `spec.runOnInfra: true`
- Adds tolerations for infra node taints

**Configuration Applied:**
```yaml
apiVersion: pipelines.openshift.io/v1alpha1
kind: GitopsService
metadata:
  name: cluster
  namespace: openshift-gitops
spec:
  runOnInfra: true
  tolerations:
    - effect: NoSchedule
      key: node-role.kubernetes.io/infra
    - effect: NoExecute
      key: node-role.kubernetes.io/infra
```

### 3. Ingress Controllers (`policy-infra-ingress.yaml`)

**Components Migrated:**
- Default ingress controller (router pods)
- All traffic-facing router replicas

**How It Works:**
- Patches the default `IngressController` CR
- Adds `nodePlacement` configuration with nodeSelector and tolerations

**Configuration Applied:**
```yaml
apiVersion: operator.openshift.io/v1
kind: IngressController
metadata:
  name: default
  namespace: openshift-ingress-operator
spec:
  nodePlacement:
    nodeSelector:
      matchLabels:
        node-role.kubernetes.io/infra: ''
    tolerations:
      - effect: NoSchedule
        key: node-role.kubernetes.io/infra
      - effect: NoExecute
        key: node-role.kubernetes.io/infra
```

### 4. Image Registry (`policy-infra-image-registry.yaml`)

**Components Migrated:**
- Internal OpenShift image registry pods

**How It Works:**
- Patches the cluster `Config` CR for image registry
- Adds `nodeSelector` and `tolerations`

**Configuration Applied:**
```yaml
apiVersion: imageregistry.operator.openshift.io/v1
kind: Config
metadata:
  name: cluster
spec:
  nodeSelector: 
    node-role.kubernetes.io/infra: ""
  tolerations:
  - effect: NoSchedule
    key: node-role.kubernetes.io/infra
  - effect: NoExecute
    key: node-role.kubernetes.io/infra
```

## Controlling Component Migration via Labels

### Enable/Disable Migration Per Component

Set these labels at the **cluster** or **clusterset** level in ACM:

```yaml
# Enable/disable individual component migration
autoshift.io/infra-nodes: "3"  # Must be set for any infra policies to apply

# Control which components migrate (true/false)
# These are controlled in values.yaml:
infraNodes:
  imageRegistry: 
    migrate: true          # ← Controls policy-infra-image-registry
  ingress: 
    migrate: true          # ← Controls policy-infra-ingress
  monitoring: 
    migrate: true          # ← Controls policy-infra-monitoring
  gitops: 
    migrate: true          # ← Controls policy-infra-gitops
```

### Values.yaml Configuration

From `autoshiftv2/policies/stable/infra-nodes/values.yaml`:

```yaml
infraNodes:
  labelPrefix: "autoshift.io/"
  remediationAction: "enforce"
  
  # Image Registry Migration
  imageRegistry: 
    migrate: true                    # Enable migration
    remediationAction: enforce
  
  # Ingress Migration
  ingress: 
    migrate: true
    remediationAction: enforce
  
  # Monitoring Stack Migration
  monitoring: 
    migrate: true
    remediationAction: enforce
  
  # GitOps Migration
  gitops: 
    migrate: true
    remediationAction: enforce
    namespace: openshift-gitops       # Target namespace
```

## How to Move Components TO Infrastructure Nodes

### Step 1: Provision Infrastructure Nodes

Set the cluster label to create infrastructure nodes:

```yaml
# On your managed cluster in ACM
autoshift.io/infra-nodes: "3"
autoshift.io/infra-nodes-provider: "aws"  # or 'vmware' or 'test'
autoshift.io/infra-nodes-instance-type: "m5.xlarge"  # AWS
autoshift.io/infra-nodes-zone-1: "us-east-2a"
autoshift.io/infra-nodes-zone-2: "us-east-2b"
autoshift.io/infra-nodes-zone-3: "us-east-2c"
```

This triggers the `policy-infra-nodes-aws` (or vmware) policy which creates:
- 3 infrastructure MachineSets
- Nodes labeled with `node-role.kubernetes.io/infra`
- Nodes tainted with `node-role.kubernetes.io/infra:NoSchedule` and `NoExecute`

### Step 2: Enable Component Migration

Once infra nodes exist and are Ready, the migration policies automatically activate because:

**Placement Rule** (from each migration policy):
```yaml
spec:
  predicates:
    - requiredClusterSelector:
        labelSelector:
          matchExpressions:
            - key: autoshift.io/infra-nodes
              operator: Exists           # ← Cluster has infra-nodes label
            - key: autoshift.io/infra-nodes
              operator: NotIn
              values:
                - "0"                    # ← And it's not set to 0
```

The policies apply automatically when:
- `autoshift.io/infra-nodes` label exists on the cluster
- The value is NOT "0"
- The specific component migration is enabled in `values.yaml` (`migrate: true`)

### Step 3: Verify Migration

```bash
# Check that infra nodes exist
oc get nodes -l node-role.kubernetes.io/infra

# Verify monitoring pods moved to infra nodes
oc get pods -n openshift-monitoring -o wide

# Verify GitOps pods moved to infra nodes
oc get pods -n openshift-gitops -o wide

# Verify ingress router pods moved to infra nodes
oc get pods -n openshift-ingress -o wide

# Verify image registry pods moved to infra nodes
oc get pods -n openshift-image-registry -o wide
```

## How to Move Components BACK to Worker Nodes

### Automatic Removal When Infra Nodes Deleted

Set the infra-nodes label to **"0"**:

```yaml
autoshift.io/infra-nodes: "0"  # ← Triggers REMOVE policies
```

This activates the "remove" policies which:
- Clear nodeSelectors (set to `{}`)
- Clear tolerations (set to `[]`)
- Allow pods to reschedule to worker nodes

**Placement Rule** (from remove policies):
```yaml
spec:
  predicates:
    - requiredClusterSelector:
        labelSelector:
          matchExpressions:
            - key: autoshift.io/infra-nodes
              operator: In
              values:
                - "0"                    # ← Triggers when set to 0
```

### Example: Remove Monitoring from Infra Nodes

The `policy-remove-infra-monitoring` applies:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |+
    prometheusK8s:
      nodeSelector: {}        # ← Cleared
      tolerations: []         # ← Cleared
    alertmanagerMain:
      nodeSelector: {}
      tolerations: []
    # ... all components get cleared
```

Result: Monitoring pods reschedule to worker nodes.

## Policy Dependencies

The migration policies depend on infra nodes being provisioned first:

```yaml
dependencies:
  install:
    - "policy-infra-nodes-test"  # Ensures infra nodes exist before migrating
  
  remove:
    - policy-remove-infra-gitops
    - policy-remove-infra-image-registry
    - policy-remove-infra-ingress
    - policy-remove-infra-monitoring
```

## Advanced: Infrastructure MachineConfigPool

Enable a dedicated MachineConfigPool for infra nodes:

```yaml
autoshift.io/infra-machineconfigpool: "true"
```

**Benefits:**
- Separate infra nodes from worker MachineConfigPool
- Apply infra-specific MachineConfigs without affecting workers
- Required for infra-specific kubelet tuning

## Advanced: Infra Node Kubelet Tuning

Enable specialized kubelet configuration for infra nodes:

```yaml
autoshift.io/infra-node-config: "true"
autoshift.io/infra-machineconfigpool: "true"  # Required first

# Optional tuning parameters:
autoshift.io/infra-max-pods: "250"
autoshift.io/infra-system-reserved-memory: "2Gi"
autoshift.io/infra-system-reserved-cpu: "500m"
autoshift.io/infra-kube-reserved-memory: "1Gi"
autoshift.io/infra-kube-reserved-cpu: "500m"
autoshift.io/infra-eviction-memory: "500Mi"
```

**Purpose:**
- Reserve more resources for platform services (vs worker nodes)
- Protect monitoring, logging, and router from eviction
- Optimize for platform workload characteristics

From `values.yaml` defaults:
```yaml
nodeConfig:
  maxPods: 250
  systemReservedMemory: 2Gi      # Higher than workers
  systemReservedCpu: 500m
  kubeReservedMemory: 1Gi        # Higher than workers
  kubeReservedCpu: 500m
  evictionMemory: 500Mi
```

## Complete Example: Production Cluster

### Cluster Labels (ACM)

```yaml
# Provision 3 infra nodes on AWS
autoshift.io/infra-nodes: "3"
autoshift.io/infra-nodes-provider: "aws"
autoshift.io/infra-nodes-instance-type: "m5.2xlarge"
autoshift.io/infra-nodes-volume-size: "120"
autoshift.io/infra-nodes-zone-1: "us-east-2a"
autoshift.io/infra-nodes-zone-2: "us-east-2b"
autoshift.io/infra-nodes-zone-3: "us-east-2c"

# Create dedicated infra MachineConfigPool
autoshift.io/infra-machineconfigpool: "true"

# Enable infra node kubelet tuning
autoshift.io/infra-node-config: "true"
autoshift.io/infra-max-pods: "250"
autoshift.io/infra-system-reserved-memory: "4Gi"  # Increase for heavy monitoring
autoshift.io/infra-system-reserved-cpu: "1000m"
```

### Result

**Infrastructure Created:**
- 3 AWS EC2 instances (m5.2xlarge) across 3 AZs
- Nodes labeled: `node-role.kubernetes.io/infra`
- Nodes tainted: `node-role.kubernetes.io/infra:NoSchedule` + `NoExecute`
- Dedicated infra MachineConfigPool
- Kubelet tuned with 4Gi system reserved memory

**Components Migrated to Infra Nodes:**
- ✅ Monitoring stack (Prometheus, Alertmanager, etc.)
- ✅ OpenShift GitOps (ArgoCD)
- ✅ Ingress controllers (routers)
- ✅ Image registry

**Subscription Benefit:**
- 3 infra nodes **do NOT count** against OpenShift worker subscriptions
- Only worker nodes count toward subscription limits

## How to Add Additional Components

The engineering team mentioned: *"Currently in the infra node policy section there are ways to move some components to and from infra nodes"*

To add **new** components beyond the built-in four, you would:

### Option 1: Extend Existing Policies

Add to an existing policy (e.g., `policy-infra-monitoring.yaml`):

```yaml
# Add new component to monitoring ConfigMap
data:
  config.yaml: |+
    # Existing components...
    
    # New component
    myNewComponent:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
      tolerations:
      - key: node-role.kubernetes.io/infra
        effect: NoSchedule
      - key: node-role.kubernetes.io/infra
        effect: NoExecute
```

### Option 2: Create New Policy

Create a new policy file following the pattern:

```yaml
# policy-infra-[component].yaml
apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  name: policy-infra-[component]
spec:
  disabled: {{ not .Values.infraNodes.[component].migrate }}
  dependencies:
  - apiVersion: policy.open-cluster-management.io/v1
    compliance: Compliant
    kind: Policy
    name: policy-infra-nodes-test  # Wait for infra nodes to exist
  policy-templates:
    - objectDefinition:
        apiVersion: policy.open-cluster-management.io/v1
        kind: ConfigurationPolicy
        spec:
          object-templates:
            - complianceType: musthave
              objectDefinition:
                # Your component CR with nodeSelector + tolerations
```

### Option 3: Supplemental Policies (Recommended for Custom Components)

For components like ACS (Advanced Cluster Security), create supplemental policies outside the core infra-nodes chart, similar to what's in the 71638776-autoshift repo:

```yaml
# policies/policy-acs-infra-nodes.yaml
apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  name: policy-acs-infra-nodes
spec:
  policy-templates:
    - objectDefinition:
        kind: ConfigurationPolicy
        spec:
          object-templates:
            - complianceType: musthave
              objectDefinition:
                apiVersion: platform.stackrox.io/v1alpha1
                kind: Central
                metadata:
                  name: stackrox-central-services
                  namespace: stackrox
                spec:
                  customize:
                    deploymentDefaults:
                      pinToNodes: InfraRole  # ← ACS-specific infra config
```

## Summary

The AutoShift infra-nodes policy provides:

✅ **Automated Infrastructure Node Provisioning**
- AWS, VMware, or test providers
- Configurable sizing and zones

✅ **Automatic Component Migration** (4 built-in components)
- Monitoring stack (9 components)
- OpenShift GitOps
- Ingress controllers
- Image registry

✅ **Bidirectional Movement**
- Move TO infra: Set `infra-nodes: "3"`
- Move FROM infra: Set `infra-nodes: "0"`
- Controlled per-component via `values.yaml`

✅ **Subscription Optimization**
- Infra nodes don't consume worker subscriptions
- Significant cost savings for large fleets

✅ **Extensible Architecture**
- Add custom component migrations
- Follow existing policy patterns
- Create supplemental policies for application-specific needs

## References

- **AutoShift v2 Repository**: `/home/sscaling/Repos/autoshiftv2`
- **Infra Nodes Policy**: `policies/stable/infra-nodes/`
- **Values Reference**: `docs/values-reference.md`
