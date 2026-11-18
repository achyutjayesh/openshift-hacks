# Manual OpenShift Cluster Build on VMware vSphere (IPI, FIPS, Disconnected)

## Table of Contents

- [Context](#context)
- [Part 1: Guidance for openshift-install](#part-1-guidance-for-openshift-install)
  - [Pre-Checks](#pre-checks)
  - [Preparation](#preparation)
  - [Obtaining a FIPS-capable installation program](#obtaining-a-fips-capable-installation-program)
  - [Install Steps](#install-steps)
  - [Post-Install Verification](#post-install-verification)
  - [Troubleshooting](#troubleshooting)
  - [Previous Issues Log](#previous-issues-log)
- [Part 2: Guidance for VMware vSphere and the Network Side of the Build](#part-2-guidance-for-vmware-vsphere-and-the-network-side-of-the-build)
  - [Step 0: Prerequisites](#step-0-prerequisites-you-set-up-before-the-install)
  - [Step 1: Installer prepares vSphere](#step-1-installer-prepares-vsphere)
  - [Step 2: Bootstrap VM boots](#step-2-bootstrap-vm-boots)
  - [Step 3: Masters boot](#step-3-masters-boot)
  - [Step 4: OVN-Kubernetes comes up](#step-4-ovn-kubernetes-comes-up)
  - [Step 5: Bootstrap hands off](#step-5-bootstrap-hands-off)
  - [Step 6: Workers and ingress](#step-6-workers-and-ingress)
  - [Firewall Summary for vSphere](#firewall-summary-for-vsphere)
  - [Network Problems to Check First](#network-problems-to-check-first-on-vsphere)

---

## Context

This document is required when building a cluster **manually**, rather than through the Ansible cluster build automation.

Hub clusters must always be built manually. The automation path registers the new cluster with Red Hat Advanced Cluster Management (ACM), which runs on the hub clusters, and provisions it using Hive. That flow does not apply to the hubs themselves.

> **Conventions used in this document**
>
> All hostnames, domains, IP addresses and credentials below are **examples**. Replace every `<placeholder>` and example value with the values for your environment before running anything.

---

## Part 1: Guidance for openshift-install

### Pre-Checks

#### DHCP

DHCP is a critical service and must be available before you start. SSH to each DHCP server and confirm the service is `active`:

```bash
sudo systemctl status dhcpd
```

#### Connectivity

Run these from the installation host (toolhost):

```bash
# DNS – must resolve external registries
dig +short registry.redhat.io
dig +short quay.io
dig +short mirror.openshift.com

# Proxy reachability
curl --proxy http://<proxy-host>:<proxy-port> https://www.google.com

# Mirror registry login
podman login <mirror-registry-host>:8443 -u <username> -p <password>

# Cluster subnet reachable from the toolhost
ping -c 3 <ip-in-cluster-subnet>
```

---

### Preparation

#### Identify the cluster details

The block below covers two scenarios: a **hub** (admin / management) cluster and a **spoke** (workload / application) cluster. Set the variables for the cluster you are building; they are used by commands throughout this page.

This document assumes there has been no previous attempt to build the cluster, so `WORK_DIR` will be new and empty.

```bash
# Common
CLUSTER_BUILD_ROOT_DIR=/home/<service-user>/cluster-build
ARCH=x86_64

##### Option A: hub / admin / management cluster
# Cluster Name       : mgmt-hub-01
# Base Domain        : example.local
# Platform           : VMware vSphere – IPI (Installer-Provisioned Infrastructure)
# Network Type       : dual-stack
# Environment        : Disconnected / Restricted (mirror registry used)
# Installation Host  : toolhost-rhel9-01
CLUSTER_NAME=mgmt-hub-01
BASE_DOMAIN=example.local
WORK_DIR=$CLUSTER_BUILD_ROOT_DIR/$CLUSTER_NAME

##### Option B: spoke / workload / application cluster
# Cluster Name       : workload-spoke-01
# Base Domain        : apps-example.local
# Platform           : VMware vSphere – IPI (Installer-Provisioned Infrastructure)
# Network Type       : dual-stack
# Environment        : Disconnected / Restricted (mirror registry used)
# Installation Host  : toolhost-rhel9-01
# CLUSTER_NAME=workload-spoke-01
# BASE_DOMAIN=apps-example.local
# WORK_DIR=$CLUSTER_BUILD_ROOT_DIR/$CLUSTER_NAME
```

#### Create the working directory

```bash
mkdir -p $WORK_DIR
cd $WORK_DIR
pwd
```

> **Note:** If the cluster build has been attempted before, `WORK_DIR` will already exist and may contain files from the earlier attempt.

#### Generate SSH keys

This is a one-time activity. If the cluster was built or attempted previously, **reuse the keys that are already present** in `WORK_DIR`.

```bash
cd $WORK_DIR
openssl rand -hex 8 > ./passphrase
ssh-keygen -t ecdsa -b 384 -C "$CLUSTER_NAME-build" -N "$(cat passphrase)" -f ./$CLUSTER_NAME-build

# Show the passphrase and the public key (the public key goes into install-config.yaml as sshKey)
cat passphrase
ssh-keygen -y -f ./$CLUSTER_NAME-build
```

#### Populate install-config.yaml

Get the latest template from your platform automation repository (for example `roles/cluster-build/templates/install-config.yml.j2`). The file is a Jinja2 template, so render it with the playbook's `render` tag rather than editing it by hand:

```bash
ansible-playbook clusterdeploy.yml -i inventory_localhost --tags render \
  -e var_clustername=<CLUSTER-INVENTORY-NAME> \
  -e var_pullsecret="$(jq -c . ./pull-secret.json)" \
  --become --ask-become-pass
```

Copy the rendered output to `$WORK_DIR/install-config.yaml` and review it. For a FIPS build it must contain `fips: true`.

> The file name must be `install-config.yaml` (**y-a-m-l**, not y-m-l).

---

### Obtaining a FIPS-capable installation program

> **Important:** For a FIPS installation, `openshift-install-fips` must be run on a **RHEL 9** toolhost. Check the OS of your toolhost with `cat /etc/redhat-release` before continuing.

```bash
cd $WORK_DIR

# Create pull-secret.json. Make sure quay.io is included.
# You can extract it from the ACM hub cluster or copy it from a previous build directory.
cp $CLUSTER_BUILD_ROOT_DIR/pull-secret.json $WORK_DIR/

# Verify the JSON format
jq . pull-secret.json

# Set the target OCP version and extract the FIPS installer
TARGET_VERSION=<ocp-version>-$ARCH     # e.g. 4.18.26-x86_64
oc adm release extract \
  --registry-config="$WORK_DIR/pull-secret.json" \
  --command=openshift-install-fips \
  --to=$WORK_DIR \
  quay.io/openshift-release-dev/ocp-release:$TARGET_VERSION
```

Confirm the binary was created:

```text
$WORK_DIR
├── install-config.yaml        <-- configuration with fips: true
├── openshift-install-fips     <-- FIPS-capable installer
└── pull-secret.json           <-- from the previous step
```

Check the version matches `TARGET_VERSION`:

```bash
./openshift-install-fips version
```

Example output:

```text
openshift-install 4.18.26
built from commit <commit-hash>
release image quay.io/openshift-release-dev/ocp-release@sha256:<digest>
WARNING Release Image Architecture not detected. Release Image Architecture is unknown
release architecture unknown
default architecture amd64
```

You can ignore the architecture warning.

---

### Install Steps

All commands below are run from `$WORK_DIR`. The installer consumes (deletes) `install-config.yaml` from its `--dir`, so always work on a **copy** inside the `install/` subdirectory and keep the original in `$WORK_DIR`.

#### 1. Standard clean installation

Use this for first-time or clean installs.

```bash
cd $WORK_DIR

# Clean up any previous run (safe to run on a first install)
./openshift-install-fips destroy cluster --dir install/ 2>/dev/null || true
rm -rf install/

# Fresh start
mkdir install
cp install-config.yaml install/

# Install
./openshift-install-fips create cluster --dir install/ --log-level=info
```

#### 2. Fast retry / debug install

```bash
cd $WORK_DIR

# Destroy the failed cluster
./openshift-install-fips destroy cluster --dir install/

# Recreate a fresh install/ directory
rm -rf install/ && mkdir install && cp install-config.yaml install/

# Re-run the install with debug logging
./openshift-install-fips create cluster --dir install/ --log-level=debug
```

#### 3. Successful cluster build

These are the final log lines of a successful build:

```text
INFO All cluster operators have completed progressing
INFO Checking to see if there is a route at openshift-console/console...
DEBUG Route found in openshift-console namespace: console
DEBUG OpenShift console route is admitted
INFO Install complete!
INFO To access the cluster as the system:admin user when using 'oc', run
INFO     export KUBECONFIG=<WORK_DIR>/install/auth/kubeconfig
INFO Access the OpenShift web-console here: https://console-openshift-console.apps.<cluster>.<domain>
INFO Login to the console with user: "kubeadmin", and password: "<generated-password>"
```

In the vSphere console you should see **3 master nodes** and **no bootstrap node** (the installer deletes it).

---

### Post-Install Verification

```bash
cd $WORK_DIR
export KUBECONFIG=install/auth/kubeconfig

# Quick health checks
oc whoami
oc get nodes
oc get co
oc get clusterversion
curl -k https://api.$CLUSTER_NAME.$BASE_DOMAIN:6443/version

# Check FIPS is enabled in the install config
oc get configmap cluster-config-v1 -n kube-system \
  -o jsonpath='{.data.install-config}' | grep -i fips
# Expected: fips: true

# Check FIPS is enabled on every node
for node in $(oc get nodes -o name); do
  oc debug $node -- chroot /host sysctl crypto.fips_enabled
done
# Expected on each node: crypto.fips_enabled = 1
```

---

### Troubleshooting

#### Debug access to the bootstrap and master nodes

```bash
cd $WORK_DIR

# The private key is protected by the passphrase generated earlier
cat passphrase

# SSH to the bootstrap node (exists only during installation)
ssh -i ./$CLUSTER_NAME-build core@<bootstrap-ip>

# SSH to any master or worker
ssh -i ./$CLUSTER_NAME-build core@<node-ip>

# Find the current bootstrap IP
grep "bootstrap" install/.openshift_install.log | tail -5

# List all node IPs the installer is using (adjust the prefix to your machine network)
grep -Eo '([0-9]{1,3}\.){3}[0-9]{1,3}' install/.openshift_install.log | sort -u | grep '^<subnet-prefix>'
```

#### Useful commands on the bootstrap or master nodes

```bash
journalctl -f -u bootkube.service           # watch bootstrap progress
journalctl -b -u kubelet.service -u crio    # kubelet and CRI-O logs
env | grep -i proxy                         # proxy settings
cat /etc/resolv.conf                        # DNS servers
dig registry.redhat.io                      # registry resolution
sudo podman ps -a                           # bootstrap containers
sudo crictl ps                              # running pods/containers
```

#### Collect logs for Red Hat Support

On a failed install, generate a bundle with `./openshift-install-fips gather bootstrap --dir install/`, then make it easy to retrieve:

```bash
# These commands assume you are not the root user
cp install/log-bundle-*.tar.gz ~        # copy the log bundle to your home directory
chmod 644 ~/log-bundle-*.tar.gz         # make the log bundle readable
```

---

### Previous Issues Log

Add every new issue found during a cluster build to this table, so the team builds up a reference of past problems and their fixes.

| Issue | Symptom | Root Cause | Permanent Fix Applied |
|---|---|---|---|
| Proxy intermittent / total outage | Bootstrap stuck; timeouts reaching `quay.io` | One ESXi host had the wrong physical uplink, which broke the NSX overlay to the top-of-rack switch | Network team corrected the uplink on the faulty ESXi host |
| DNS resolution failures | Cannot resolve `registry.redhat.io`, `quay.io` or the internal mirror registry | The local environment DNS servers blocked external internet domains, and the nameservers in `/etc/resolv.conf` were unreachable on 53/TCP. Test with: `timeout 5 bash -c '</dev/tcp/<nameserver_ip>/53' && echo open \|\| echo closed` | Switched the installer host and VMs to the corporate DNS servers; the virtualization team opened connectivity to the nameservers |
| Toolhost → cluster subnet unreachable | SSH to nodes failed; Ignition serving failed | NSX distributed firewall blocked traffic from the toolhost subnet to the cluster segment | NSX DFW rule permanently opened (toolhost → cluster segment) |
| Control-plane machines not provisioned within 15 minutes; logs show `must provide bootstrap host address` | VMs were created in vSphere but had no IPs. The installer found no addresses (`found bootstrap address: []`, `found master address: []` ×3) | DHCP servers were not running: an OpenSCAP policy had uninstalled the `dhcpd` package | Linux team updated the OpenSCAP policy for the DHCP servers to allow the `dhcp` package |

---

## Part 2: Guidance for VMware vSphere and the Network Side of the Build

### Step 0: Prerequisites you set up before the install

#### 1. One subnet / port group for the cluster (the machineNetwork)

Example plan for `10.10.20.0/24`:

| Item | Example IP |
|---|---|
| Gateway | 10.10.20.1 |
| API VIP | 10.10.20.5 |
| Ingress VIP | 10.10.20.6 |
| DHCP pool (bootstrap, masters, workers) | 10.10.20.50 – 10.10.20.150 |

- Both VIPs must sit **inside** the machine network and **outside** the DHCP pool.
- Plan for at least 1 bootstrap + 3 masters + 2 or more workers, with spare addresses for scaling.

#### 2. DHCP on the port group

DHCP should hand out the gateway, DNS and NTP. Newer releases can use static IPs instead, configured per host in `install-config.yaml` and passed to the VM through guestinfo.

#### 3. DNS records on your corporate DNS

| Record | Points to |
|---|---|
| `api.<cluster>.<domain>` | API VIP |
| `*.apps.<cluster>.<domain>` | Ingress VIP |
| `api-int.<cluster>.<domain>` (optional) | API VIP |

`api-int` is resolved by CoreDNS running on the nodes, but adding it to corporate DNS as well does no harm.

#### 4. Reachability

- Installer host → vCenter on 443
- Installer host → ESXi hosts on 443 (the OVA upload uses NFC)
- Cluster nodes → vCenter on 443 (the vSphere CPI, CSI driver and Machine API all call vCenter)
- Cluster nodes → registry or mirror on 443 (directly or through a proxy), and → NTP on 123

#### 5. install-config.yaml networking block

```yaml
platform:
  vsphere:
    apiVIPs: [10.10.20.5]
    ingressVIPs: [10.10.20.6]
    failureDomains: [...]   # vCenter, datacenter, cluster, datastore, network (port group)
networking:
  machineNetwork: [{cidr: 10.10.20.0/24}]
  clusterNetwork: [{cidr: 10.128.0.0/14, hostPrefix: 23}]
  serviceNetwork: [172.30.0.0/16]
  networkType: OVNKubernetes
```

None of these ranges may overlap your real networks or the OVN internal ranges (`100.64.0.0/16`, `100.88.0.0/16`, `169.254.0.0/17`).

---

### Step 1: Installer prepares vSphere

1. The installer logs in to the vCenter API on 443 and checks permissions, the port group, the datastore and the VIPs.
2. It uploads the RHCOS OVA to the datastore and turns it into a VM template. The disk data flows through vCenter to an ESXi host over NFC on 443.
3. It creates the VM folder and a vSphere tag containing the cluster infra ID. The tag is used later for cleanup.
4. It renders the Ignition configs. Each VM receives its config through `guestinfo.ignition.config.data`. With static IPs, the network settings go into `guestinfo.afterburn.initrd.network-kargs`.

### Step 2: Bootstrap VM boots

1. The bootstrap VM is cloned from the template and attached to the cluster port group.
2. It gets its IP, gateway and DNS from DHCP.
3. Ignition reads the full bootstrap config from guestinfo. No HTTP server is needed.
4. The bootstrap node pulls the release images from quay.io or your mirror.
5. It starts the on-prem networking static pods:
   - **keepalived** claims the API VIP (`10.10.20.5`) on the bootstrap node using VRRP (unicast by default on newer versions) and announces it with gratuitous ARP.
   - **CoreDNS** answers `api-int`, `api` and the node names locally.
   - **HAProxy** balances the API across healthy backends.
6. `bootkube.sh` starts the temporary control plane:
   - single-member etcd
   - kube-apiserver on 6443
   - Machine Config Server (MCS) on 22623

At this point `api-int:6443` and `api-int:22623` resolve to the VIP, and the VIP sits on the bootstrap node.

### Step 3: Masters boot

1. Three master VMs are cloned. Each gets a DHCP IP (for example `.51`, `.52`, `.53`).
2. Their guestinfo holds a small pointer Ignition config referencing `https://api-int.<cluster>.<domain>:22623/config/master`.
3. Each master resolves `api-int` to the API VIP, reaches the bootstrap MCS on 22623 and downloads its full Ignition config.
4. The masters write their configs, reboot, and start their own keepalived, HAProxy and CoreDNS static pods.
5. The kubelet on each master registers with the bootstrap API (VIP:6443).
6. etcd grows to three members on the master IPs (client traffic on 2379, peer traffic on 2380).
7. kube-apiserver comes up on each master.
8. HAProxy on each master listens on 9445, and an nftables rule redirects API VIP traffic on 6443 to it. HAProxy then balances across the masters' kube-apiservers on 6443.

### Step 4: OVN-Kubernetes comes up

1. The Cluster Network Operator deploys OVN-Kubernetes.
2. On every node, `configure-ovs.sh` creates `br-ex` and moves the VM's IP from `ens192` to `br-ex`.
3. Each node gets a pod subnet (for example, master-0 gets `10.128.0.0/23`).
4. Node-to-node pod traffic runs over Geneve on UDP 6081 between the VM IPs.
5. Service IPs (`172.30.x.x`) become OVN load balancers. Cluster DNS is served from `172.30.0.10`.
6. The vSphere CPI sets each node's addresses and provider ID. The CSI driver talks to vCenter on 443 for persistent volumes.
7. The nodes go `Ready`.

### Step 5: Bootstrap hands off

1. Once etcd has quorum on the three masters and their API servers are healthy, the bootstrap node reports bootstrap complete.
2. keepalived on the bootstrap node stops or drops in priority. The API VIP moves to a master, which sends gratuitous ARP so the network learns the new location.
3. The installer deletes the bootstrap VM through vCenter.
4. From here on, the API VIP and MCS run only on the masters.

### Step 6: Workers and ingress

1. The Machine API (vSphere provider) clones worker VMs from the template. Their guestinfo contains the worker pointer Ignition config.
2. Each worker gets a DHCP IP and fetches `api-int:22623/config/worker` from the MCS, now running on the masters.
3. Worker CSRs are approved automatically. Each worker joins and gets its `/23` pod subnet and Geneve tunnels.
4. The router pods (HAProxy ingress) run with `hostNetwork` on the workers, listening on 80 and 443.
5. keepalived on the workers holds the Ingress VIP (`10.10.20.6`) on a node with a healthy router pod. If that router moves, the VIP moves with it.
6. Traffic flows `*.apps` → Ingress VIP → router → app pods. The console and OAuth come up, and the install finishes.

---

### Firewall Summary for vSphere

| Port | From → To | Purpose |
|---|---|---|
| 443/TCP | Installer and nodes → vCenter | vSphere API, CPI, CSI, Machine API |
| 443/TCP | Installer → ESXi hosts | OVA upload (NFC) |
| 6443/TCP | Clients and nodes → API VIP / masters | Kubernetes API |
| 22623/TCP | Nodes → API VIP (internal only) | Machine Config Server |
| 2379–2380/TCP | Master ↔ master | etcd |
| 9445/TCP | Within masters | HAProxy for the API |
| 10250/TCP | Between all nodes | Kubelet |
| 6081/UDP | Between all nodes | Geneve |
| 9000–9999 TCP/UDP | Between all nodes | Host services and metrics |
| 30000–32767 TCP/UDP | Between all nodes | NodePorts |
| 80, 443/TCP | Clients → Ingress VIP | Apps, console, OAuth |
| IP protocol 112 | Between nodes on the port group | VRRP (keepalived) |
| 53, 123 | Nodes → DNS and NTP | Name resolution, time sync |

---

### Network Problems to Check First on vSphere

- **VIP conflicts:** a VIP already in use, or inside the DHCP range, causes ARP conflicts.
- **Duplicate cluster names on one L2 network:** keepalived derives its VRRP router IDs from the cluster name, so two clusters with the same name collide.
- **DHCP missing DNS or NTP options:** masters can't resolve `api-int`, or clock skew breaks certificates.
- **Installer can't reach the ESXi hosts on 443:** the OVA upload fails.
- **DRS or vMotion during the install:** can cause a brief VIP flap; it usually recovers on its own.
- **`br-ex` failures on multi-NIC VMs:** use one vNIC (VMXNET3) per node unless you've planned for more.
- **MTU:** Geneve adds about 100 bytes, so the cluster MTU is the port group MTU minus 100. Check this carefully on NSX overlays.
