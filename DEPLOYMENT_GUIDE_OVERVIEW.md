# Deployment Guide Overview

This repository contains comprehensive guides for deploying OpenShift Container Platform in disconnected/air-gapped environments for regulated manufacturing industries.

## Available Documentation

### 1. Nutanix Disconnected Installation Guide
**File**: `NUTANIX_DISCONNECTED_INSTALL.md`

**Purpose**: Complete step-by-step guide for deploying OpenShift 4.20 on Nutanix AHV in disconnected environments for PAS-X MES workload clusters.

**Key Features**:
- Full air-gapped/disconnected installation procedures
- RHEL 9 specific instructions
- Mirror registry setup and configuration
- Custom MachineSet creation for specialized workloads
- Multi-tier storage configuration
- GxP regulatory compliance guidance
- Disaster recovery strategies

**Target Audience**: 
- Platform engineers deploying production manufacturing workload clusters
- Teams implementing PAS-X MES on OpenShift in regulated environments
- Organizations requiring air-gapped Kubernetes deployments

### 2. AutoShift Hub Deployment Automation
**File**: `playbooks/deploy-autoshift-stack.yaml`

**Purpose**: Automated deployment of ACM Hub clusters for centralized multi-cluster management.

**Key Features**:
- AWS IPI cluster provisioning
- OpenShift GitOps installation via Helm OCI
- Advanced Cluster Management installation via Helm OCI
- Infrastructure and storage MachineSets
- Fully idempotent playbook design

**Target Audience**:
- Platform teams deploying centralized management infrastructure
- Multi-site deployment scenarios (hub-and-spoke architecture)

## Architecture Patterns

### Hub-and-Spoke Multi-Cluster Management

This repository supports a distributed architecture pattern:

- **Management Hub**: Single ACM hub cluster for centralized visibility and policy enforcement
- **Workload Clusters**: Distributed OpenShift clusters at manufacturing sites running PAS-X MES applications
- **Local Registries**: Site-local mirrored container registries for air-gapped operations

### Cluster Sizing Profiles

#### PAS-X Workload Cluster (Manufacturing Site)
- **Control Plane**: 3 nodes (4 vCPU, 16 GB RAM each)
- **Specialized Worker Nodes**:
  - PAS-X Central: 2 nodes (20 vCPU, 32 GB RAM) - Tainted
  - Database (PostgreSQL): 3 nodes (8 vCPU, 24 GB RAM) - Tainted
  - PAS-X PDA: 2 nodes (8 vCPU, 24 GB RAM) - Tainted
  - PAS-X Misc: 4 nodes (8 vCPU, 32 GB RAM) - General workloads
- **Total**: 17 nodes, 136 vCPU, 408 GB RAM

#### ACM Hub Cluster (Management)
- **Control Plane**: 3 nodes (4 vCPU, 16 GB RAM each)
- **Infrastructure Nodes**: 3+ nodes (8 vCPU, 32 GB RAM each)
  - No worker subscription consumption
  - Hosts ACM, Multi-Cluster Observability, ingress controllers
- **Total**: 6+ nodes, 36+ vCPU, 144+ GB RAM

#### Generic Production Cluster
- **Control Plane**: 3 nodes (8 vCPU, 32 GB RAM each)
- **Worker Nodes**: 3+ nodes (8 vCPU, 32 GB RAM each)

## Technology Stack

### Core OpenShift Components
- OpenShift Container Platform 4.20
- Red Hat Advanced Cluster Management (RHACM/ACM)
- OpenShift GitOps (ArgoCD)
- Multi-Cluster Observability (MCO)

### Infrastructure
- **Compute**: Nutanix AHV (workload clusters) or AWS EC2 (hub clusters)
- **Storage Tiers**:
  - Block: Nutanix CSI (high-IOPS for databases)
  - File: NFS/SMB (shared application files)
  - Archive: NetApp/Data Domain (long-term storage)
- **Networking**: Static IPs, encrypted communication (TLS/HTTPS)

