# OpenShift & AutoShift Deployment Automation

This repository contains the Ansible playbooks and configurations required to provision the Red Hat OpenShift Container 
Platform (OCP) on AWS and deploy the AutoShift stack (GitOps and Advanced Cluster Management). This automation supports 
a hub-and-spoke centralized management architecture for distributed workload environments.

## Repository Structure
*   `playbooks/aws/`: Playbooks for managing AWS infrastructure limits (e.g., Elastic IP Quotas).
*   `playbooks/ocp/`: Playbooks for provisioning the OpenShift cluster via IPI, installing CLI prerequisites, and defining custom MachineSets.
*   `playbooks/deploy-autoshift-stack.yaml`: The primary playbook that provisions the OCP cluster and layers on the OpenShift GitOps and ACM components via OCI registries.
*   `playbooks/group_vars/`: Directory for variable definition files driving the deployment.

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
ansible-playbook deploy-autoshift-stack.yaml -i <path_to_inventory>
```
