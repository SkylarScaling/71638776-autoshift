# AutoShift Bugs and Limitations

This document tracks bugs and limitations discovered in the AutoShift v2 project during implementation of the minimal hub architecture (3 control plane + 3 infra, 0 workers).

**AutoShift Repository:** https://github.com/auto-shift/autoshiftv2  
**AutoShift Version Tested:** Policies from main branch (as of 2026-06-10)

---

## 🐛 Bug #1: Zone Label Template Parsing Error

**Status:** 🔴 Blocking  
**Component:** `policies/stable/infra-nodes/templates/policy-infra-nodes-aws.yaml`  
**Severity:** High

### Description

When multiple zone labels are defined (`infra-nodes-zone-1`, `infra-nodes-zone-2`, `infra-nodes-zone-3`), the ACM policy template generates invalid Go template syntax that fails to parse.

### Root Cause

The template loop that reads zone labels from cluster labels generates a Go template `list` call without spaces between quoted strings:

```go
{{- $zones_from_labels := list "us-east-2a""us-east-2b""us-east-2c" }}
```

This should be:
```go
{{- $zones_from_labels := list "us-east-2a" "us-east-2b" "us-east-2c" }}
```

### Location

File: `policies/stable/infra-nodes/templates/policy-infra-nodes-aws.yaml`  
Line: ~30

```yaml
{{ "{{-" }} $zones_from_labels := list {{ "{{hub" }} range $label, $labelvalue := .ManagedClusterLabels {{ "hub}}" }}{{ "{{hub" }} if $label | hasPrefix "autoshift.io/infra-nodes-zone" {{ "hub}}" }}"{{ "{{hub" }} $labelvalue {{ "hub}}" }}"{{ "{{hub" }} end {{ "hub}}" }}{{ "{{hub" }} end {{ "hub}}" }} {{ "}}" }}
```

The issue is that there's no space separator between the zone values being concatenated.

### Error Message

```
failed to parse the template JSON string: template: tmpl:3: unexpected "\"us-east-2"... in operand
```

### Reproduction

1. Set cluster labels:
   ```yaml
   autoshift.io/infra-nodes: '3'
   autoshift.io/infra-nodes-provider: 'aws'
   autoshift.io/infra-nodes-zone-1: 'us-east-2a'
   autoshift.io/infra-nodes-zone-2: 'us-east-2b'
   autoshift.io/infra-nodes-zone-3: 'us-east-2c'
   ```

2. Deploy AutoShift with infra-nodes policy enabled

3. Check policy status:
   ```bash
   oc get policy policy-infra-nodes-aws -n policies-autoshift
   ```

4. Policy will be `NonCompliant` with template parsing error

### Workaround

