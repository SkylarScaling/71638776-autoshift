# OpenShift & AutoShift Deployment Automation

This repository contains the Ansible playbooks and configurations required to provision the Red Hat OpenShift Container 
Platform (OCP) on AWS and Nutanix, and deploy the AutoShift stack (GitOps and Advanced Cluster Management). This automation supports 
a hub-and-spoke centralized management architecture for distributed workload environments.

## Documentation

- **[Deployment Guide Overview](DEPLOYMENT_GUIDE_OVERVIEW.md)**: Architecture patterns and use cases
- **[Nutanix Disconnected Install Guide](NUTANIX_DISCONNECTED_INSTALL.md)**: Complete guide for air-gapped deployments on Nutanix AHV
- **[Playbook Idempotency Guide](playbooks/IDEMPOTENCY.md)**: Details on safe re-execution of automation

## Repository Structure
*   `playbooks/aws/`: Playbooks for managing AWS infrastructure limits (e.g., Elastic IP Quotas).
*   `playbooks/ocp/`: Playbooks for provisioning the OpenShift cluster via IPI, installing CLI prerequisites, and defining custom MachineSets.
*   `playbooks/deploy-autoshift-stack.yaml`: The primary playbook that provisions the OCP cluster and layers on the OpenShift GitOps and ACM components via OCI registries.
*   `playbooks/apply-supplemental-policies.yaml`: Applies supplemental ACM policies that extend autoshift functionality (useful for existing clusters).
*   `playbooks/group_vars/`: Directory for variable definition files driving the deployment.
*   `policies/`: Supplemental ACM policies that customize autoshift deployments beyond upstream capabilities.

## Prerequisites

Before utilizing this repository, ensure your local deployment host has the following tools installed:

*   [Python 3+](https://www.python.org/downloads/)
*   [pip and setuptools](https://packaging.python.org/en/latest/tutorials/installing-packages/#ensure-pip-setuptools-and-wheel-are-up-to-date)
*   [Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)
*   [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html)

*Note: Ansible requires Python 3 to use the required modules, so it is recommended to install it using `pip3 install ansible`*.

*Note: You can use the following repo to easily set up a local Ansible execution environment: https://github.com/SkylarScaling/ansible-core-setup-for-linux*

## Setup & Execution

### Example Inventory

```yaml
---
all:
  hosts:
    localhost:
      ansible_connection: local

  vars:
    # --------------------------------------------------------------------------
    # General Directory Variables (Overrides for group_vars/all.yaml)
    # --------------------------------------------------------------------------
    home_dir: /home/REPLACE_ME
    tmp_dir: "{{ home_dir }}/tmp"

    # --------------------------------------------------------------------------
    # OpenShift Cluster Settings
    # --------------------------------------------------------------------------
    ocp_cluster:
      name: "autoshift-dev"
    base_domain: "sandboxXXXX.opentlc.com"
    openshift_version: "4.20"      # Must be X.Y format as per your assert task
    ocp_patch_version: "18"        # Must be strictly numeric
    ssh_key: "{{ lookup('file', '~/.ssh/id_ed25519.pub') }}"
    pull_secret: "{{ lookup('file', '~/pull-secret.json') | from_json }}"

    # --------------------------------------------------------------------------
    # AWS Configuration & Credentials
    # NOTE: In a real-world scenario, you might want to use Ansible Vault 
    # to encrypt the secret_access_key.
    # --------------------------------------------------------------------------
    aws:
      aws_access_key_id: REPLACE_ME
      aws_secret_access_key: REPLACE_ME
      aws_region: us-east-2

    # --------------------------------------------------------------------------
    # Custom MachineSet Configuration (Infra & Storage)
    # Used by playbooks/ocp/aws-create-infra-machinesets.yaml
    # --------------------------------------------------------------------------
    install_config:
      cluster_name: "{{ ocp_cluster.name }}"
      infra:
        replicas: 3
        instance_type: "m5.2xlarge"
      storage:
        replicas: 3
        instance_type: "m5.4xlarge"
...
```

### Deploy the Hub Cluster and AutoShift Stack
To provision the baseline OCP cluster and install OpenShift GitOps and Advanced Cluster Management, run the primary deployment playbook:
```bash
ansible-playbook playbooks/deploy-autoshift-stack.yaml -i <path_to_inventory>
```

**Note:** This playbook is **fully idempotent** and can be safely re-run multiple times. See [playbooks/IDEMPOTENCY.md](playbooks/IDEMPOTENCY.md) for details.

This playbook will:
1. Provision an OCP cluster on AWS using IPI
2. Create custom MachineSets for infrastructure and storage nodes
3. Install OpenShift GitOps via Helm from OCI registry
4. Install Advanced Cluster Management via Helm from OCI registry
5. Deploy the AutoShift ArgoCD Application pointing to autoshift policies
6. Apply supplemental policies (e.g., configure ACS to use infrastructure nodes)

### Apply Supplemental Policies to Existing Cluster
If you need to apply or update supplemental policies on an already-deployed cluster:
```bash
ansible-playbook playbooks/apply-supplemental-policies.yaml -i <path_to_inventory>
```

Supplemental policies include:
- **ACS Infrastructure Nodes**: Configures Advanced Cluster Security Central to run on infrastructure nodes instead of worker nodes

## AutoShift Bug Fixes (IMPORTANT)

This automation includes **critical bug fixes** for AutoShift v2's infrastructure node provisioning. See [AUTOSHIFT_BUGS_AND_LIMITATIONS.md](AUTOSHIFT_BUGS_AND_LIMITATIONS.md) for complete documentation.

### What's Fixed Automatically

The automation **disables AutoShift's broken `policy-infra-nodes-aws`** and applies a working replacement policy:

**File:** `policies/policy-infra-nodes-machineset-create.yaml`

This policy correctly:
- ✅ Reads `infra-nodes-instance-type` and `infra-nodes-volume-size` cluster labels
- ✅ Uses correct AWS resource naming (`{infra-id}-node` security group, `{infra-id}-subnet-private-{zone}` subnet)
- ✅ Creates MachineSets with proper instance type and volume size
- ✅ Prevents disk pressure issues on infra nodes

**No manual intervention required** - the fix is applied automatically during deployment.

### Fresh Deployment Expectations

On a fresh build, you will see:

1. **Infra nodes provision correctly:**
   - Instance type: As specified in `infra-nodes-instance-type` label (e.g., m6a.4xlarge)
   - Volume size: As specified in `infra-nodes-volume-size` label (e.g., 120 GB)
   - No disk pressure

2. **One policy shows NonCompliant (expected):**
   - `policy-infra-nodes-aws`: NonCompliant (disabled due to bugs)
   - `policy-infra-nodes-machineset-create`: Compliant (our working replacement)

3. **Single-AZ deployment:**
   - All 3 infra nodes created in one availability zone (workaround for zone label parsing bug)
   - Acceptable for dev/lab environments

See [DEPLOYMENT_NOTES.md](DEPLOYMENT_NOTES.md) for detailed upgrade procedure.
