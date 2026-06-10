# OpenShift 4.20 Disconnected Installation on Nutanix AHV

Complete step-by-step guide for installing OpenShift 4.20 on Nutanix AHV in a disconnected environment using a jumphost.

**Purpose**: This guide provides production-ready cluster deployment procedures for **workload clusters** in distributed manufacturing environments running PAS-X MES applications.

For **ACM Hub cluster** deployment, refer to the [`deploy-autoshift-stack.yaml`](playbooks/deploy-autoshift-stack.yaml) playbook in this repository.

## Documentation Sources

This guide is based on official Red Hat OpenShift documentation:

- [Creating a mirror registry with mirror registry for Red Hat OpenShift](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/disconnected_environments/installing-mirroring-creating-registry) - Official Red Hat documentation for mirror-registry tool
- [Mirroring images for a disconnected installation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/disconnected_environments/installing-mirroring-disconnected) - Image mirroring with oc-mirror plugin
- [OpenShift 4.20 Disconnected Environments (PDF)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/pdf/disconnected_environments/OpenShift_Container_Platform-4.20-Disconnected_environments-en-US.pdf) - Complete offline documentation
- [Quay mirror-registry GitHub Repository](https://github.com/quay/mirror-registry) - Official mirror-registry tool source and documentation
- [Installing OpenShift step-by-step (Community Guide)](https://hackmd.io/@johnsimcall/Sk1gG5G6o)

## Prerequisites

**Jumphost Requirements:**
- **Red Hat Enterprise Linux 9.x** with internet access initially (for mirroring)
  - RHEL 8 is also supported but RHEL 9 is recommended for OpenShift 4.20
  - AlmaLinux 9 is also supported as a deployment host alternative
- **Minimum 500GB disk space** for container registry mirror
  - Base OS: 50GB
  - **Container registry mirror**: 400-500GB (for OCP 4.20 + full operator catalog)
  - Temporary files: 50GB
- Access to both internet-connected and air-gapped networks (or ability to transfer data)
- **Podman 4.0 or later** (included in RHEL 9)
- **OpenSSL** (included in RHEL 9)

**Nutanix Environment:**
- Nutanix AHV cluster with Prism Central
- **Sufficient resources** for workload cluster nodes (see cluster sizing below)
  - Minimum: 14 nodes, 124 vCPU, 360 GB RAM per site
- Network VLAN with **static IPs** for infrastructure components (reduces firewall complexity)
- DNS configured for cluster domains
- **No internet access** from cluster nodes (air-gapped/GxP-controlled environment)

**Storage Infrastructure:**
- Nutanix local block storage (high-IOPS for databases)
- Windows NFS/SMB file server (for shared application files)
- Optional: NetApp/Data Domain for long-term archive

**Required Credentials:**
- Red Hat pull secret (from console.redhat.com)
- Nutanix Prism Central credentials (service account for cluster provisioning)
- Nutanix Prism Element UUID and Subnet UUID
- SSH key pair (ed25519 recommended)
- Active Directory/LDAP bind credentials (for RBAC integration)

## Step 1: Prepare Jumphost (Internet-Connected Phase)

### 1.0 Verify RHEL 9 System

**IMPORTANT**: All commands in this guide are written for **Red Hat Enterprise Linux 9.x**. 

```bash
# Verify RHEL version (should show 9.x)
cat /etc/redhat-release

# Should output something like: "Red Hat Enterprise Linux release 9.4 (Plow)"
# Minimum RHEL 9.0 required, RHEL 9.2+ recommended for OpenShift 4.20

# Verify system architecture
uname -m
# Should output: x86_64

# Check available disk space (need 500GB+ free)
df -h /opt /tmp ~
```

### 1.1 Install Required Tools

**Note**: This guide uses `dnf` (default package manager in RHEL 9), not `yum`.

```bash
# Ensure system is registered with Red Hat Subscription Manager
sudo subscription-manager register
sudo subscription-manager attach --auto

# Enable required repositories (usually enabled by default in RHEL 9)
sudo subscription-manager repos --enable=rhel-9-for-x86_64-baseos-rpms
sudo subscription-manager repos --enable=rhel-9-for-x86_64-appstream-rpms

# Update system to latest packages
sudo dnf update -y

# Install dependencies (all available in RHEL 9 default repos)
# Note: RHEL 9 uses Podman 4.x by default (Docker is not included)
sudo dnf install -y podman httpd-tools jq skopeo openssl tar gzip wget

# Verify podman version (RHEL 9 includes Podman 4.0+)
podman --version
# Should show version 4.0 or higher

# Start and enable podman socket (optional, for systemd integration)
systemctl --user enable --now podman.socket

# Download OpenShift installer and CLI
export VERSION=4.20.18
export ARCH=x86_64

wget https://mirror.openshift.com/pub/openshift-v4/${ARCH}/clients/ocp/${VERSION}/openshift-install-linux.tar.gz
wget https://mirror.openshift.com/pub/openshift-v4/${ARCH}/clients/ocp/${VERSION}/openshift-client-linux.tar.gz

# Extract binaries
tar -xzf openshift-install-linux.tar.gz
tar -xzf openshift-client-linux.tar.gz

# Move to PATH
sudo mv oc kubectl openshift-install /usr/local/bin/
sudo chmod +x /usr/local/bin/{oc,kubectl,openshift-install}

# Verify installations
oc version
openshift-install version
podman version
```

### 1.2 Download and Install Mirror Registry

```bash
# Download the mirror registry tool from Red Hat
cd ~
wget https://developers.redhat.com/content-gateway/rest/mirror/pub/openshift-v4/clients/mirror-registry/latest/mirror-registry.tar.gz

# Alternative download locations (if above fails):
# wget https://mirror.openshift.com/pub/openshift-v4/clients/mirror-registry/latest/mirror-registry.tar.gz
# OR from GitHub releases: https://github.com/quay/mirror-registry/releases

# Extract the tool
tar -xzf mirror-registry.tar.gz

# Install mirror registry (this creates a self-contained Quay registry with podman)
# Note: Use sudo for local installation
sudo ./mirror-registry install \
  --quayHostname $(hostname) \
  --quayRoot /opt/quay \
  --initPassword passw0rd \
  --initUser init \
  --verbose

# The installer will output the registry URL and credentials
# Registry will be available at: https://$(hostname):8443
# Credentials: init / passw0rd (or auto-generated if --initPassword not specified)

# Trust the certificate (mirror-registry auto-generates it in quay-rootCA directory)
sudo cp /opt/quay/quay-rootCA/rootCA.pem /etc/pki/ca-trust/source/anchors/quay-rootCA.pem
sudo update-ca-trust

# Verify registry is running
podman login -u init -p passw0rd --tls-verify=false $(hostname):8443

# OR test with curl
curl -k -u init:passw0rd https://$(hostname):8443/v2/_catalog

# Configure firewall (firewalld is default in RHEL 9)
sudo firewall-cmd --add-port=8443/tcp --permanent
sudo firewall-cmd --reload

# Verify firewall rule
sudo firewall-cmd --list-ports

# Note: SELinux is enabled by default in RHEL 9
# The mirror-registry tool handles SELinux contexts automatically
# Verify SELinux is not blocking (should see "Enforcing")
getenforce
```

#### Mirror Registry Installation Options

The `mirror-registry` tool supports additional configuration options:

```bash
# Remote installation (install on a different host)
./mirror-registry install \
  --targetHostname registry.example.com \
  --targetUsername admin \
  --ssh-key ~/.ssh/id_rsa \
  --quayHostname registry.example.com \
  --quayRoot /opt/quay \
  --initPassword passw0rd \
  --verbose

# Custom SSL certificates (instead of auto-generated)
./mirror-registry install \
  --sslCert /path/to/ssl.cert \
  --sslKey /path/to/ssl.key \
  --quayHostname $(hostname) \
  --quayRoot /opt/quay

# Specify custom storage paths
./mirror-registry install \
  --quayRoot /opt/quay \
  --quayStorage /data/quay-storage \
  --pgStorage /data/postgres-data \
  --verbose
```

**Mirror Registry Ports:**
- **8443**: HTTPS access to Quay registry (default)
- **5432**: PostgreSQL database (internal)
- **6379**: Redis cache (internal)

**Alternative: Manual Registry Setup (not recommended)**

If the mirror-registry tool is not available, you can set up a basic container registry manually. However, Red Hat recommends using the mirror-registry tool for OpenShift disconnected installations.

```bash
# This alternative is NOT recommended - use mirror-registry tool instead
export REGISTRY_DIR=/opt/registry
sudo mkdir -p ${REGISTRY_DIR}/{auth,certs,data}

sudo openssl req -newkey rsa:4096 -nodes -sha256 -keyout ${REGISTRY_DIR}/certs/domain.key \
  -x509 -days 365 -out ${REGISTRY_DIR}/certs/domain.crt \
  -subj "/C=US/ST=State/L=City/O=Company/CN=$(hostname)" \
  -addext "subjectAltName=DNS:$(hostname),DNS:localhost,IP:$(hostname -I | awk '{print $1}')"

sudo htpasswd -bBc ${REGISTRY_DIR}/auth/htpasswd admin passw0rd
sudo cp ${REGISTRY_DIR}/certs/domain.crt /etc/pki/ca-trust/source/anchors/
sudo update-ca-trust

sudo podman run -d --name mirror-registry \
  -p 5000:5000 \
  -v ${REGISTRY_DIR}/data:/var/lib/registry:z \
  -v ${REGISTRY_DIR}/auth:/auth:z \
  -v ${REGISTRY_DIR}/certs:/certs:z \
  -e REGISTRY_AUTH=htpasswd \
  -e REGISTRY_AUTH_HTPASSWD_REALM="Registry Realm" \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
  --restart=always \
  docker.io/library/registry:2

sudo firewall-cmd --add-port=5000/tcp --permanent
sudo firewall-cmd --reload
```

**Note**: The rest of this guide assumes you're using the **official mirror-registry tool** (port 8443). If using the manual registry setup, substitute `$(hostname):8443` with `$(hostname):5000` and `init` with `admin` in all subsequent commands.

## Step 2: Mirror OpenShift Content

### 2.1 Prepare Environment Variables

```bash
export OCP_RELEASE=4.20.18
export LOCAL_REGISTRY="$(hostname):8443"  # Port 8443 for mirror-registry, 5000 for manual registry
export LOCAL_REPOSITORY='ocp4/openshift4'
export PRODUCT_REPO='openshift-release-dev'
export LOCAL_SECRET_JSON='/root/pull-secret-mirror.json'
export RELEASE_NAME="ocp-release"
export ARCHITECTURE=x86_64
```

### 2.2 Create Mirror Pull Secret

```bash
# Download your pull secret from console.redhat.com to ~/pull-secret.json

# Create mirror pull secret with local registry credentials
# Note: Using 'init' user which is the default for mirror-registry
cat ~/pull-secret.json | jq '.auths += {"'${LOCAL_REGISTRY}'": {"auth": "'$(echo -n init:passw0rd | base64 -w0)'","email": "init@example.com"}}' > ${LOCAL_SECRET_JSON}
```

### 2.3 Mirror OpenShift Images

```bash
# This will take 1-2 hours depending on bandwidth
oc adm release mirror -a ${LOCAL_SECRET_JSON} \
  --from=quay.io/${PRODUCT_REPO}/${RELEASE_NAME}:${OCP_RELEASE}-${ARCHITECTURE} \
  --to=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY} \
  --to-release-image=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}:${OCP_RELEASE}-${ARCHITECTURE}

# Save the output - you'll need the imageContentSources section
```

### 2.4 Mirror OperatorHub Content (Optional but Recommended)

```bash
# Download operator catalog pruning tools
wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/${OCP_RELEASE}/opm-linux.tar.gz
tar -xzf opm-linux.tar.gz
sudo mv opm /usr/local/bin/

# Mirror Red Hat operator catalogs
oc adm catalog mirror registry.redhat.io/redhat/redhat-operator-index:v4.20 \
  ${LOCAL_REGISTRY}/olm \
  -a ${LOCAL_SECRET_JSON} \
  --filter-by-os="linux/amd64"

oc adm catalog mirror registry.redhat.io/redhat/certified-operator-index:v4.20 \
  ${LOCAL_REGISTRY}/olm \
  -a ${LOCAL_SECRET_JSON} \
  --filter-by-os="linux/amd64"
```

## Step 3: Transfer Content to Disconnected Environment

### 3.1 Package Registry Data

**For mirror-registry tool installation:**

```bash
# Stop the mirror registry services
cd ~
sudo systemctl stop quay-app quay-postgres quay-redis 2>/dev/null || true
# OR use the uninstall command without auto-approve to preserve data
# sudo ./mirror-registry uninstall --quayRoot /opt/quay

# Create tarball (this will be large - 100GB+)
sudo tar -czf /tmp/quay-data.tar.gz -C /opt/quay .
sudo tar -czf /tmp/quay-certs.tar.gz -C /etc/pki/ca-trust/source/anchors/ quay-rootCA.pem

# Copy binaries and configs
mkdir /tmp/ocp-binaries
cp /usr/local/bin/{oc,kubectl,openshift-install} /tmp/ocp-binaries/
cp ${LOCAL_SECRET_JSON} /tmp/ocp-binaries/
cp ~/mirror-registry.tar.gz /tmp/ocp-binaries/

# Copy the mirror-registry executable
cp ~/mirror-registry /tmp/ocp-binaries/

# Save the image content sources output
# Copy any imageContentSources-*.yaml files generated during mirroring
cp imageset-config.yaml /tmp/ocp-binaries/ 2>/dev/null || true
```

**For manual registry installation:**

```bash
# Stop the registry
sudo podman stop mirror-registry

# Create tarball (this will be large - 100GB+)
sudo tar -czf /tmp/registry-data.tar.gz -C /opt/registry .
sudo tar -czf /tmp/registry-certs.tar.gz -C /etc/pki/ca-trust/source/anchors/ domain.crt

# Copy binaries and configs
mkdir /tmp/ocp-binaries
cp /usr/local/bin/{oc,kubectl,openshift-install} /tmp/ocp-binaries/
cp ${LOCAL_SECRET_JSON} /tmp/ocp-binaries/
```

### 3.2 Transfer to Disconnected Jumphost

**For mirror-registry installation**, transfer these files to your disconnected jumphost:
- `/tmp/quay-data.tar.gz`
- `/tmp/quay-certs.tar.gz`
- `/tmp/ocp-binaries/` (includes mirror-registry tool and executable)
- Any `imageContentSources` and `catalogSource` YAML files generated during mirroring

**For manual registry installation**, transfer these files:
- `/tmp/registry-data.tar.gz`
- `/tmp/registry-certs.tar.gz`
- `/tmp/ocp-binaries/`
- Any `imageContentSources` and `catalogSource` YAML files generated during mirroring

## Step 4: Set Up Disconnected Jumphost

**IMPORTANT**: The disconnected jumphost must also be running **RHEL 9.x** with the same prerequisites as the internet-connected jumphost.

### 4.1 Restore Registry

**For mirror-registry tool installation:**

```bash
# On the disconnected RHEL 9 jumphost
# Ensure required packages are installed
sudo dnf install -y podman openssl tar gzip

# Verify podman version (4.0+ required)
podman --version

# Install binaries
cd ~
tar -xzf ocp-binaries/mirror-registry.tar.gz
cp ocp-binaries/mirror-registry .
chmod +x mirror-registry

sudo cp ocp-binaries/{oc,kubectl,openshift-install} /usr/local/bin/
sudo chmod +x /usr/local/bin/{oc,kubectl,openshift-install}

# Install certificate FIRST (before extracting data)
sudo tar -xzf quay-certs.tar.gz -C /etc/pki/ca-trust/source/anchors/
sudo update-ca-trust

# Extract registry data
sudo mkdir -p /opt/quay
sudo tar -xzf quay-data.tar.gz -C /opt/quay/

# Start the mirror registry with existing data
# The install command will detect existing /opt/quay and use it
sudo ./mirror-registry install \
  --quayHostname $(hostname) \
  --quayRoot /opt/quay \
  --initPassword passw0rd \
  --initUser init \
  --verbose

# Verify registry is running
podman login -u init -p passw0rd --tls-verify=false $(hostname):8443

# Check catalog
curl -k -u init:passw0rd https://$(hostname):8443/v2/_catalog
```

**For manual registry installation:**

```bash
# Extract registry data
sudo mkdir -p /opt/registry
sudo tar -xzf registry-data.tar.gz -C /opt/registry/

# Install certificate
sudo tar -xzf registry-certs.tar.gz -C /etc/pki/ca-trust/source/anchors/
sudo update-ca-trust

# Install binaries
sudo cp ocp-binaries/* /usr/local/bin/
sudo chmod +x /usr/local/bin/{oc,kubectl,openshift-install}

# Install podman if not present
sudo dnf install -y podman

# Start registry
sudo podman run -d --name mirror-registry \
  -p 5000:5000 \
  -v /opt/registry/data:/var/lib/registry:z \
  -v /opt/registry/auth:/auth:z \
  -v /opt/registry/certs:/certs:z \
  -e REGISTRY_AUTH=htpasswd \
  -e REGISTRY_AUTH_HTPASSWD_REALM="Registry Realm" \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
  --restart=always \
  docker.io/library/registry:2
```

### 4.2 Configure Firewall

**Note**: RHEL 9 uses firewalld by default. Ensure the firewalld service is running.

**For mirror-registry (port 8443):**
```bash
# Ensure firewalld is running (default in RHEL 9)
sudo systemctl enable --now firewalld
sudo systemctl status firewalld

# Add port 8443
sudo firewall-cmd --add-port=8443/tcp --permanent
sudo firewall-cmd --reload

# Verify
sudo firewall-cmd --list-ports
```

**For manual registry (port 5000):**
```bash
sudo firewall-cmd --add-port=5000/tcp --permanent
sudo firewall-cmd --reload
sudo firewall-cmd --list-ports
```

## Step 5: Create Install Configuration

### 5.1 Generate SSH Key (if needed)

**Note**: RHEL 9 includes OpenSSH 8.7+ which fully supports ed25519 keys (recommended for OpenShift).

```bash
# Generate ed25519 SSH key (recommended for OpenShift 4.20)
ssh-keygen -t ed25519 -N '' -f ~/.ssh/id_ed25519 -C "ocp-cluster-key"

# Verify key was created
ls -la ~/.ssh/id_ed25519*

# Set proper permissions (important for RHEL 9 security)
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub
```

### 5.2 Create install-config.yaml

**Important**: The IPI installer creates the initial control plane and a default worker MachineSet. For PAS-X workload clusters, we will:
1. Install with **0 worker replicas** initially
2. Create custom MachineSets post-installation (Step 8) for specialized workload nodes

For other deployment types, set `compute.replicas` to 3 or more.

```bash
mkdir ~/ocp-install
cd ~/ocp-install

# Choose configuration based on cluster type:

# ============================================================================
# OPTION 1: PAS-X Workload Cluster (Manufacturing Site)
# ============================================================================
cat > install-config.yaml <<EOF
apiVersion: v1
baseDomain: example.com  # Update to your domain
compute:
- architecture: amd64
  hyperthreading: Enabled
  name: worker
  platform:
    nutanix:
      cpus: 4
      coresPerSocket: 2
      memoryMiB: 16384
      osDisk:
        diskSizeGiB: 120
  replicas: 0  # IMPORTANT: Set to 0 - we create custom MachineSets in Step 8
controlPlane:
  architecture: amd64
  hyperthreading: Enabled
  name: master
  platform:
    nutanix:
      cpus: 4
      coresPerSocket: 2
      memoryMiB: 16384  # 16 GB RAM per control plane node
      osDisk:
        diskSizeGiB: 120
  replicas: 3
EOF

# ============================================================================
# OPTION 2: ACM Hub Cluster (Management)
# ============================================================================
# cat > install-config.yaml <<EOF
# apiVersion: v1
# baseDomain: example.com
# compute:
# - architecture: amd64
#   hyperthreading: Enabled
#   name: worker
#   platform:
#     nutanix:
#       cpus: 4
#       coresPerSocket: 2
#       memoryMiB: 16384
#       osDisk:
#         diskSizeGiB: 120
#   replicas: 0  # No standard workers - use Infrastructure MachineSets
# controlPlane:
#   architecture: amd64
#   hyperthreading: Enabled
#   name: master
#   platform:
#     nutanix:
#       cpus: 4
#       coresPerSocket: 2
#       memoryMiB: 16384
#       osDisk:
#         diskSizeGiB: 120
#   replicas: 3
# EOF

# ============================================================================
# OPTION 3: Generic Production Cluster
# ============================================================================
# cat > install-config.yaml <<EOF
# apiVersion: v1
# baseDomain: example.com
# compute:
# - architecture: amd64
#   hyperthreading: Enabled
#   name: worker
#   platform:
#     nutanix:
#       cpus: 8
#       coresPerSocket: 2
#       memoryMiB: 32768
#       osDisk:
#         diskSizeGiB: 120
#   replicas: 3
# controlPlane:
#   architecture: amd64
#   hyperthreading: Enabled
#   name: master
#   platform:
#     nutanix:
#       cpus: 8
#       coresPerSocket: 2
#       memoryMiB: 32768
#       osDisk:
#         diskSizeGiB: 120
#   replicas: 3
# EOF

# ============================================================================
# Continue with common configuration (applies to all options)
# ============================================================================
metadata:
  name: ocp-cluster
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineNetwork:
  - cidr: 192.168.1.0/24  # Your Nutanix network CIDR
  networkType: OVNKubernetes
  serviceNetwork:
  - 172.30.0.0/16
platform:
  nutanix:
    apiVIPs:
    - 192.168.1.100  # API VIP - must be available
    ingressVIPs:
    - 192.168.1.101  # Ingress VIP - must be available
    prismCentral:
      endpoint:
        address: prism-central.example.com
        port: 9440
      password: password
      username: admin
    prismElements:
    - endpoint:
        address: prism-element.example.com
        port: 9440
      uuid: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx  # PE cluster UUID
    subnetUUIDs:
    - xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx  # Subnet UUID
publish: External
pullSecret: '$(cat /tmp/ocp-binaries/pull-secret-mirror.json | jq -c .)'
sshKey: '$(cat ~/.ssh/id_ed25519.pub)'
additionalTrustBundle: |
$(sed 's/^/  /' /etc/pki/ca-trust/source/anchors/quay-rootCA.pem)
imageContentSources:
- mirrors:
  - $(hostname):8443/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-release
- mirrors:
  - $(hostname):8443/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
EOF
```

**Important Notes:**
- Get the Prism Element UUID and Subnet UUID from Nutanix:
  - PE UUID: Prism Central > Compute & Storage > Clusters > <cluster> > Details
  - Subnet UUID: Prism Central > Network & Security > Subnets > <subnet> > Details
- If using **manual registry** (port 5000), change:
  - `additionalTrustBundle` certificate path to `/etc/pki/ca-trust/source/anchors/domain.crt`
  - `imageContentSources` mirrors to `$(hostname):5000/ocp4/openshift4`

## Step 6: Install OpenShift

### 6.1 Backup Configuration

```bash
# IMPORTANT: Back up install-config.yaml (it gets consumed)
cp install-config.yaml install-config.yaml.bak
```

### 6.2 Create Ignition Configs

```bash
openshift-install create ignition-configs --dir ~/ocp-install
```

### 6.3 Deploy Cluster

```bash
# This will take 30-45 minutes
openshift-install create cluster --dir ~/ocp-install --log-level=info
```

The installer will:
1. Create bootstrap VM on Nutanix
2. Create 3 control plane VMs
3. Create 3 worker VMs
4. Bootstrap the cluster
5. Destroy the bootstrap node
6. Complete installation

## Step 7: Verify Cluster Installation

### 7.1 Access Cluster

```bash
export KUBECONFIG=~/ocp-install/auth/kubeconfig
oc whoami

# Get console URL and kubeadmin password
cat ~/ocp-install/auth/kubeadmin-password
oc whoami --show-console

# Verify cluster operators are running
oc get clusteroperators

# Check nodes (should show 3 control plane nodes only for PAS-X workload config)
oc get nodes
```

## Step 8: Create Custom MachineSets (PAS-X Workload Clusters)

**Note**: Skip this step if you installed with worker replicas > 0 (generic production cluster). This step is **required** for PAS-X workload clusters where we set `compute.replicas: 0` in Step 5.2.

### 8.1 Get Existing MachineSet Template

First, we'll get the infrastructure details from the cluster to use in our custom MachineSets:

```bash
# Get cluster infrastructure name (used in MachineSet names)
export CLUSTER_NAME=$(oc get infrastructure cluster -o jsonpath='{.status.infrastructureName}')
echo "Cluster infrastructure name: ${CLUSTER_NAME}"

# Get Nutanix cluster UUID
export NUTANIX_CLUSTER_UUID=$(oc get machines -n openshift-machine-api -o jsonpath='{.items[0].spec.providerSpec.value.cluster.uuid}')
echo "Nutanix cluster UUID: ${NUTANIX_CLUSTER_UUID}"

# Get Nutanix subnet UUID
export NUTANIX_SUBNET_UUID=$(oc get machines -n openshift-machine-api -o jsonpath='{.items[0].spec.providerSpec.value.subnets[0].uuid}')
echo "Nutanix subnet UUID: ${NUTANIX_SUBNET_UUID}"

# Get Nutanix image name
export NUTANIX_IMAGE=$(oc get machines -n openshift-machine-api -o jsonpath='{.items[0].spec.providerSpec.value.image.name}')
echo "Nutanix image: ${NUTANIX_IMAGE}"
```

### 8.2 Create PAS-X Central MachineSet (2 nodes, 20 vCPU, 32 GB RAM)

```bash
cat > machineset-pasx-central.yaml <<EOF
apiVersion: machine.openshift.io/v1beta1
kind: MachineSet
metadata:
  name: ${CLUSTER_NAME}-pasx-central
  namespace: openshift-machine-api
  labels:
    machine.openshift.io/cluster-api-cluster: ${CLUSTER_NAME}
spec:
  replicas: 2
  selector:
    matchLabels:
      machine.openshift.io/cluster-api-cluster: ${CLUSTER_NAME}
      machine.openshift.io/cluster-api-machineset: ${CLUSTER_NAME}-pasx-central
  template:
    metadata:
      labels:
        machine.openshift.io/cluster-api-cluster: ${CLUSTER_NAME}
        machine.openshift.io/cluster-api-machine-role: worker
        machine.openshift.io/cluster-api-machine-type: worker
        machine.openshift.io/cluster-api-machineset: ${CLUSTER_NAME}-pasx-central
    spec:
      metadata:
        labels:
          node-role.kubernetes.io/pasx-central: ""
      taints:
      - effect: NoSchedule
        key: pasx-central
        value: "true"
      providerSpec:
        value:
          apiVersion: machine.openshift.io/v1
          kind: NutanixMachineProviderConfig
          cluster:
            type: uuid
            uuid: ${NUTANIX_CLUSTER_UUID}
          image:
            type: name
            name: ${NUTANIX_IMAGE}
          subnets:
          - type: uuid
            uuid: ${NUTANIX_SUBNET_UUID}
          vcpuSockets: 10
          vcpusPerSocket: 2
          memorySize: 32Gi
          systemDiskSize: 120Gi
          userDataSecret:
            name: worker-user-data
EOF

oc apply -f machineset-pasx-central.yaml
```

### 8.3 Create Percona PostgreSQL MachineSet (3 nodes, 8 vCPU, 24 GB RAM)

```bash
cat > machineset-percona-pg.yaml <<EOF
apiVersion: machine.openshift.io/v1beta1
kind: MachineSet
metadata:
  name: ${CLUSTER_NAME}-percona-pg
  namespace: openshift-machine-api
  labels:
    machine.openshift.io/cluster-api-cluster: ${CLUSTER_NAME}
spec:
  replicas: 3
  selector:
    matchLabels:
      machine.openshift.io/cluster-api-cluster: ${CLUSTER_NAME}
      machine.openshift.io/cluster-api-machineset: ${CLUSTER_NAME}-percona-pg
  template:
    metadata:
      labels:
        machine.openshift.io/cluster-api-cluster: ${CLUSTER_NAME}
        machine.openshift.io/cluster-api-machine-role: worker
        machine.openshift.io/cluster-api-machine-type: worker
        machine.openshift.io/cluster-api-machineset: ${CLUSTER_NAME}-percona-pg
    spec:
      metadata:
        labels:
          node-role.kubernetes.io/database: ""
      taints:
      - effect: NoSchedule
        key: database
        value: "true"
      providerSpec:
        value:
          apiVersion: machine.openshift.io/v1
          kind: NutanixMachineProviderConfig
          cluster:
            type: uuid
            uuid: ${NUTANIX_CLUSTER_UUID}
          image:
            type: name
            name: ${NUTANIX_IMAGE}
          subnets:
          - type: uuid
            uuid: ${NUTANIX_SUBNET_UUID}
          vcpuSockets: 4
          vcpusPerSocket: 2
          memorySize: 24Gi
          systemDiskSize: 120Gi
          userDataSecret:
            name: worker-user-data
EOF

oc apply -f machineset-percona-pg.yaml
```

### 8.4 Create PAS-X PDA MachineSet (2 nodes, 8 vCPU, 24 GB RAM)

```bash
cat > machineset-pasx-pda.yaml <<EOF
apiVersion: machine.openshift.io/v1beta1
kind: MachineSet
metadata:
  name: ${CLUSTER_NAME}-pasx-pda
  namespace: openshift-machine-api
  labels:
    machine.openshift.io/cluster-api-cluster: ${CLUSTER_NAME}
spec:
  replicas: 2
  selector:
    matchLabels:
      machine.openshift.io/cluster-api-cluster: ${CLUSTER_NAME}
      machine.openshift.io/cluster-api-machineset: ${CLUSTER_NAME}-pasx-pda
  template:
    metadata:
      labels:
        machine.openshift.io/cluster-api-cluster: ${CLUSTER_NAME}
        machine.openshift.io/cluster-api-machine-role: worker
        machine.openshift.io/cluster-api-machine-type: worker
        machine.openshift.io/cluster-api-machineset: ${CLUSTER_NAME}-pasx-pda
    spec:
      metadata:
        labels:
          node-role.kubernetes.io/pasx-pda: ""
      taints:
      - effect: NoSchedule
        key: pasx-pda
        value: "true"
      providerSpec:
        value:
          apiVersion: machine.openshift.io/v1
          kind: NutanixMachineProviderConfig
          cluster:
            type: uuid
            uuid: ${NUTANIX_CLUSTER_UUID}
          image:
            type: name
            name: ${NUTANIX_IMAGE}
          subnets:
          - type: uuid
            uuid: ${NUTANIX_SUBNET_UUID}
          vcpuSockets: 4
          vcpusPerSocket: 2
          memorySize: 24Gi
          systemDiskSize: 120Gi
          userDataSecret:
            name: worker-user-data
EOF

oc apply -f machineset-pasx-pda.yaml
```

### 8.5 Create PAS-X Misc MachineSet (4 nodes, 8 vCPU, 32 GB RAM)

```bash
cat > machineset-pasx-misc.yaml <<EOF
apiVersion: machine.openshift.io/v1beta1
kind: MachineSet
metadata:
  name: ${CLUSTER_NAME}-pasx-misc
  namespace: openshift-machine-api
  labels:
    machine.openshift.io/cluster-api-cluster: ${CLUSTER_NAME}
spec:
  replicas: 4
  selector:
    matchLabels:
      machine.openshift.io/cluster-api-cluster: ${CLUSTER_NAME}
      machine.openshift.io/cluster-api-machineset: ${CLUSTER_NAME}-pasx-misc
  template:
    metadata:
      labels:
        machine.openshift.io/cluster-api-cluster: ${CLUSTER_NAME}
        machine.openshift.io/cluster-api-machine-role: worker
        machine.openshift.io/cluster-api-machine-type: worker
        machine.openshift.io/cluster-api-machineset: ${CLUSTER_NAME}-pasx-misc
    spec:
      metadata:
        labels:
          node-role.kubernetes.io/pasx-misc: ""
      providerSpec:
        value:
          apiVersion: machine.openshift.io/v1
          kind: NutanixMachineProviderConfig
          cluster:
            type: uuid
            uuid: ${NUTANIX_CLUSTER_UUID}
          image:
            type: name
            name: ${NUTANIX_IMAGE}
          subnets:
          - type: uuid
            uuid: ${NUTANIX_SUBNET_UUID}
          vcpuSockets: 4
          vcpusPerSocket: 2
          memorySize: 32Gi
          systemDiskSize: 120Gi
          userDataSecret:
            name: worker-user-data
EOF

oc apply -f machineset-pasx-misc.yaml
```

### 8.6 Verify MachineSets and Nodes

```bash
# Watch MachineSets
oc get machinesets -n openshift-machine-api

# Watch Machines being created
oc get machines -n openshift-machine-api -w

# Watch nodes joining the cluster (Ctrl+C to exit)
oc get nodes -w

# Verify all 14 worker nodes are Ready
oc get nodes

# Expected output:
# - 3 control plane nodes (master)
# - 2 pasx-central nodes
# - 3 percona-pg nodes (database)
# - 2 pasx-pda nodes
# - 4 pasx-misc nodes
# Total: 17 nodes (3 control + 14 workers)
```

### 8.7 Understanding Taints and Tolerations

The custom MachineSets use **taints** to ensure only specific PAS-X workloads run on dedicated nodes:

- **PAS-X Central nodes**: Tainted with `pasx-central=true:NoSchedule`
- **PostgreSQL nodes**: Tainted with `database=true:NoSchedule`
- **PAS-X PDA nodes**: Tainted with `pasx-pda=true:NoSchedule`
- **PAS-X Misc nodes**: No taints (general PAS-X workloads)

To schedule pods on tainted nodes, add tolerations to your pod specs:

```yaml
tolerations:
- key: "pasx-central"
  operator: "Equal"
  value: "true"
  effect: "NoSchedule"
```

## Step 9: Post-Installation Configuration

### 9.1 Configure OperatorHub (if mirrored)

```bash
# Disable default catalogs
oc patch OperatorHub cluster --type json \
  -p '[{"op": "add", "path": "/spec/disableAllDefaultSources", "value": true}]'

# Apply mirrored catalog sources (from Step 2.4 output)
oc apply -f catalogsource-redhat-operators.yaml
oc apply -f catalogsource-certified-operators.yaml
```

### 7.2 Configure OperatorHub (if mirrored)

```bash
# Disable default catalogs
oc patch OperatorHub cluster --type json \
  -p '[{"op": "add", "path": "/spec/disableAllDefaultSources", "value": true}]'

# Apply mirrored catalog sources (from Step 2.4 output)
oc apply -f catalogsource-redhat-operators.yaml
oc apply -f catalogsource-certified-operators.yaml
```

### 9.2 Configure Storage (PAS-X Multi-Tier Strategy)

**PAS-X architecture requires three storage tiers:**
1. **Block storage** (Nutanix CSI) - High-IOPS for databases
2. **File storage** (NFS/SMB) - Shared application files
3. **Object/Archive** (Data Domain/NetApp) - Long-term storage

#### 9.2.1 Verify Nutanix CSI Driver

```bash
# Check if Nutanix CSI driver is installed (usually automatic with IPI)
oc get storageclass

# Expected output should include Nutanix storage classes
# If not present, install Nutanix CSI Operator from OperatorHub
```

#### 9.2.2 Create StorageClasses

```bash
# High-performance block storage for PostgreSQL (RWO)
cat > storageclass-nutanix-block.yaml <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nutanix-volume
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: csi.nutanix.com
parameters:
  csi.storage.k8s.io/provisioner-secret-name: ntnx-secret
  csi.storage.k8s.io/provisioner-secret-namespace: openshift-cluster-csi-drivers
  csi.storage.k8s.io/node-publish-secret-name: ntnx-secret
  csi.storage.k8s.io/node-publish-secret-namespace: openshift-cluster-csi-drivers
  csi.storage.k8s.io/controller-expand-secret-name: ntnx-secret
  csi.storage.k8s.io/controller-expand-secret-namespace: openshift-cluster-csi-drivers
  storageType: NutanixVolumes
  fsType: ext4
allowVolumeExpansion: true
reclaimPolicy: Delete
volumeBindingMode: Immediate
EOF

oc apply -f storageclass-nutanix-block.yaml
```

#### 9.2.3 Configure Windows NFS/SMB Provisioner

**Prerequisites**: Windows Server 2022 NFS/SMB file server configured and accessible from cluster.

```bash
# Install NFS Subdir External Provisioner from OperatorHub
# OR use CSI driver for SMB

# Example: Manual NFS StorageClass (if not using operator)
cat > storageclass-nfs.yaml <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-pasx-shared
provisioner: nfs.csi.k8s.io
parameters:
  server: windows-nfs-server.example.com  # Your Windows NFS server
  share: /pasx-shared
  mountPermissions: "0755"
volumeBindingMode: Immediate
reclaimPolicy: Retain
EOF

oc apply -f storageclass-nfs.yaml
```

#### 9.2.4 Configure Internal Image Registry

```bash
# Create PVC for image registry storage using Nutanix block storage
oc create -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: image-registry-storage
  namespace: openshift-image-registry
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 500Gi
  storageClassName: nutanix-volume
EOF

# Configure registry to use PVC
oc patch configs.imageregistry.operator.openshift.io cluster --type merge \
  -p '{"spec":{"storage":{"pvc":{"claim":"image-registry-storage"}},"managementState":"Managed"}}'
```

### 9.3 Configure Active Directory Integration

```bash
# Create LDAP bind secret
oc create secret generic ldap-bind-password \
  --from-literal=bindPassword='<your-ad-bind-password>' \
  -n openshift-config

# Create LDAP identity provider
cat > ldap-oauth.yaml <<EOF
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - name: active_directory
    type: LDAP
    mappingMethod: claim
    ldap:
      attributes:
        id:
        - sAMAccountName
        email:
        - mail
        name:
        - displayName
        preferredUsername:
        - sAMAccountName
      bindDN: "CN=ocp-bind,OU=ServiceAccounts,DC=example,DC=com"
      bindPassword:
        name: ldap-bind-password
      insecure: false
      ca:
        name: ad-ca-cert
      url: "ldaps://ad-server.example.com:636/DC=example,DC=com?sAMAccountName"
EOF

oc apply -f ldap-oauth.yaml

# Sync AD groups for RBAC
cat > ldap-group-sync.yaml <<EOF
kind: LDAPSyncConfig
apiVersion: v1
url: ldaps://ad-server.example.com:636
bindDN: "CN=ocp-bind,OU=ServiceAccounts,DC=example,DC=com"
bindPassword: <your-bind-password>
insecure: false
groupUIDNameMapping:
  "CN=OCP-Admins,OU=Groups,DC=example,DC=com": ocp-admins
  "CN=OCP-Readers,OU=Groups,DC=example,DC=com": ocp-readers
rfc2307:
  groupsQuery:
    baseDN: "OU=Groups,DC=example,DC=com"
    scope: sub
    filter: "(objectClass=group)"
    derefAliases: never
  groupUIDAttribute: dn
  groupNameAttributes: [ cn ]
  groupMembershipAttributes: [ member ]
  usersQuery:
    baseDN: "OU=Users,DC=example,DC=com"
    scope: sub
    derefAliases: never
  userUIDAttribute: dn
  userNameAttributes: [ sAMAccountName ]
EOF

# Sync groups
oc adm groups sync --sync-config=ldap-group-sync.yaml --confirm
```

### 9.4 Configure RBAC for Application Teams

```bash
# Grant cluster-admin to platform engineers
oc adm policy add-cluster-role-to-group cluster-admin ocp-admins

# Grant cluster-reader for audit/read-only access
oc adm policy add-cluster-role-to-group cluster-reader ocp-readers

# Create namespace-scoped admin role for PAS-X team (example)
oc create namespace pasx-production
oc adm policy add-role-to-group admin pasx-team -n pasx-production
```

## Key Considerations and Requirements

### DNS Requirements

DNS must be configured before installation:
- `api.<cluster-name>.<base-domain>` → API VIP (static IP recommended)
- `api-int.<cluster-name>.<base-domain>` → API VIP (static IP recommended)
- `*.apps.<cluster-name>.<base-domain>` → Ingress VIP (static IP recommended)

**Best Practice**: Use **static IPs** for all infrastructure components to reduce firewall complexity across distributed sites.

### Network Requirements and Firewall Rules

#### Core OpenShift Connectivity

| Source | Destination | Port/Protocol | Purpose |
|--------|-------------|---------------|---------|
| Jumphost / Admin workstations | OCP API (api VIP) | 6443/TCP | Kubernetes API access |
| End users | OCP Ingress (*.apps VIP) | 443/TCP (HTTPS) | Application access |
| End users | OCP Ingress (*.apps VIP) | 80/TCP (HTTP) | HTTP redirect to HTTPS |
| OpenShift nodes | Local mirror registry | 8443/TCP (HTTPS) | Pull container images |
| OpenShift nodes | Nutanix Prism Central | 9440/TCP (HTTPS) | IPI provisioning, CSI operations |
| OpenShift nodes | DNS servers | 53/UDP | Name resolution |
| OpenShift nodes | NTP servers | 123/UDP | Time synchronization |

#### PAS-X Application-Specific Connectivity

| Source | Destination | Port/Protocol | Purpose |
|--------|-------------|---------------|---------|
| **PAS-X Clients** (Windows Session Hosts) | OCP Ingress | 443/TCP (HTTPS) | PAS-X Web UI and Classic UI access |
| **Shop Floor Automation** (DCS systems) | MSI Service/Adapter pods | 502/TCP (Vendor-specific) | Equipment and automation integration |
| **PAS-X Pods** | Windows NFS Server | 2049/TCP (NFS), 111/TCP (RPC) | Mount shared file PVCs |
| **PAS-X Printing Service** | CUPS Print Server | 9100/TCP, 631/TCP (IPP) | PDF reports and label printing |
| **Monitoring/Observability** | PAS-X microservices | 8087/TCP | Spring Actuator Prometheus metrics |
| **ACM Hub** | Managed cluster API | 6443/TCP | Multi-cluster management |
| **Managed cluster** | ACM Hub ingress | 443/TCP | Cluster registration, observability |

#### Internal OpenShift Networks (Non-Routed)

- **Service Network**: 172.30.0.0/16 (default) - Internal service communication
- **Cluster Network**: 10.128.0.0/14 (default) - Pod-to-pod communication

**Important**: These internal networks are **not routed** outside the cluster and do not require firewall rules.

### Network Isolation and Air-Gap Requirements

**Critical for GxP Compliance:**
- Clusters have **NO direct internet access**
- All container images pulled from **local mirror registry only**
- All traffic between components must be **encrypted** (TLS/HTTPS)
- Static IPs used for infrastructure to simplify firewall rules

### Regulatory Compliance (GxP)

**This architecture supports FDA 21 CFR Part 11, EU Annex 11, and GAMP5 compliance requirements for regulated industries:**

#### Validation Requirements

**Installation Qualification (IQ)**:
- Automated cluster build process generates evidence
- All configurations tracked in Git (GitOps)
- Infrastructure-as-Code ensures reproducible deployments

**Operational Qualification (OQ)**:
- Cluster operators monitored for health
- All Day-2 operations performed via GitOps (auditable)
- Change control enforced through Git commit history

**Performance Qualification (PQ)**:
- Application-level testing (PAS-X MES)
- Load testing for concurrent user capacity (200+ users per site)
- Batch processing validation (hundreds of concurrent batches)

#### Data Encryption

```bash
# Enable etcd encryption at rest
oc patch etcd cluster --type=merge -p '{"spec":{"encryption":{"type":"aescbc"}}}'

# Verify encryption status
oc get openshiftapiserver -o=jsonpath='{range .items[0].status.conditions[?(@.type=="Encrypted")]}{.reason}{"\n"}{.message}{"\n"}'
```

**Data in Transit**:
- All OpenShift API communication uses TLS
- All ingress routes use HTTPS (TLS termination at router)
- PAS-X pods communicate internally via encrypted channels
- External integrations (AD, NFS, databases) use TLS/encrypted protocols

#### Audit Logging

```bash
# Configure audit log forwarding (requires cluster-logging operator)
# Audit logs stored for regulatory compliance periods (typically 7+ years)

# View audit logs
oc adm node-logs --role=master --path=kube-apiserver/
oc adm node-logs --role=master --path=openshift-apiserver/
```

#### Change Control via GitOps

All cluster configurations must be:
1. Defined in YAML manifests
2. Committed to Git repository
3. Reviewed and approved via pull request
4. Applied via ArgoCD/OpenShift GitOps
5. Traceable to specific commit SHA

**Example GitOps workflow**:
```bash
# All changes tracked in git
git log --oneline playbooks/deploy-autoshift-stack.yaml

# Each commit provides audit trail
git show <commit-sha>
```

### Disaster Recovery and Backup Strategy

**PAS-X Architecture DR Breakdown:**

| Component | Storage Type | Backup Method | Recovery Method | Criticality |
|-----------|--------------|---------------|-----------------|-------------|
| **OCP etcd** (Cluster State) | Block (Nutanix) | Daily automated etcd backups | Rebuild from IaC/GitOps + restore etcd | Low/Moderate |
| **PAS-X Database** (Percona PG) | Block (Nutanix) | Continuous backup to NetApp/Data Domain | Database restore from backup | **CRITICAL** |
| **RabbitMQ** (Message Bus) | Block (Nutanix) | None (ephemeral) | Automatic regeneration from database state | Zero Impact |
| **Shared Files** (PDFs/Reports) | File (NFS/SMB) | Windows Server backups to NetApp | Restore from file server backup | Moderate |
| **Container Images** | Mirror Registry | Registry data snapshots | Restore registry data | Low (can re-mirror) |

#### Critical Backup Configurations

**etcd Backup (Automated Daily)**:
```bash
# Install etcd-backup CronJob
cat > etcd-backup-cronjob.yaml <<EOF
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-backup
  namespace: openshift-etcd
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: etcd-backup
            image: quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:...
            command:
            - /bin/bash
            - -c
            - |
              /usr/local/bin/cluster-backup.sh /backup
              # Copy to external storage (NetApp/Data Domain)
              rsync -av /backup/ backup-server:/etcd-backups/
            volumeMounts:
            - name: backup
              mountPath: /backup
          volumes:
          - name: backup
            persistentVolumeClaim:
              claimName: etcd-backup-pvc
          restartPolicy: OnFailure
          nodeSelector:
            node-role.kubernetes.io/master: ""
          tolerations:
          - operator: Exists
EOF

oc apply -f etcd-backup-cronjob.yaml
```

**PAS-X PostgreSQL Backup** (handled by Percona Operator):
```bash
# Percona PostgreSQL Operator provides automated backups
# Configure backup schedule and retention in PerconaXtraDBCluster CR

# Example backup configuration
cat > percona-backup-config.yaml <<EOF
apiVersion: pgv2.percona.com/v2
kind: PerconaPGCluster
metadata:
  name: pasx-postgres
spec:
  backups:
    pgbackrest:
      repos:
      - name: repo1
        schedules:
          full: "0 1 * * 0"       # Weekly full backup
          differential: "0 1 * * 1-6"  # Daily differential
          incremental: "0 */4 * * *"   # Every 4 hours
        volume:
          volumeClaimSpec:
            accessModes:
            - ReadWriteOnce
            resources:
              requests:
                storage: 500Gi
            storageClassName: nutanix-volume
EOF
```

**Disaster Recovery Runbook** (separate document recommended):
1. Site-level failure: Cluster isolated to single manufacturing site
2. Rebuild cluster using IaC automation (`deploy-autoshift-stack.yaml`)
3. Restore etcd from backup (if needed for speed)
4. Restore PAS-X PostgreSQL database from NetApp/Data Domain
5. Apply application configurations via GitOps
6. Validate system using PQ test scripts

**Recovery Time Objective (RTO)**: 
- Cluster rebuild: 2-4 hours
- Database restore: 1-2 hours (depends on backup size)
- Total RTO: ~4-6 hours

**Recovery Point Objective (RPO)**:
- Database: Continuous backup (RPO < 15 minutes)
- Cluster configuration: Git commits (RPO ~0, no data loss)

### Troubleshooting

```bash
# Watch bootstrap progress
openshift-install wait-for bootstrap-complete --dir ~/ocp-install --log-level=info

# SSH to bootstrap node if needed
ssh -i ~/.ssh/id_ed25519 core@<bootstrap-ip>
journalctl -b -f -u bootkube.service

# Watch cluster operators
oc get clusteroperators
oc get co <operator-name> -o yaml

# Check installation logs
tail -f ~/ocp-install/.openshift_install.log
```

## Common Issues

### Certificate Trust Issues

If you see certificate errors:

**For mirror-registry:**
```bash
# On jumphost
sudo cp /opt/quay/quay-rootCA/rootCA.pem /etc/pki/ca-trust/source/anchors/quay-rootCA.pem
sudo update-ca-trust

# Verify with podman login
podman login -u init -p passw0rd $(hostname):8443

# OR verify with curl
curl -k -u init:passw0rd https://$(hostname):8443/v2/_catalog
```

**For manual registry:**
```bash
# On jumphost
sudo cp /opt/registry/certs/domain.crt /etc/pki/ca-trust/source/anchors/
sudo update-ca-trust

# Verify
curl -u admin:passw0rd https://$(hostname):5000/v2/_catalog
```

### Registry Authentication Failures

Verify pull secret contains local registry credentials:
```bash
cat /tmp/ocp-binaries/pull-secret-mirror.json | jq .
```

### Nutanix API Connection Issues

Test Prism Central connectivity:
```bash
curl -k https://prism-central.example.com:9440/api/nutanix/v3/clusters/list \
  -u admin:password \
  -H "Content-Type: application/json" \
  -d '{"kind":"cluster"}'
```

### Installation Timeout

If installation times out, check:
- DNS resolution for API and apps domains
- VIP addresses are not in use
- DHCP pool has enough addresses for all nodes
- Prism Element has sufficient resources

## Cluster Sizing and Resource Requirements

### Architecture Overview

This guide provisions **workload clusters** for manufacturing sites. Each site requires:
- 1 OpenShift cluster with custom MachineSets for PAS-X MES workloads
- Local mirrored container registry
- Integration with site-specific infrastructure (Windows NFS/SMB, Active Directory, CUPS printing)

### Cluster Sizing Options

Choose the appropriate sizing based on your deployment type:

#### Option 1: PAS-X Workload Cluster (Production Manufacturing Site)

**Initial IPI Installation** (Step 6):
- **Control Plane**: 3 nodes × 4 vCPU × 16GB RAM × 120GB disk
- **Bootstrap** (temporary): 1 node × 4 vCPU × 16GB RAM × 120GB disk

**Post-Install Custom MachineSets** (Step 8 - created after cluster provisioning):
- **PAS-X Central**: 2 nodes × 20 vCPU × 32GB RAM × 120GB disk (Tainted)
- **Percona PostgreSQL**: 3 nodes × 8 vCPU × 24GB RAM × 120GB disk (Tainted)
- **PAS-X PDA**: 2 nodes × 8 vCPU × 24GB RAM × 120GB disk (Tainted)
- **PAS-X Misc**: 4 nodes × 8 vCPU × 32GB RAM × 120GB disk

**Total per Workload Cluster:**
- **Nodes**: 14 worker nodes + 3 control plane = 17 nodes
- **vCPU**: 124 vCPU (workers) + 12 vCPU (control plane) = 136 vCPU
- **RAM**: 360 GB (workers) + 48 GB (control plane) = 408 GB

#### Option 2: ACM Hub Cluster (Management Hub)

**Note**: See `deploy-autoshift-stack.yaml` playbook for automated ACM hub deployment.

**Initial IPI Installation**:
- **Control Plane**: 3 nodes × 4 vCPU × 16GB RAM × 120GB disk
- **Bootstrap** (temporary): 1 node × 4 vCPU × 16GB RAM × 120GB disk

**Post-Install Infrastructure MachineSets** (no standard worker nodes):
- **Infrastructure Nodes**: 3+ nodes × 8 vCPU × 32GB RAM × 120GB disk
  - Purpose: ACM operators, Multi-Cluster Observability (MCO), ingress controllers
  - Benefit: **Do NOT consume OpenShift worker subscriptions**
  - Label: `node-role.kubernetes.io/infra`

**Total per ACM Hub:**
- **Nodes**: 3 infrastructure nodes + 3 control plane = 6 nodes
- **vCPU**: 24 vCPU (infra) + 12 vCPU (control plane) = 36 vCPU
- **RAM**: 96 GB (infra) + 48 GB (control plane) = 144 GB

#### Option 3: Generic Production Cluster (General Purpose)

For non-PAS-X production workloads:
- **Control Plane**: 3 nodes × 8 vCPU × 32GB RAM × 120GB disk
- **Worker Nodes**: 3+ nodes × 8 vCPU × 32GB RAM × 120GB disk
- **Bootstrap** (temporary): 1 node × 4 vCPU × 16GB RAM × 120GB disk

### Jumphost Storage Requirements

- Base OS: 50GB
- **Container registry mirror**: 400-500GB
  - OpenShift 4.20 release images: ~12 GB
  - Red Hat Operator catalog (full): ~358 GB
  - PAS-X application images: ~50-100 GB
- Temporary files and workspace: 50GB
- **Total recommended**: **600GB minimum**

**Note**: The mirror registry stores all container images required for air-gapped operation, including OpenShift platform components, Red Hat certified operators, and application images.

## Next Steps After Cluster Installation

### For PAS-X Workload Clusters

After completing Steps 1-9, the cluster is ready for PAS-X MES application deployment:

1. **Register with ACM Hub** (if not already done)
   ```bash
   # Import cluster to ACM hub for centralized management
   # This is typically done from the ACM hub console or via GitOps
   ```

2. **Deploy PAS-X Application Stack**
   - Percona PostgreSQL Operator
   - RabbitMQ Operator
   - PAS-X Central pods (scheduled on pasx-central nodes with tolerations)
   - PAS-X PDA pods (scheduled on pasx-pda nodes with tolerations)
   - PAS-X supporting services (scheduled on pasx-misc nodes)

3. **Configure PAS-X Storage**
   - PostgreSQL databases: Nutanix block storage (12Gi RWO PVCs)
   - RabbitMQ message queues: Nutanix block storage (12Gi RWO PVCs)
   - Shared files (PDFs, labels, reports): Windows NFS/SMB (15Gi RWX PVCs)

4. **Configure PAS-X Networking**
   - Create Routes for PAS-X Web UI and Classic UI
   - Configure ingress for DCS/shop floor automation (MSI adapter)
   - Set up network policies for pod-to-pod communication

5. **Integration with External Systems**
   - Windows Session Hosts (RDP access for PAS-X clients)
   - CUPS print servers (label and report printing)
   - Shop floor equipment (DCS integration via MSI adapter)

6. **Enable Monitoring and Observability**
   ```bash
   # PAS-X microservices expose Prometheus metrics on port 8087
   # Configure ServiceMonitor for automatic scraping
   ```

7. **Perform Validation Testing**
   - Installation Qualification (IQ): Verify all components installed correctly
   - Operational Qualification (OQ): Test Day-2 operations (backup, restore, failover)
   - Performance Qualification (PQ): Load testing (200+ concurrent users, hundreds of batches)

### For ACM Hub Cluster

1. **Deploy ACM Operators**
   ```bash
   # Use deploy-autoshift-stack.yaml playbook or manual installation
   # Install Advanced Cluster Management operator
   # Install MultiCluster Observability operator
   ```

2. **Create Infrastructure MachineSets**
   - Follow Step 8 pattern but create infrastructure nodes instead of PAS-X nodes
   - Label nodes with `node-role.kubernetes.io/infra`
   - Configure ACM/MCO pods to schedule on infrastructure nodes

3. **Import Managed Clusters**
   - Import 5 workload clusters from manufacturing sites
   - Organize clusters into ClusterSets (e.g., "pasx-production")
   - Apply cluster labels for targeting policies

4. **Configure Multi-Cluster Observability**
   - Deploy MCO with object storage backend
   - Configure retention policies for metrics
   - Create Grafana dashboards for fleet-wide visibility

5. **Implement GitOps for Fleet Management**
   - Deploy OpenShift GitOps (ArgoCD)
   - Create ApplicationSets for deploying to multiple clusters
   - Implement policy-as-code for compliance enforcement

### General Post-Installation Tasks

1. **Configure Backup Strategy**
   - Set up daily etcd backups (see DR section)
   - Configure PostgreSQL continuous backups to NetApp/Data Domain
   - Test restore procedures

2. **Implement Monitoring Alerting**
   - Configure AlertManager for critical alerts
   - Integrate with existing monitoring systems (if any)
   - Set up PagerDuty/email notifications for production issues

3. **Document Cluster Configuration**
   - Record all IPs, VIPs, and DNS entries
   - Document AD/LDAP group mappings
   - Create runbooks for common operations
   - Generate IQ evidence documentation for regulatory compliance

4. **Security Hardening** (if not already done)
   - Enable etcd encryption at rest
   - Configure network policies to restrict pod-to-pod communication
   - Review and harden RBAC policies
   - Enable audit logging forwarding

5. **Capacity Planning**
   - Monitor resource utilization across nodes
   - Plan for scaling MachineSets as workload grows
   - Set resource quotas and limit ranges per namespace

## RHEL 9 Specific Considerations

### Package Management
- RHEL 9 uses **dnf** as the default package manager (not yum)
- Most packages are available in `baseos` and `appstream` repositories
- Podman 4.x is included by default (no Docker in RHEL 9)

### Container Runtime
- **Podman 4.0+** is the default container runtime
- Rootless containers are fully supported and recommended
- cgroups v2 is the default (vs cgroups v1 in RHEL 8)

### Security
- **SELinux** is enforcing by default - mirror-registry tool handles SELinux contexts automatically
- **firewalld** is the default firewall (enabled by default)
- FIPS mode can be enabled if required for compliance

### Networking
- NetworkManager is the default network management tool
- systemd-resolved is used for DNS resolution

### System Management
- **systemd** version 250+ (newer than RHEL 8)
- **OpenSSL 3.0** (vs OpenSSL 1.1.1 in RHEL 8)
- Python 3.9+ is the default Python version

### Known Compatibility
- OpenShift 4.20 is fully tested and supported on RHEL 9
- Mirror registry tool requires RHEL 9.0 or later
- Podman version must be 4.0 or higher (included in RHEL 9)

### Troubleshooting RHEL 9 Specific Issues

**Podman socket issues:**
```bash
# Enable user podman socket
systemctl --user enable --now podman.socket
systemctl --user status podman.socket
```

**SELinux denials:**
```bash
# Check for SELinux denials
sudo ausearch -m AVC -ts recent

# If needed, temporarily set to permissive for troubleshooting (not recommended for production)
sudo setenforce 0
# Re-enable after testing
sudo setenforce 1
```

**Firewalld not running:**
```bash
# Start and enable firewalld
sudo systemctl enable --now firewalld
sudo systemctl status firewalld
```

## Additional Resources

- [OpenShift 4.20 Documentation](https://docs.openshift.com/container-platform/4.20/)
- [RHEL 9 Documentation](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9)
- [Nutanix Platform Integration](https://docs.openshift.com/container-platform/4.20/installing/installing_nutanix/preparing-to-install-on-nutanix.html)
- [Disconnected Installation Guide](https://docs.openshift.com/container-platform/4.20/installing/disconnected_install/index.html)
- [Podman on RHEL 9](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/building_running_and_managing_containers/index)
