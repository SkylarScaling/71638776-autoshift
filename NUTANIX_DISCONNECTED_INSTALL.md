# OpenShift 4.20 Disconnected Installation on Nutanix AHV

Complete step-by-step guide for installing OpenShift 4.20 on Nutanix AHV in a disconnected environment using a jumphost.

## Documentation Sources

This guide is based on official Red Hat OpenShift documentation:

- [Creating a mirror registry with mirror registry for Red Hat OpenShift](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/disconnected_environments/installing-mirroring-creating-registry) - Official Red Hat documentation for mirror-registry tool
- [Mirroring images for a disconnected installation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/disconnected_environments/installing-mirroring-disconnected) - Image mirroring with oc-mirror plugin
- [OpenShift 4.20 Disconnected Environments (PDF)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/pdf/disconnected_environments/OpenShift_Container_Platform-4.20-Disconnected_environments-en-US.pdf) - Complete offline documentation
- [Quay mirror-registry GitHub Repository](https://github.com/quay/mirror-registry) - Official mirror-registry tool source and documentation
- [Installing OpenShift step-by-step](https://hackmd.io/@johnsimcall/Sk1gG5G6o) - Community guide for disconnected installations

## Prerequisites

**Jumphost Requirements:**
- **Red Hat Enterprise Linux 9.x** with internet access initially (for mirroring)
  - RHEL 8 is also supported but RHEL 9 is recommended for OpenShift 4.20
- Minimum 500GB disk space for container registry mirror
- Access to both internet-connected and air-gapped networks (or ability to transfer data)
- **Podman 4.0 or later** (included in RHEL 9)
- **OpenSSL** (included in RHEL 9)

**Nutanix Environment:**
- Nutanix AHV cluster with Prism Central
- Sufficient resources (CPU, RAM, storage) for cluster nodes
- Network VLAN with DHCP or static IPs available
- DNS configured for cluster domains

**Required Credentials:**
- Red Hat pull secret (from console.redhat.com)
- Nutanix Prism Central credentials
- SSH key pair

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

```bash
mkdir ~/ocp-install
cd ~/ocp-install

cat > install-config.yaml <<EOF
apiVersion: v1
baseDomain: example.com
compute:
- architecture: amd64
  hyperthreading: Enabled
  name: worker
  platform:
    nutanix:
      cpus: 8
      coresPerSocket: 2
      memoryMiB: 32768
      osDisk:
        diskSizeGiB: 120
  replicas: 3
controlPlane:
  architecture: amd64
  hyperthreading: Enabled
  name: master
  platform:
    nutanix:
      cpus: 8
      coresPerSocket: 2
      memoryMiB: 32768
      osDisk:
        diskSizeGiB: 120
  replicas: 3
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

## Step 7: Post-Installation Configuration

### 7.1 Access Cluster

```bash
export KUBECONFIG=~/ocp-install/auth/kubeconfig
oc whoami

# Get console URL and kubeadmin password
cat ~/ocp-install/auth/kubeadmin-password
oc whoami --show-console
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

### 7.3 Configure Image Registry for Disconnected

```bash
# Create PVC for registry storage (if using ODF or other storage)
oc create -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: image-registry-storage
  namespace: openshift-image-registry
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 500Gi
  storageClassName: <your-storage-class>
EOF

# Configure registry to use PVC
oc patch configs.imageregistry.operator.openshift.io cluster --type merge \
  -p '{"spec":{"storage":{"pvc":{"claim":"image-registry-storage"}},"managementState":"Managed"}}'
```

## Key Considerations

### DNS Requirements

DNS must be configured before installation:
- `api.<cluster-name>.<base-domain>` → API VIP
- `api-int.<cluster-name>.<base-domain>` → API VIP
- `*.apps.<cluster-name>.<base-domain>` → Ingress VIP

### Network Requirements

- Port 6443 (API) accessible from jumphost
- Port 443/80 (apps) accessible from users
- Nutanix network must allow VM-to-VM communication

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

## Resource Requirements

### Minimum Cluster Requirements

- **Control Plane**: 3 nodes × 8 vCPU × 32GB RAM × 120GB disk
- **Worker Nodes**: 3 nodes × 8 vCPU × 32GB RAM × 120GB disk
- **Bootstrap** (temporary): 1 node × 4 vCPU × 16GB RAM × 120GB disk

### Jumphost Storage Requirements

- Base OS: 50GB
- Container registry mirror: 100-150GB
- Operator catalogs (if mirrored): 50-100GB
- Temporary files: 50GB
- **Total recommended**: 500GB

## Next Steps

After successful installation:

1. **Configure authentication**: Set up LDAP, OIDC, or other identity providers
2. **Install storage**: Deploy OpenShift Data Foundation or configure external storage
3. **Configure monitoring**: Set up persistent storage for Prometheus
4. **Install operators**: Deploy additional operators from mirrored catalog
5. **Configure networking**: Set up egress IPs, network policies as needed
6. **Backup etcd**: Set up regular etcd backup jobs

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