**Option A:** Omit zone labels entirely (uses first worker MachineSet's zone by default)

```yaml
# Remove these labels:
# infra-nodes-zone-1: 'us-east-2a'
# infra-nodes-zone-2: 'us-east-2b'
# infra-nodes-zone-3: 'us-east-2c'
```

Result: AutoShift creates 1 MachineSet with all replicas in a single AZ (no HA across zones).

**Option B:** Manually create MachineSets in additional zones after AutoShift creates the first one.

### Proposed Fix

Update the template to add a space separator between zone values:

```yaml
{{ "{{-" }} $zones_from_labels := list {{ "{{hub" }} range $i, $label := (keys .ManagedClusterLabels | sortAlpha) {{ "hub}}" }}{{ "{{hub" }} if $label | hasPrefix "autoshift.io/infra-nodes-zone" {{ "hub}}" }}{{ "{{hub" }} if $i {{ "hub}}" }} {{ "{{hub" }} end {{ "hub}}" }}"{{ "{{hub" }} index .ManagedClusterLabels $label {{ "hub}}" }}"{{ "{{hub" }} end {{ "hub}}" }}{{ "{{hub" }} end {{ "hub}}" }} {{ "}}" }}
```

Or refactor to build the list properly with proper spacing.

### Impact

- ❌ Cannot deploy infra nodes across multiple availability zones via labels
- ❌ Loses high availability benefits of multi-AZ deployment
- ⚠️ Fallback to single-AZ deployment works but is not production-ready

---

## 🐛 Bug #2: Instance Type Label Not Applied

**Status:** 🔴 Blocking  
**Component:** `policies/stable/infra-nodes/templates/policy-infra-nodes-aws.yaml`  
**Severity:** High

### Description

The `autoshift.io/infra-nodes-instance-type` label is defined in the cluster labels but is **not applied** to the generated MachineSet. Instead, AutoShift uses the worker MachineSet's instance type as a fallback.

### Root Cause

The template contains logic to read the label:

```yaml
instanceType: {{ "{{" }} "{{ "{{hub" }} index .ManagedClusterLabels "autoshift.io/infra-nodes-instance-type" {{ "hub}}" }}" | default $worker_ms.spec.template.spec.providerSpec.value.instanceType {{ "}}" }}
```

However, the **hub template interpolation is not working correctly**, causing the label lookup to fail and always fall back to the worker instance type.

### Location

File: `policies/stable/infra-nodes/templates/policy-infra-nodes-aws.yaml`  
Lines: ~150-160

### Reproduction

1. Set cluster labels:
   ```yaml
   autoshift.io/infra-nodes: '3'
   autoshift.io/infra-nodes-provider: 'aws'
   autoshift.io/infra-nodes-instance-type: 'm6a.4xlarge'
   ```

2. Deploy AutoShift and wait for MachineSet creation

3. Check actual instance type:
   ```bash
   oc get machineset.machine.openshift.io <infra-machineset-name> -n openshift-machine-api \
     -o jsonpath='{.spec.template.spec.providerSpec.value.instanceType}'
   ```

4. Result: Shows worker instance type (e.g., `m6a.xlarge`) instead of `m6a.4xlarge`

### Expected Behavior

MachineSet should use the instance type specified in `autoshift.io/infra-nodes-instance-type` label.

### Actual Behavior

MachineSet uses the worker MachineSet's instance type, ignoring the label.

### Workaround

**Implemented in this repository:**

Create a supplemental ACM policy (`policies/policy-infra-nodes-machineset-fix.yaml`) that:
1. Depends on `policy-infra-nodes-aws` being Compliant
2. Reads the `infra-nodes-instance-type` label
3. Patches the MachineSet with the correct instance type

Apply after AutoShift deployment:
```bash
oc apply -f policies/policy-infra-nodes-machineset-fix.yaml
```

### Proposed Fix

Debug why the hub template interpolation `{{ "{{hub" }} index .ManagedClusterLabels "autoshift.io/infra-nodes-instance-type" {{ "hub}}" }}` is not evaluating correctly. The template should properly read the label value and use it instead of defaulting to the worker instance type.

### Impact

- ❌ Infra nodes created with incorrect (too small) instance type
- ❌ Insufficient CPU/memory for ACM + ACS + monitoring workloads
- ⚠️ Can cause resource pressure and pod evictions

---

## 🐛 Bug #3: Volume Size Label Not Applied

**Status:** 🔴 Blocking  
**Component:** `policies/stable/infra-nodes/templates/policy-infra-nodes-aws.yaml`  
**Severity:** High

### Description

The `autoshift.io/infra-nodes-volume-size` label is defined in the cluster labels but is **not applied** to the generated MachineSet's EBS volume configuration. The `volumeSize` field is entirely missing from the `blockDevices` configuration.

### Root Cause

The template copies the worker MachineSet's EBS configuration but doesn't include the `volumeSize` field or read it from the label:

```yaml
blockDevices:
  - ebs:
      encrypted: {{ $ebs.encrypted }}
      iops: {{ $ebs.iops }}
      # volumeSize is MISSING
      volumeType: {{ $ebs.volumeType }}
```

There's no logic to read `autoshift.io/infra-nodes-volume-size` and insert it into the MachineSet.

### Location

File: `policies/stable/infra-nodes/templates/policy-infra-nodes-aws.yaml`  
Lines: ~165-175

### Reproduction

1. Set cluster labels:
   ```yaml
   autoshift.io/infra-nodes: '3'
   autoshift.io/infra-nodes-provider: 'aws'
   autoshift.io/infra-nodes-volume-size: '120'
   ```

2. Deploy AutoShift and wait for MachineSet creation

3. Check EBS volume configuration:
   ```bash
   oc get machineset.machine.openshift.io <infra-machineset-name> -n openshift-machine-api \
     -o jsonpath='{.spec.template.spec.providerSpec.value.blockDevices[0].ebs}'
   ```

4. Result: No `volumeSize` field present (defaults to AWS default, typically 16GB)

### Expected Behavior

MachineSet should include:
```yaml
blockDevices:
  - ebs:
      volumeSize: 120
```

### Actual Behavior

MachineSet has no `volumeSize` field, causing AWS to use the default root volume size (16GB for most AMIs).

### Workaround

**Implemented in this repository:**

The supplemental policy `policies/policy-infra-nodes-machineset-fix.yaml` also patches the volume size:

```yaml
blockDevices:
  - ebs:
      volumeSize: {{ $volumeSize }}
```

### Proposed Fix

Add logic to read the `infra-nodes-volume-size` label and insert it into the blockDevices configuration:

```yaml
blockDevices:
  - ebs:
      encrypted: {{ $ebs.encrypted }}
      iops: {{ $ebs.iops }}
      {{- $volumeSize := index .ManagedClusterLabels "autoshift.io/infra-nodes-volume-size" | default "120" }}
      volumeSize: {{ $volumeSize | toInt }}
      volumeType: {{ $ebs.volumeType }}
```

### Impact

- ❌ Infra nodes created with **16GB root volumes** instead of 120GB
- ❌ Immediate **disk pressure** (85%+ usage) when running ACM + ACS + monitoring
- ❌ Pod evictions and CrashLoopBackOff errors
- 🔴 **Cluster is non-functional** - critical workloads cannot run

---

## ⚠️ Limitation #1: Single Availability Zone Without Zone Labels

**Status:** 🟡 Known Limitation  
**Component:** `policies/stable/infra-nodes/templates/policy-infra-nodes-aws.yaml`  
**Severity:** Medium

### Description

When zone labels are omitted (to avoid Bug #1), AutoShift creates **only 1 MachineSet** using the first worker MachineSet's availability zone. The `replicas` count is applied to this single MachineSet, resulting in all infra nodes in one AZ.

### Behavior

With labels:
```yaml
autoshift.io/infra-nodes: '3'
autoshift.io/infra-nodes-provider: 'aws'
# NO zone labels
```

AutoShift creates:
- ✅ 1 MachineSet: `cluster-name-infra-us-east-2a` with 3 replicas

Expected for HA:
- ❌ 3 MachineSets: one per AZ with 1 replica each

### Workaround

**Option A:** Manually create additional MachineSets in other AZs  
**Option B:** Wait for Bug #1 fix and use zone labels  
**Option C:** Create a supplemental policy to create MachineSets in additional zones

### Impact

- ⚠️ No high availability across availability zones
- ⚠️ Single AZ failure takes down all infra nodes
- ⚠️ Not suitable for production environments

---

## ⚠️ Limitation #2: No Control Plane Zone Detection

**Status:** 🟡 Design Limitation  
**Component:** `policies/stable/infra-nodes/templates/policy-infra-nodes-aws.yaml`  
**Severity:** Low

### Description

When zone labels are omitted, AutoShift falls back to the **worker MachineSet's zone** rather than detecting control plane zones. In a cluster with 0 workers (minimal hub architecture), this means:

1. Worker MachineSets exist but have 0 replicas
2. AutoShift reads the first worker MachineSet's zone (e.g., us-east-2a)
3. Infra MachineSet is created in that zone
4. But control plane nodes may span us-east-2a, us-east-2b, us-east-2c

### Expected Behavior

AutoShift could detect control plane node zones and create infra MachineSets in matching AZs for better distribution.

### Actual Behavior

Infra nodes go to the same zone as the (non-existent) workers.

### Impact

- ⚠️ Suboptimal zone distribution
- ℹ️ Works but not ideal for HA

---

## 📋 Summary and Recommendations

### Critical Bugs (Must Fix)

| Bug | Severity | Impact | Workaround Available |
|-----|----------|--------|---------------------|
| #1 - Zone label parsing | 🔴 High | No multi-AZ infra nodes | ✅ Omit zone labels (single AZ) |
| #2 - Instance type not applied | 🔴 High | Wrong instance size | ✅ Supplemental policy patch |
| #3 - Volume size not applied | 🔴 High | Disk pressure, cluster non-functional | ✅ Supplemental policy patch |

### Workarounds Implemented in This Repository

1. **`playbooks/deploy-autoshift-stack.yaml`**
   - Zone labels removed to avoid Bug #1
   - Accepts single-AZ deployment

2. **`policies/policy-infra-nodes-machineset-fix.yaml`**
   - Patches MachineSet instance type (Bug #2 fix)
   - Patches MachineSet volume size (Bug #3 fix)
   - Applied in sync-wave "2" after AutoShift

3. **`policies/policy-acs-infra-nodes.yaml`**
   - Ensures ACS components run on infra nodes
   - Includes both NoSchedule and NoExecute tolerations

4. **`policies/policy-acm-infra-nodes.yaml`**
   - Ensures ACM components run on infra nodes

### Upstream Contribution Plan

1. **File GitHub issues** in `auto-shift/autoshiftv2` for each bug
2. **Submit Pull Requests** with proposed fixes
3. **Test fixes** in this environment before submitting
4. **Document** in upstream README that zone labels are currently broken

### Testing Checklist for Fixes

Before submitting upstream fixes, verify:

- [ ] Zone labels parse correctly with 3+ zones defined
- [ ] Instance type label is read and applied to MachineSet
- [ ] Volume size label is read and applied to MachineSet  
- [ ] Multiple MachineSets created (one per zone) when zones specified
- [ ] MachineSets have correct replicas distribution across zones
- [ ] All infra nodes provision with correct instance type and volume size
- [ ] No disk pressure on infra nodes after deployment
- [ ] Platform workloads successfully migrate to infra nodes

---

## Additional Notes

### AutoShift Version Information

```bash
# Check AutoShift version from OCI registry
helm show chart oci://quay.io/autoshift/policies/infra-nodes

# Git commit referenced
git clone https://github.com/auto-shift/autoshiftv2.git
cd autoshiftv2
git log --oneline policies/stable/infra-nodes/ | head -5
```

### Related AutoShift Documentation

- [Infra Nodes Policy Values Reference](https://github.com/auto-shift/autoshiftv2/blob/main/policies/stable/infra-nodes/values.yaml)
- [Infra Nodes Policy Template](https://github.com/auto-shift/autoshiftv2/blob/main/policies/stable/infra-nodes/templates/policy-infra-nodes-aws.yaml)

### Contact

For questions about these bugs or workarounds:
- **This Repository:** [Internal documentation and workarounds]
- **Upstream AutoShift:** https://github.com/auto-shift/autoshiftv2/issues