### Deployment Tooling
- Ansible (Infrastructure-as-Code)
- Helm (OCI registry-based deployments)
- GitOps (declarative configuration management)

## Regulatory Compliance Features

### GxP Validation Support
- **IQ (Installation Qualification)**: Automated evidence generation during cluster build
- **OQ (Operational Qualification)**: GitOps-based change control with full audit trail
- **PQ (Performance Qualification)**: Load testing validation for concurrent users and batch processing

### Data Security
- Encryption at rest (etcd encryption)
- Encryption in transit (TLS for all communication)
- Audit logging for regulatory compliance
- LDAP/Active Directory integration for RBAC

### FDA 21 CFR Part 11 Compliance
- Electronic signatures via Git commits
- Audit trails via GitOps
- Data integrity via immutable infrastructure
- Access controls via OpenShift RBAC

## Use Cases

### Manufacturing Execution Systems (MES)
- PAS-X 3.4 application hosting
- Distributed manufacturing sites (5+ locations)
- High availability and disaster recovery
- Regulatory compliance (pharmaceutical, biotech)

### Multi-Cluster Management
- Centralized visibility across distributed sites
- Fleet-wide policy enforcement
- Unified backup and disaster recovery
- Coordinated upgrades and maintenance

### Air-Gapped/Disconnected Environments
- Zero internet connectivity from production clusters
- Local container registry mirrors
- Operator lifecycle management via mirrored catalogs
- Security-first design for regulated industries

## Getting Started

### For Workload Clusters (Manufacturing Sites)
1. Review prerequisites in `NUTANIX_DISCONNECTED_INSTALL.md`
2. Set up internet-connected jumphost for mirroring (Steps 1-3)
3. Transfer mirrored content to disconnected environment (Step 4)
4. Deploy cluster using install-config.yaml templates (Steps 5-7)
5. Create custom MachineSets for specialized workloads (Step 8)
6. Configure storage, networking, and RBAC (Step 9)
7. Perform validation testing (IQ/OQ/PQ)

### For ACM Hub Cluster
1. Prepare AWS credentials and inventory file
2. Run `ansible-playbook playbooks/deploy-autoshift-stack.yaml -i inventory`
3. Import workload clusters into ACM
4. Configure Multi-Cluster Observability
5. Implement GitOps application delivery

## Best Practices

### Network Architecture
- Use static IPs for all infrastructure components
- Implement firewall rules at the network edge (see network requirements table)
- Keep OpenShift internal networks non-routed (Service/Cluster networks)
- Encrypt all traffic between components

### Storage Strategy
- Block storage (Nutanix): Databases, message queues
- File storage (NFS/SMB): Shared application files, reports
- Object/Archive: Long-term retention, compliance

### Disaster Recovery
- Daily automated etcd backups
- Continuous database backups to enterprise storage
- Git-based configuration recovery (zero RPO)
- RTO target: 4-6 hours for full site recovery

### Change Management
- All changes via GitOps (Git repository)
- Pull request reviews before production
- Automated testing in non-production environments
- Full audit trail via Git commit history

## Support and Contributions

This repository is maintained as reference architecture for OpenShift deployments in regulated manufacturing environments. 

**Note**: Customer-specific information has been removed from this public repository. The architecture patterns and technical guidance are generalizable to any manufacturing organization requiring:
- Air-gapped OpenShift deployments
- GxP regulatory compliance
- Multi-site distributed architecture
- Manufacturing Execution System (MES) hosting

## Additional Resources

- [OpenShift 4.20 Documentation](https://docs.openshift.com/container-platform/4.20/)
- [Red Hat Advanced Cluster Management Documentation](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/)
- [OpenShift GitOps Documentation](https://docs.openshift.com/container-platform/4.20/cicd/gitops/understanding-openshift-gitops.html)
- [RHEL 9 Documentation](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9)
- [Nutanix OpenShift Integration](https://docs.openshift.com/container-platform/4.20/installing/installing_nutanix/preparing-to-install-on-nutanix.html)
