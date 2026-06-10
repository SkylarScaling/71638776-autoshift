# Playbook Idempotency

The `deploy-autoshift-stack.yaml` playbook is designed to be **fully idempotent** - it can be run multiple times on the same cluster without causing errors or unnecessary changes.

## Idempotency Checks

### Phase 1: OCP Cluster Provisioning

#### Cluster Provisioning (ocp/aws_provision_ocp_ipi role)
- ✅ Checks for existing cluster directory: `{{ install_dir }}/{{ ocp_cluster.name }}/auth/kubeadmin-password`
- ✅ Skips IPI installation if cluster already exists
- ✅ Safe to re-run on provisioned clusters

#### Infrastructure MachineSets (ocp/aws_create_machineset role)
- ✅ Uses `kubernetes.core.k8s` with `state: present` (inherently idempotent)
- ✅ Creates MachineSets only if they don't exist
- ✅ Safe to re-run - won't recreate existing MachineSets

#### Storage MachineSets (ocp/aws_create_machineset role)
- ✅ Uses `kubernetes.core.k8s` with `state: present` (inherently idempotent)
- ✅ Creates MachineSets only if they don't exist
- ✅ Safe to re-run - won't recreate existing MachineSets

### Phase 2: AutoShift Stack Installation

#### OpenShift GitOps
- ✅ Checks if Helm release exists before installing
- ✅ Skips installation if `openshift-gitops` release already deployed
- ✅ Wait task succeeds immediately if operator already ready

**Implementation:**
```yaml
- name: Check if OpenShift GitOps Helm release exists
  # Returns 'true' if release exists
  
- name: Install OpenShift GitOps via Helm OCI
  when: gitops_helm_check.stdout != 'true'  # Only installs if not present
```

#### Advanced Cluster Management
- ✅ Checks if Helm release exists before installing
- ✅ Skips installation if `advanced-cluster-management` release already deployed
- ✅ Wait task succeeds immediately if MultiClusterHub already Running

**Implementation:**
```yaml
- name: Check if Advanced Cluster Management Helm release exists
  # Returns 'true' if release exists
  
- name: Install Advanced Cluster Management via Helm OCI
  when: acm_helm_check.stdout != 'true'  # Only installs if not present
```

#### AutoShift ArgoCD Application
- ✅ Uses `kubernetes.core.k8s` with `state: present` (inherently idempotent)
- ✅ Updates Application if spec changes, otherwise no-op
- ✅ Safe to re-run multiple times

#### ClusterSet Labeling
- ✅ Uses `kubernetes.core.k8s` with `state: present` (inherently idempotent)
- ✅ Adds labels if missing, preserves if already present
- ✅ Safe to re-run multiple times

#### Supplemental Policies
- ✅ Waits for `policies-autoshift` namespace to exist (succeeds immediately if present)
- ✅ Uses `kubernetes.core.k8s` with `state: present` (inherently idempotent)
- ✅ ACM policies are additive - safe to re-apply

## Testing Idempotency

To verify idempotency, run the playbook twice on the same cluster:

```bash
# First run - provisions everything
ansible-playbook playbooks/deploy-autoshift-stack.yaml -i <inventory>

# Second run - should skip most tasks, show "ok" status
ansible-playbook playbooks/deploy-autoshift-stack.yaml -i <inventory>
```

Expected second-run behavior:
- ✅ All "Check if ... exists" tasks should show existing resources
- ✅ Helm install tasks should be **skipped** (not changed)
- ✅ `kubernetes.core.k8s` tasks should show **ok** (not changed)
- ✅ Wait tasks should succeed immediately
- ✅ No errors or failures
- ✅ Cluster state unchanged

## Benefits

1. **Safe Re-runs**: Can re-run after failures without duplicating resources
2. **Partial Recovery**: If a step fails, re-running picks up where it left off
3. **Configuration Updates**: Can update ArgoCD Application values and re-apply
4. **Development Workflow**: Safe to iterate during development/testing

## Known Limitations

- The playbook does **not** upgrade existing Helm releases (by design)
- To upgrade GitOps or ACM, manually run `helm upgrade` or delete the release first
- ArgoCD Application values changes **will** trigger Application updates
- ACM policies use `musthave`/`mustonlyhave` - changing policy content will update managed resources
