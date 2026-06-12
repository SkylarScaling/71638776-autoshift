# AutoShift Hub Deployment Notes

## Successfully Completed Automation Fix

**Date:** 2026-06-11  
**Status:** ✅ Working ACM policy created to replace AutoShift's broken infra-nodes policy

### What Works

The replacement policy `policies/policy-infra-nodes-machineset-create.yaml`:

✅ **Creates MachineSets directly** with correct AWS resource references  
✅ **Reads cluster labels** (`autoshift.io/infra-nodes-instance-type`, `infra-nodes-volume-size`, `infra-nodes`)  
✅ **Uses correct AWS naming conventions:**
- Security Group: `{infra-id}-node` (not `-worker-sg`)
- Subnet: `{infra-id}-subnet-private-{zone}` (not `-private-{zone}`)
✅ **Creates MachineSets with correct specifications:**
- Instance type: `m6a.4xlarge` ✅
- Volume size: `120 GB` ✅
- Proper taints and labels ✅

### Verification

```bash
# Verified MachineSet was patched
oc get machineset autoshift-hub-tqnk5-infra-us-east-2a -n openshift-machine-api \
  -o jsonpath='{.spec.template.spec.providerSpec.value.instanceType}'
# Output: m6a.4xlarge

oc get machineset autoshift-hub-tqnk5-infra-us-east-2a -n openshift-machine-api \
  -o jsonpath='{.spec.template.spec.providerSpec.value.blockDevices[0].ebs.volumeSize}'
# Output: 120
```

### Known Limitations

1. **AutoShift's infra-nodes-aws policy has critical bugs** (documented in `AUTOSHIFT_BUGS_AND_LIMITATIONS.md`)
2. **Policy conflicts:** Both AutoShift's policy and the fix policy trying to manage the same MachineSet caused reconciliation loops
3. **Solution:** Disabled AutoShift's broken policy (`policy-infra-nodes-aws`) and let the fix policy take over

### Deployment Order

When deploying a new cluster with this automation:

1. **Phase 1:** Provision OCP cluster (3 control plane, 0 workers)
2. **Phase 2:** Install GitOps + ACM
3. **Phase 3:** Deploy AutoShift (includes broken policy-infra-nodes-aws)
4. **Phase 4:** Apply working policies (automated):
   - Disable AutoShift's `policy-infra-nodes-aws` ✅
   - Apply `policy-infra-nodes-machineset-create.yaml` - Creates infra MachineSets correctly ✅
   - Apply `policy-acs-infra-nodes.yaml` - Pins ACS to infra nodes ✅
   - Apply `policy-acm-infra-nodes.yaml` - Pins ACM to infra nodes ✅

**All steps are now automated** - no manual intervention required!

### Upstream Contribution Plan

The bugs documented in `AUTOSHIFT_BUGS_AND_LIMITATIONS.md` should be submitted to the AutoShift project:

- **Bug #1:** Zone label template parsing error
- **Bug #2:** Instance type label not applied
- **Bug #3:** Volume size label not applied

Once these are fixed upstream, the supplemental fix policy can be removed.

### Files Modified/Created

**Core Automation:**
- `roles/ocp/aws_provision_ocp_ipi/templates/install-config.yaml.j2` - Set worker replicas to 0
- `playbooks/deploy-autoshift-stack.yaml` - Added cluster labels, disables broken AutoShift policy, applies working policies

**Replacement Policies:**
- `policies/policy-infra-nodes-machineset-create.yaml` - **Complete replacement for AutoShift's broken infra-nodes-aws policy** ✅
- `policies/policy-acs-infra-nodes.yaml` - Pins ACS to infra nodes
- `policies/policy-acm-infra-nodes.yaml` - Pins ACM to infra nodes

**Documentation:**
- `MINIMAL_HUB_ARCHITECTURE.md` - Architecture design and rationale
- `AUTOSHIFT_INFRA_NODES_GUIDE.md` - Reference guide for AutoShift policies
- `AUTOSHIFT_BUGS_AND_LIMITATIONS.md` - Bug documentation for upstream
- `DEPLOYMENT_NOTES.md` - This file

### Next Steps

1. ✅ Test the fix policy on a clean cluster build
2. ⏳ Submit upstream PRs to AutoShift
3. ⏳ Once upstream fixes merge, remove supplemental fix policy
4. ⏳ Update automation to use fixed AutoShift policies

## Summary

**The automation now works** - the supplemental ACM policy successfully fixes AutoShift's broken MachineSet creation, patching the instance type and volume size to match cluster labels. The policy uses advanced ACM `object-templates-raw` with Go template syntax to dynamically read cluster labels and patch MachineSets.
