# OpenShift Kubernetes Distribution - OKD v4.21.0 Bare Metal - User Provisioned Infrastructure (UPI)

** This branch is for [v4.21.0-okd-scos version](https://origin-release.apps.ci.l2s4.p1.openshiftapps.com/releasestream/4-scos-stable/release/4.21.0-okd-scos.6)

**OpenShift Kubernetes Distribution (OKD)** — User-provisioned bare-metal cluster for homelab and on-premise deployments.

- **Repository:** [OKD-BareMetal](https://github.com/mnhomelab/OKD-BareMetal)  
- **Network architecture (full scale):** [OKD-BareMetal – Network Diagram](https://mnhomelab.github.io/OKD-BareMetal/)

## Architecture Diagram

![Architecture Diagram](./picture/Architecture.png)

## Table of Contents

1. [Introduction to OKD](#1-introduction-to-okd)
2. [Architecture](#2-architecture)
3. [OKD Cluster Limitation](#3-okd-cluster-limitation)
4. [OKD Installation](#4-okd-installation)
5. [Production OKD Installation (User-Provisioned Bare Metal – HomeLab)](#5-production-okd-installation)
6. [Single Node OKD](#6-single-node-okd)
7. [Reference](#7-reference)
8. [Appendix](#appendix)

---

## 1. Introduction to OKD

**OKD** is a platform for developing and running containerized applications. It is designed to scale from a few machines and applications to thousands of machines serving many clients.

- Built on **Kubernetes** — the same technology used for large-scale telecom, streaming, gaming, and banking.
- Implemented with **open Red Hat technologies**, so you can run containerized applications across on-premise and multi-cloud environments.

---

## 2. Architecture

OKD is a cloud-style Kubernetes container platform. Its foundation is Kubernetes; see [product architecture](https://docs.okd.io/latest/architecture/) for more.

### 2.1 Custom Operating System

OKD uses **Fedora CoreOS (FCOS)** / **SCOS** (Stream CoreOS), a container-focused OS for running OKD and related tools, with fast installation, Operator-based management, and simpler upgrades.

FCOS/SCOS includes:

- **Ignition** — first-boot configuration for bringing up and configuring machines.
- **CRI-O** — Kubernetes-native container runtime (replaces Docker used in OKD 3).
- **Kubelet** — primary node agent that runs and monitors containers.

In OKD 4, **FCOS/SCOS is required for all control plane machines**. Compute (worker) machines can use FCOS or RHEL; RHEL workers require more OS lifecycle maintenance.

### 2.2 Glossary of Common Terms for OKD Architecture

| Term | Description |
|------|-------------|
| **access policies** | Roles that define how users, apps, and entities interact in the cluster. |
| **admission plugins** | Enforce security, resource limits, or configuration requirements. |
| **authentication** | Cluster admins configure user auth (OAuth token or X.509 cert) for API access. |
| **bootstrap** | Temporary machine that runs minimal Kubernetes and deploys the control plane. |
| **certificate signing requests (CSRs)** | Request for a signer to sign a certificate; can be approved or denied. |
| **Cluster Version Operator (CVO)** | Checks OKD Update Service for valid updates and update paths. |
| **compute nodes** | Run user workloads; also called worker nodes. |
| **configuration drift** | Node config does not match machine config. |
| **containers** | Lightweight executable images (app + dependencies), runnable anywhere. |
| **control plane** | Orchestration layer exposing API to define, deploy, and manage containers. |
| **CRI-O** | Kubernetes-native container runtime. |
| **deployment** | Kubernetes resource that maintains application lifecycle. |
| **Ignition** | FCOS utility for disk setup during initial configuration. |
| **installer-provisioned infrastructure** | Installer deploys and configures the cluster infrastructure. |
| **kubelet** | Node agent ensuring containers run in a pod. |
| **Machine Config Daemon (MCD)** | Checks nodes for configuration drift. |
| **Machine Config Operator (MCO)** | Applies new configuration to cluster machines. |
| **mirror registry** | Registry holding a mirror of OKD images. |
| **node** | Worker machine (VM or physical) in the cluster. |
| **OpenShift CLI (oc)** | Command-line tool for OKD. |
| **Operator** | Preferred way to package, deploy, and manage apps in OKD. |
| **pod** | Smallest compute unit: one or more containers with shared resources. |
| **user-provisioned infrastructure** | You provide infrastructure; installer generates assets, you create it and deploy the cluster. |
| **web console** | UI for managing OKD. |
| **worker node** | Same as compute node. |

*(Full glossary is in the source documentation.)*

---

## 3. OKD Cluster Limitation

### 3.1 OKD Tested Cluster Maximums for Major Releases

#### 3.1.1 On Cloud

For cost and resource estimation on cloud (OCP), use:

- **Calculator:** https://access.redhat.com/labs/ocplimitscalculator/

#### 3.1.2 On Prem

**Maximum Tested Cluster Scale (4.x)**

| Resource Type | Maximum Tested Value | Notes / Conditions |
|---------------|----------------------|--------------------|
| Number of Nodes | 2,000 | Tested cluster size |
| Number of Pods | 150,000 | Total across cluster |
| Pods per Node | 2,500 | Upper tested density; needs hostPrefix and maxPods config |
| Namespaces | 10,000 | Logical isolation |
| Builds | 10,000 | S2I, default pod RAM 512 Mi |
| Pods per Namespace | 25,000 | High multi-tenant |
| Routes (2 default routers) | 9,000 | Ingress capacity |
| Secrets | 80,000 | Including service + app secrets |
| ConfigMaps | 90,000 | Application config |
| Services | 10,000 | Cluster-wide |
| Services per Namespace | 5,000 | Per tenant/app |
| Backends per Service | 5,000 | Large-scale load balancing |
| Deployments per Namespace | 2,000 | Rollout objects |
| BuildConfigs | 12,000 | OpenShift build definitions |
| Custom Resource Definitions (CRDs) | 1,024 | API extensions limit |

- Tested on **31 servers**: 3 control plane, 2 infrastructure, 26 worker nodes.
- For 2,500 pods per node: `hostPrefix: 20` and custom kubelet `maxPods: 2500` (see [Running 2500 pods per node](https://docs.okd.io/latest/scalability_and_performance/)).
- Large etcd keyspace can hurt performance; periodic etcd defragmentation is recommended.
- **Reference:** [Planning your environment according to object maximums](https://docs.okd.io/latest/scalability_and_performance/planning-your-environment-according-to-object-maximums.html)

---

## 4. OKD Installation

The OKD installer supports four deployment methods:

1. **Interactive (Assisted Installer)** — Web-based; good for internet-connected networks; smart defaults and pre-flight checks; REST API for automation.
2. **Local Agent-based** — For disconnected or restricted networks; download and configure Agent-based Installer; CLI configuration; can use self-contained media without external registry.
3. **Automated (installer-provisioned)** — Installer uses BMC for provisioning; connected or disconnected.
4. **Full control (user-provisioned)** — You prepare and maintain infrastructure; maximum customizability; connected or disconnected.

All methods target:

- Highly available infrastructure (no single point of failure).
- Administrator control over when updates are applied.

### 4.1 Supported Platforms for OKD Clusters

| Platform | Installer-Provisioned | User-Provisioned | Agent-based Installer | Assisted Installer |
|----------|-----------------------|------------------|------------------------|--------------------|
| AWS | ❌ | ✅ | ❌ | ✅ |
| **Bare metal** | ❌ | ✅ | ✅ | ✅ |
| External | ❌ | ✅ | ❌ | ✅ |
| Google Cloud | ❌ | ✅ | ❌ | ✅ |
| IBM Cloud Classic | ❌ | ✅ | ❌ | ❌ |
| IBM Cloud VPC | ❌ | ✅ | ❌ | ❌ |
| IBM Power | ❌ | ✅ | ✅ | ❌ |
| IBM Z / LinuxONE | ❌ | ✅ | ✅ | ❌ |
| Microsoft Azure | ❌ | ✅ | ❌ | ❌ |
| Azure Stack Hub | ❌ | ✅ | ❌ | ❌ |
| None (existing infra) | ❌ | ✅ | ❌ | ❌ |
| Nutanix | ❌ | ✅ | ❌ | ❌ |
| Oracle Cloud (OCI) | ❌ | ✅ | ❌ | ❌ |
| OpenStack | ❌ | ✅ | ❌ | ❌ |
| VMware vSphere | ❌ | ✅ | ✅ | ✅ |

- **User-provisioned:** Can run with full internet, behind a proxy, or disconnected (mirror registry).
- **Download installer:** https://github.com/openshift/okd/releases (except Assisted Installer).

**Reference:** [Supported platforms](https://docs.okd.io/latest/architecture/architecture-installation.html#supported-platforms-for-openshift-clusters_architecture-installation)

### 4.2 Installation Process Details

Bootstrapping steps:

1. Bootstrap machine boots and hosts remote resources for control plane boot (manual if UPI).
2. Bootstrap starts single-node etcd and a temporary Kubernetes control plane.
3. Control plane machines fetch resources from bootstrap and finish boot (manual if UPI).
4. Temporary control plane schedules production control plane onto production control plane machines.
5. Cluster Version Operator (CVO) comes up; etcd Operator scales etcd on all control plane nodes.
6. Temporary control plane shuts down; production control plane takes over.
7. Bootstrap injects OKD components into production control plane.
8. Installer shuts down bootstrap (manual if UPI).
9. Control plane sets up compute nodes.
10. Control plane installs additional Operators.

**Reference:** [Update service](https://docs.okd.io/latest/architecture/architecture-installation.html#update-service-about_architecture-installation)

### 4.3 Minimum Nodes for OKD Cluster

| Hosts | Description |
|-------|-------------|
| **One temporary bootstrap** | Used only during install; can be removed after cluster is up. |
| **Three control plane** | Run core Kubernetes/OKD (API server, scheduler, controller manager, etcd). |
| **At least two compute (worker)** | Run application workloads and scale with demand. |

### 4.4 Minimum Resource Specification per Machine

| Machine | OS | CPU (vCPU) | RAM | Storage | IOPS |
|---------|----|------------|-----|---------|------|
| Bootstrap | SCOS | 4 | 16 GB | 100 GB | 300 |
| Control Plane | SCOS | 4 | 16 GB | 100 GB | 300 |
| Compute (Worker) | SCOS | 2 | 8 GB | 100 GB | 300 |

- 1 vCPU = 1 physical core when SMT/Hyper-Threading is disabled; otherwise use (threads × cores) × sockets.
- OKD/Kubernetes are sensitive to disk performance; etcd needs ≤10 ms p99 fsync. Fast storage (e.g. SSD/NVMe) is recommended.
- Fedora 7 compute is deprecated and removed in OKD 4.10+.

**Reference:** [Installing on bare metal](https://docs.okd.io/latest/installing/installing_bare_metal/upi/installing-bare-metal.html)

---

## 5. Production OKD Installation

### 5.1 User-Provisioned Cluster on Bare Metal Design (HomeLab)

This repository provides configs and steps for a **user-provisioned** OKD cluster on bare metal (homelab), including DNS, DHCP, HAProxy, install-config, and registry PV.

#### 5.1.1 Systems Specification (Example HomeLab)

| # | Name | Machine Model | Purpose | Architecture | vCPU | Memory | Storage (SSD/NVMe) |
|---|------|----------------|---------|---------------|------|--------|--------------------|
| 1 | Intel NUC | NUC7i5BNH | Service | Intel i5-7260 @ 2.20GHz | 4 | 8 GB DDR4 | 250 GB |
| 2 | JPYXKM | DAD600 / DLL600D | Bootstrap | Intel N100 @ 3.4GHz | 4 | 12 GB DDR5 | 512 GB |
| 3 | GMKtec NucBox | K9 Mini | Control Plane | Intel Ultra 125H @ 5.10 GHz | 18 | 32 GB DDR5 | 250 GB + 1 TB |
| 4 | HP EliteDesk | 705 G4 35W (396) | Control Plane | Ryzen 5 Pro 2400GE @ 3.2 GHz | 8 | 32 GB DDR4 | 250 GB |
| 5 | HP EliteDesk | 705 G4 36W (215) | Control Plane | Ryzen 5 Pro 2400GE @ 3.2 GHz | 8 | 32 GB DDR5 | 250 GB |
| 6 | GMKtec NucBox | K8 Plus Mini | Worker | AMD Ryzen 7 8845HS @ 5.1 GHz | 16 | 96 GB DDR5 | 128 GB + 1 TB |
| 7 | GMKtec NucBox | K12 Mini | Worker | AMD Ryzen 7 H 255 @ 3.8 GHz | 16 | 32 GB DDR5 | 128 GB + 1 TB |

**Useful commands:**

- CPU: `lscpu`
- Memory: `free -h`
- Storage: `lsblk`
- IOPS (example):  
  `sudo mkdir -p /var/tmp/fio`  
  `sudo fio --name=etcd-okd-test --directory=/var/tmp/fio --rw=randwrite --bs=4k --iodepth=1 --numjobs=1 --size=5G --runtime=60 --time_based --direct=1 --ioengine=libaio --fsync=1 --group_reporting`

#### 5.1.2 Network Architecture Diagram

Full-scale diagram: **[https://mnhomelab.github.io/OKD-BareMetal/](https://mnhomelab.github.io/OKD-BareMetal/)**

The `index.html` in this repo is the interactive network diagram (service node, bootstrap, control plane, workers, and ports).

#### 5.1.3 User-Provisioned Cluster on Bare Metal – Network Table (HomeLab)

| # | Name | Model | Purpose | Linux | IP Address | Hostname | MAC (example) |
|---|------|-------|---------|-------|------------|----------|----------------|
| 1 | Intel NUC | NUC7i5BNH | Service | Omarchy Linux | 192.168.8.206 / 10.0.0.1 | okd.ms1.lan | EC:9A:0C:1B:17:84 / F4:4D:30:6F:39:A3 |
| 2 | JPYXKM | DAD600/DLL600D | Bootstrap | CentOS Stream CoreOS | 10.0.0.145 | bootstrap.okd.ms1.lan | E0:51:D8:17:54:FC |
| 3 | GMKtec NucBox | K12 Mini | Control Plane | CentOS Stream CoreOS  | 10.0.0.150 | control-plane1.okd.ms1.lan | 84:47:09:71:F4:4B |
| 4 | HP EliteDesk | 705 G4 35W | Control Plane | CentOS Stream CoreOS | 10.0.0.151 | control-plane2.okd.ms1.lan | 80:E8:2C:2A:36:0D |
| 5 | HP EliteDesk | 705 G4 36W | Control Plane | CentOS Stream CoreOS  | 10.0.0.152 | control-plane3.okd.ms1.lan | 04:0E:3C:89:12:2F |
| 6 | GMKtec NucBox | K8 Plus Mini | Worker | CentOS Stream CoreOS  | 10.0.0.155 | compute1.okd.ms1.lan | C8:FF:BF:0F:7C:43 |
| 7 | GMKtec NucBox | K9 Mini | Worker | CentOS Stream CoreOS  | 10.0.0.160 | compute2.okd.ms1.lan | 84:47:09:54:7D:C9 |

---

### 5.2 User-Provisioned Cluster on Bare Metal – Pre-Steps (HomeLab)

*(Firewall may be disabled for testing; see [installation network connectivity](https://docs.okd.io/latest/installing/installing_bare_metal/upi/installing-bare-metal.html#installation-network-connectivity-user-infra_installing-bare-metal) for production.)*

#### 5.2.1 Clone Repository

```bash
git clone https://github.com/mnhomelab/OKD-BareMetal.git
```

#### 5.2.2 Setup OC Installer and OC Client

- **Installer:** https://github.com/okd-project/okd/releases (e.g. `4.21.0-okd-scos.6`)

```bash
wget https://github.com/okd-project/okd/releases/download/4.21.0-okd-scos.6/openshift-install-linux-4.21.0-okd-scos.6.tar.gz
tar -xvzf openshift-install-linux-4.21.0-okd-scos.6.tar.gz
sudo mv openshift-install /usr/bin/
```

- **Client:**

```bash
wget https://github.com/okd-project/okd/releases/download/4.21.0-okd-scos.6/openshift-client-linux-4.21.0-okd-scos.6.tar.gz
tar -xvzf openshift-client-linux-4.21.0-okd-scos.6.tar.gz
sudo mv kubectl oc /usr/bin/
```

- **Check:** `oc version` and `openshift-install version`  
  Note: release image should be `quay.io/openshift-release-dev/okd-release...` for OKD.

#### 5.2.3 Download Stream CentOS OS ISO and raw.tar.gz

On the service node:

```bash
mkdir ~/okd-images
cd ~/okd-images/
# ISO
curl -O $(openshift-install coreos print-stream-json | jq -r '.architectures.x86_64.artifacts.metal.formats.iso.disk.location')
# raw.gz
curl -O $(openshift-install coreos print-stream-json | jq -r '.architectures.x86_64.artifacts.metal.formats["raw.gz"].disk.location')
```

#### 5.2.4 Make Stream CentOS OS ISO Bootable on USB

- **Windows:** Use [Rufus](https://github.com/pbatard/rufus/releases). Partition: **MBR**; Target: **BIOS or UEFI**; File system: **Large FAT32**; Cluster size: **32 KB**; Write in **DD mode**.
- **Linux:** Identify USB (e.g. `/dev/sda`), then:

```bash
cd ~/okd-images
lsblk
sudo dd if=scos-*.x86_64.iso of=/dev/sda bs=4M status=progress
sync
```

#### 5.2.5 Enable Internet on OKD Nodes via Service Node (Optional)

- Enable IP forwarding (required):

```bash
# Temporary
sudo sysctl -w net.ipv4.ip_forward=1
# Permanent
echo "net.ipv4.ip_forward = 1" | sudo tee /etc/sysctl.d/99-ipforward.conf
sudo sysctl --system
```

- NAT (example; adjust interfaces):

```bash
sudo iptables -t nat -A POSTROUTING -o enp0s20f0u3 -j MASQUERADE
sudo iptables -A FORWARD -i eno1 -o enp0s20f0u3 -j ACCEPT
sudo iptables -A FORWARD -i enp0s20f0u3 -o eno1 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

- DHCP must set gateway and DNS to service node (e.g. `10.0.0.1`).  
- On a worker/control plane: `ip route`, `ping -c 3 8.8.8.8`, `dig google.com`.

---

### 5.3 User-Provisioned Cluster on Bare Metal – Service Node Setup (HomeLab)

#### 5.3.1 Static IP (Service Node)

- NIC to LAN (e.g. `eno1`): set `10.0.0.1/24`, DNS `10.0.0.1` in systemd-networkd (or your network config).
- Ensure `systemd-networkd` is enabled and the interface is routable; optionally disable NetworkManager for that interface.
- Verify: `networkctl status eno1`, `ip a`.

#### 5.3.2 DNS Setup

This repo provides BIND config and zone files under `scripts/DNS-Server/`.

- Install BIND (e.g. on Arch): `sudo pacman -S bind -y`
- Backup: `cp /etc/named.conf /etc/named.conf-original`
- Copy and adapt from repo:

```bash
cd <repository-location>
cp scripts/DNS-Server/named.conf /etc/named.conf
sudo chown root:named /etc/named.conf
sudo chmod 640 /etc/named.conf
```

- Create log dir: `mkdir /var/named/log/`, `chown named:named /var/named/log/`, `chmod 700 /var/named/log/`
- Create dynamic keys dir (avoids “managed-keys-directory” errors):  
  `sudo mkdir -p /var/named/dynamic`, `sudo chown -R named:named /var/named`
- Copy zones (adjust IPs if needed):

```bash
cp scripts/DNS-Server/okd.ms1.lan.zone /var/named/
cp scripts/DNS-Server/0.0.10.in-addr.arpa.zone /var/named/
chown root:named /var/named/okd.ms1.lan.zone /var/named/0.0.10.in-addr.arpa.zone
chmod 640 /var/named/okd.ms1.lan.zone /var/named/0.0.10.in-addr.arpa.zone
```

- Check: `named-checkconf`, `named-checkzone okd.ms1.lan /var/named/okd.ms1.lan.zone`, `named-checkzone 0.0.10.in-addr.arpa /var/named/0.0.10.in-addr.arpa.zone`
- Start: `systemctl enable --now named`
- Test: `dig +short @localhost A api.okd.ms1.lan`, `dig +short @localhost -x 10.0.0.1`

**Repo files:**

- `scripts/DNS-Server/named.conf` — BIND options, forward zone `okd.ms1.lan`, reverse `0.0.10.in-addr.arpa`.
- `scripts/DNS-Server/okd.ms1.lan.zone` — A records for api, api-int, *.apps, bootstrap, control-plane1–3, compute1–2, bastion, oauth, console.
- `scripts/DNS-Server/0.0.10.in-addr.arpa.zone` — PTR records for 10.0.0.x.

#### 5.3.3 DHCP Server Setup (Service Node)

- Install (e.g. Arch): `sudo pacman -S dhcp`
- Copy config from repo:

```bash
cp scripts/DHCP-Server/dhcpd.conf /etc/dhcpd.conf
```

- Open firewall for DHCP: `sudo ufw allow 67/udp`, `sudo ufw allow 68/udp`
- Enable: `sudo systemctl enable --now dhcpd4.service`

**Repo file:** `scripts/DHCP-Server/dhcpd.conf` — subnet `10.0.0.0/24`, options (routers, domain-name-servers, domain-name), and fixed host entries for bootstrap, control-plane1–3, compute1–2 (MAC + hostname). Adjust MACs for your hardware.

#### 5.3.4 HAProxy Setup

- Install (e.g. Arch): `sudo pacman -S haproxy -y`
- Copy and adapt from repo:

```bash
cp scripts/HAProxy/haproxy.cfg /etc/haproxy/
sudo chown root:haproxy /etc/haproxy/haproxy.cfg
sudo chmod 640 /etc/haproxy/haproxy.cfg
```

- Ensure stats directory exists: `sudo install -d -o haproxy -g haproxy -m 0755 /var/lib/haproxy`
- Firewall: `sudo ufw allow proto tcp to any port 6443,22623,443,80`
- Start: `systemctl enable --now haproxy.service`

**Repo file:** `scripts/HAProxy/haproxy.cfg` — TCP listeners: API 6443, Machine Config Server 22623 (control-plane1–3; bootstrap entries commented after bootstrap complete), ingress 80/443 (compute1–2), stats on 9000. Use TCP mode (no `option httplog` on TCP frontends).

#### 5.3.5 Setup Apache Web Server

- Install (e.g. Arch): `sudo pacman -Syu apache`
- Listen on 8080: `sed -i 's/Listen 80/Listen 0.0.0.0:8080/' /etc/httpd/conf/httpd.conf`
- Enable/start: `systemctl enable --now httpd`
- Test: `curl http://localhost:8080`

#### 5.3.6 Setup NFS Server

- Install: `sudo pacman -Syu nfs-utils`
- Export for registry:

```bash
sudo mkdir -p /shares/registry
sudo chown nobody:nobody /shares/registry
sudo chmod 755 /shares/registry
echo "/shares/registry 10.0.0.0/24(rw,sync,no_subtree_check,root_squash,no_wdelay)" | sudo tee /etc/exports
sudo exportfs -rv
sudo systemctl enable --now rpcbind nfs-server
```

#### 5.3.7 Setup OC Installer and OC Client

Same as [5.2.2](#522-setup-oc-installer-and-oc-client); ensure `openshift-install` and `oc` are on PATH.

- Create install directory:

```bash
rm -rf ~/okd-install
mkdir -p ~/okd-install
```

- Use this repo’s `install-config.yaml` (set your `pullSecret` and `sshKey`):

```bash
cd <repository-location>
cp scripts/Openshift/install-config.yaml ~/okd-install
```

**Repo file:** `scripts/Openshift/install-config.yaml` — baseDomain `ms1.lan`, cluster name `okd`, machineNetwork `10.0.0.0/24`, OVNKubernetes, 3 control plane, 0 compute (workers added manually). Replace `pullSecret` and `sshKey` with your values.

- Generate manifests and ignition:

```bash
openshift-install create manifests --dir ~/okd-install/
sed -i 's/mastersSchedulable: true/mastersSchedulable: false/g' ~/okd-install/manifests/cluster-scheduler-02-config.yml
openshift-install create ignition-configs --dir ~/okd-install/
```

#### 5.3.8 Add Ignition and SCOS Files to Web Server

```bash
sudo rm -rf /srv/http/okd
sudo mkdir -p /srv/http/okd
sudo cp -r ~/okd-install/* /srv/http/okd
sudo cp ~/okd-images/* /srv/http/okd/
sudo mv /srv/http/okd/scos-*.raw.gz /srv/http/okd/scos.raw.gz
sudo mv /srv/http/okd/scos-*.raw.gz.sig /srv/http/okd/scos.raw.gz.sig
sudo chown -R http:http /srv/http
sudo chmod -R 755 /srv/http
sudo systemctl restart httpd.service
```

*(Adjust paths if your HTTP root is different, e.g. `/var/www/html`.)*

#### 5.3.9 Boot SCOS from USB and Apply Ignition

- Use DD-mode USB (Ventoy is not supported for this OS).
- Ensure target disk is clean: `lsblk`, `sudo wipefs -a /dev/sdX` if needed.

- **Bootstrap node:**

```bash
sudo coreos-installer install /dev/sda --ignition-url http://10.0.0.1:8080/okd/bootstrap.ign --image-url http://10.0.0.1:8080/okd/scos.raw.gz --insecure-ignition --insecure
```

- **Control plane nodes (example device nvme0n1):**

```bash
sudo coreos-installer install /dev/nvme0n1 --ignition-url http://10.0.0.1:8080/okd/master.ign --image-url http://10.0.0.1:8080/okd/scos.raw.gz --insecure-ignition --insecure
```

#### 5.3.10 Bootstrap Completion

- On the service node, wait for bootstrap to finish (can take ~1–2 hours):

```bash
openshift-install wait-for bootstrap-complete --dir ~/okd-install --log-level=debug
```

- When it completes you’ll see “Bootstrap Complete” and “It is now safe to remove the bootstrap resources.”
- Optional: SSH to bootstrap and watch logs:  
  `ssh core@10.0.0.145`  
  `journalctl -b -f -u release-image.service -u bootkube.service -u node-image-pull.service`  
  (You can ignore `node-image-pull.service` failure after bootstrap is complete.)
- **Remove bootstrap from HAProxy:** Edit `/etc/haproxy/haproxy.cfg`, comment or remove the bootstrap server lines for 6443 and 22623, then:

```bash
sudo systemctl restart haproxy
```

- **Worker nodes:** Install with worker ignition:

```bash
sudo coreos-installer install /dev/sda --ignition-url http://10.0.0.1:8080/okd/worker.ign --image-url http://10.0.0.1:8080/okd/scos.raw.gz --insecure-ignition --insecure
```

- Debug workers: `ssh core@10.0.0.155` / `10.0.0.160`, then `sudo journalctl -u kubelet -f` or search for ignition/mco/kubelet in `journalctl -b --no-pager`.

---

### 5.4 User-Provisioned Cluster on Bare Metal – Post-Steps (HomeLab)

#### 5.4.1 Approve Certificate Signing Requests

On the service node (with `kubeconfig` from `~/okd-install/auth/kubeconfig` or `export KUBECONFIG=~/okd-install/auth/kubeconfig`):

```bash
oc get csr
# Approve one by one:
oc adm certificate approve <csr-name>
# Or all pending:
oc get csr -oname | xargs oc adm certificate approve
oc get csr,node -A
```

Nodes may stay `NotReady` for several minutes until pods and config are provisioned.

#### 5.4.2 Configure Registry

- Set image registry to use PVC and create a PV that backs it (this repo provides the PV manifest):

```bash
oc edit configs.imageregistry.operator.openshift.io
# Set:
#   managementState: Managed
#   storage:
#     pvc:
#       claim:  # leave blank
```

- Confirm PVC exists: `oc get pvc -n openshift-image-registry` (may be Pending).
- Create PV from repo (NFS server must be 10.0.0.1 and export `/shares/registry`):

```bash
oc create -f manifest/registry-pv.yaml
```

- Check binding: `oc get pvc -n openshift-image-registry`

**Repo file:** `manifest/registry-pv.yaml` — 100Gi ReadWriteMany NFS PV, server `10.0.0.1`, path `/shares/registry`.

#### 5.4.3 Configure Remote Access

- On clients that need to reach the console and apps, add to `/etc/hosts`:

```
10.0.0.1 console-openshift-console.apps.okd.ms1.lan
10.0.0.1 *.apps.okd.ms1.lan
```

- Open: `https://console-openshift-console.apps.okd.ms1.lan/`

![OKD Console ](./picture/Console.png)

#### 5.4.4 Make Masters Schedulable or Unschedulable

- View: `oc get schedulers.config.openshift.io cluster -o yaml | grep -i mastersSchedulable`
- Allow workloads on control plane:  
  `oc patch schedulers.config.openshift.io cluster --type=merge -p '{"spec":{"mastersSchedulable":true}}'`
- Disallow (recommended):  
  `oc patch schedulers.config.openshift.io cluster --type=merge -p '{"spec":{"mastersSchedulable":false}}'`

---

### 5.5 OKD Troubleshooting – User-Provisioned Bare Metal

| # | Problem | Solution |
|---|---------|----------|
| 1 | **invalid managed-keys-directory /var/named/dynamic: file not found** | `sudo mkdir -p /var/named/dynamic`, `sudo chown -R named:named /var/named`, `sudo chmod 750 /var/named /var/named/dynamic`, `sudo systemctl restart named` |
| 2 | **HAProxy:** `option httplog` not usable with proxy (needs `mode http`) or SSL cipher errors | Use TCP mode for API and MCS; remove `option httplog` from TCP frontends (use `tcplog`). Avoid binding SSL options to TCP backends. |
| 3 | **dig -x 10.0.0.145** times out (e.g. to 127.0.0.53) | On the node 10.0.0.145, open DNS: `sudo ufw allow 53/tcp` (and 53/udp if needed). |
| 4 | **Fedora CoreOS bare-metal boot fails to mount /run/media/iso** | Write USB with Rufus in **DD mode** (MBR, BIOS or UEFI, Large FAT32, 32 KB cluster). See [coreos/fedora-coreos-tracker#554](https://github.com/coreos/fedora-coreos-tracker/issues/554). |
| 5 | **cannot bind UNIX socket /var/lib/haproxy/stats** | `sudo install -d -o haproxy -g haproxy -m 0755 /var/lib/haproxy`, `sudo systemctl restart haproxy` |
| 6 | **node-image-pull.service:** unauthorized to quay.io/openshift-release-dev/ocp-v4.0-art-dev | Use the correct OKD release (e.g. 4.21.0-okd-scos.6) from https://github.com/okd-project/okd/releases and regenerate ignition and manifests. |
| 7 | **node-image-pull.service: Failed with result 'exit-code'** | Re-run bootstrap (reinstall bootstrap node and/or regenerate ignition if pull-secret or release mismatch). |
| 8 | **wait-for bootstrap-complete:** “Still waiting for the Kubernetes API: ... EOF” | Often HAProxy/TLS: remove strict SSL verification for API backends or use correct TLS settings so API is reachable. Check HAProxy config and backend connectivity. |

Additional debug on bootstrap/control plane: `sudo crictl ps`.

---

## 6. Single Node OKD

### 6.1 Prerequisites

- **Administration host** to prepare ISO, create USB, and monitor install.
- **CPU:** x86_64, arm64, ppc64le, or s390x.
- **Server:** Production-grade; see table below.

**Minimum resource requirements (single node):**

| Profile | Compute | Memory | Storage |
|---------|---------|--------|---------|
| Minimum | 8 vCPUs | 16 GB RAM | 120 GB |

### 6.2 Assisted Installer

Single-node deployments can use the Assisted Installer flow (web or API).

### 6.3 OKD Troubleshooting

See [5.5](#55-okd-troubleshooting--user-provisioned-bare-metal) and official docs; for GPU worker setup see “GPU Configuration with OKD Worker Node” and NVIDIA/NFD Operator docs.

---

## 7. Reference

- **ocp4-metal-install** — Community UPI for OKD on bare metal: https://github.com/ryanhay/ocp4-metal-install  
- **Installing OKD on Bare Metal (official):** https://docs.okd.io/latest/installing/installing_bare_metal/upi/installing-bare-metal.html  
- **okd_bare_metal (example repo):** https://github.com/eduardolucioac/okd_bare_metal  
- **OKD SNO UPI:** https://okd.io/docs/project/guides/upi-sno  
- **YouTube – OKD Bare Metal Install:** https://youtu.be/10w6sJ0hbhI  
- **OKD Cluster OpenShift Installation:** https://www.linuxsysadmins.com/okd-cluster-openshift-installation/  
- **BIND DNS for OKD HA:** https://www.linuxsysadmins.com/setting-bind-dns-server-for-okd-ha-clusters/  
- **DHCP for OKD HA:** https://www.linuxsysadmins.com/configuring-dhcp-server-for-okd-ha-clusters/  
- **HAProxy for OKD HA:** https://www.linuxsysadmins.com/configuring-haproxy-lb-for-okd-ha-cluster/  

---

## Appendix – Quick Reference, Assumptions & Operational Guardrails

### Scope & Intent

Operational reference for OKD clusters deployed with **User-Provided Infrastructure (UPI)** on bare metal or homelab: design assumptions, constraints, validated patterns, and common failure domains.

### Deployment Assumptions

- Fully user-managed infrastructure  
- Static IP addressing preferred  
- Fedora CoreOS / SCOS required for bootstrap and control plane  
- External DNS, DHCP, HAProxy, HTTP, and NFS required  
- Internet required at install unless using a mirror  

### Mandatory External Dependencies

| Service | Role |
|---------|------|
| **DNS (BIND)** | Forward and reverse zones must be correct |
| **DHCP** | Static reservations recommended |
| **HAProxy** | TCP for API (6443) and MCS (22623) |
| **HTTP server** | Serves ignition and OS artifacts |
| **NFS** | Persistent registry storage |

### Minimum & Recommended Sizing

- **Control plane:** 4 vCPU, 16 GB RAM, 100 GB SSD/NVMe, ≤10 ms disk latency  
- **Compute:** 2+ vCPU, 8+ GB RAM, 100 GB storage  

### Bootstrap Lifecycle

- Bootstrap is temporary; `node-image-pull.service` failure after completion is expected.  
- Remove bootstrap from HAProxy after `bootstrap-complete`.  

### Network & Port Contract

| Port | Purpose |
|------|---------|
| 6443 | Kubernetes API |
| 22623 | Machine Config Server |
| 80 / 443 | Ingress |
| 53 | DNS |
| 67 / 68 | DHCP |

### Ignition & OS Boot Constraints

- Ventoy is not supported; use DD mode for USB.  
- Disks should be wiped before install.  
- Ignition URLs must be reachable until first reboot.  

### Common Failure Domains

- DNS misconfiguration  
- HAProxy TLS / EOF errors  
- Invalid pull-secret  
- Slow disk I/O (etcd)  
- Firewall blocking required ports  

### Operational Best Practices

- Approve CSRs in batches  
- Keep masters unschedulable unless needed  
- Use a clean install directory per attempt  
- Enable debug logging during bootstrap  

### Supported Variants

- HA OKD (multi-node)  
- Single Node OKD (SNO)  
- Disconnected installs with mirror registry  
- Homelab / lab environments  

### Final Notes

- UPI gives maximum control and maximum responsibility.  
- Correct DNS and HAProxy configuration are the main factors for a successful install.  

---

## Repository Layout (OKD-BareMetal)

| Path | Purpose |
|------|---------|
| `scripts/DNS-Server/named.conf` | BIND options and zone definitions for `okd.ms1.lan` and `0.0.10.in-addr.arpa` |
| `scripts/DNS-Server/okd.ms1.lan.zone` | Forward zone A records (api, api-int, *.apps, bootstrap, control-plane, compute) |
| `scripts/DNS-Server/0.0.10.in-addr.arpa.zone` | Reverse zone PTR records for 10.0.0.x |
| `scripts/DHCP-Server/dhcpd.conf` | DHCP subnet and fixed host entries for bootstrap, control plane, workers |
| `scripts/HAProxy/haproxy.cfg` | Load balancer for API (6443), MCS (22623), ingress (80/443) |
| `scripts/Openshift/install-config.yaml` | OKD install config (baseDomain, cluster name, networking, pullSecret, sshKey) |
| `manifest/registry-pv.yaml` | NFS PersistentVolume for image registry |
| `index.html` | Interactive network architecture diagram (publish to GitHub Pages for full-scale view) |
| `Documentation/OKD-Baremetal-Documentation.docx` | Full handbook (source for this README) |

---

**License / usage:** Use and adapt the configs for your environment; adjust IPs, hostnames, and MACs to match your hardware and network.
