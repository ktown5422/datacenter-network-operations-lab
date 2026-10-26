# Data Center Network Operations & Troubleshooting Lab

> **Status:** In progress — Ubuntu server provisioning, baseline connectivity checks, SSH verification, and LVM expansion completed. IPv4 routing and subnetting exercises are next.

A hands-on, step-by-step networking and Linux infrastructure lab built with **UTM** and **Ubuntu Server 26.04.1 LTS (ARM64)** on an Apple Silicon Mac (16 GB RAM). The goal is to learn networking fundamentals, practice realistic data center troubleshooting, and create evidence-based documentation for entry-level **Data Center Technician** interviews.

**No Azure or other cloud subscription required.** This is a local virtual lab; it does not represent production data center employment or physical cabling experience.

## Contents

1. [Learning objectives](#learning-objectives)
2. [Lab architecture](#lab-architecture)
3. [Prerequisites](#prerequisites)
4. [Progress tracker](#progress-tracker)
5. [Phase 1 — Create the VM](#phase-1--create-the-vm)
6. [Phase 2 — Install Ubuntu](#phase-2--install-ubuntu)
7. [Phase 3 — Inspect and prepare the server](#phase-3--inspect-and-prepare-the-server)
8. [Phase 3B — Expand LVM storage](#phase-3b--expand-lvm-storage)
9. [Phase 4A — Establish a network baseline](#phase-4a--establish-a-network-baseline)
10. [Phase 4B — IPv4, subnetting, routes, and ARP](#phase-4b--ipv4-subnetting-routes-and-arp)
11. [Remaining phases](#remaining-phases)
12. [Troubleshooting](#troubleshooting)
13. [Evidence and screenshots](#evidence-and-screenshots)
14. [Interview talking points](#interview-talking-points)

## Learning objectives

- Provision and administer an Ubuntu Server VM.
- Explain IPv4 addresses, CIDR notation, subnets, gateways, DHCP, DNS, ARP, TCP/UDP, and routing.
- Use `ip`, `ping`, `getent`, `ss`, `systemctl`, `journalctl`, `tcpdump`, and `iperf3` to investigate problems.
- Build a multi-server test environment, then add controlled firewall and routing exercises.
- Monitor infrastructure and write clear incident records with evidence and verification.

## Lab architecture

**Current deployment (verified):**

```text
Apple Silicon Mac / UTM
       |
  Shared Network (UTM)
  192.168.64.0/24
       |
  Gateway 192.168.64.1
       |
  dc-network-01 (Ubuntu Server 26.04.1 LTS)
  enp0s1: 192.168.64.3/24 (observed; DHCP may change)
```

**Planned expansion (not yet built):**

```text
                 UTM virtual networking
                      |
       +--------------+---------------+
       |              |               |
 dc-network-01   dc-server-01    dc-monitor-01
 Linux networking   NGINX         Prometheus
 routing/firewall   SSH           Grafana
```

> UTM shared networking is convenient for initial access but is not, by itself, a realistic multi-subnet routed topology. We will verify VM-to-VM communication and configure suitable virtual interfaces before implementing separate networks.

## Prerequisites

- Apple Silicon Mac with **16 GB RAM** (this lab's host).
- [UTM](https://mac.getutm.app/) installed.
- Ubuntu Server **26.04.1 LTS ARM64** installation ISO from [Ubuntu](https://ubuntu.com/download/server/arm).
- At least 20 GB virtual disk space for the first VM, plus host space for backups and future VMs.
- Basic familiarity with a terminal; commands are explained below.

### VM specifications

| Setting | Value |
|---|---|
| VM name / hostname | `dc-network-01` |
| Virtualization | UTM → Virtualize → Linux |
| OS | Ubuntu Server 26.04.1 LTS ARM64 |
| vCPU | 2 |
| Configured RAM | 2 GB |
| Virtual disk | 20 GB |
| Network | Shared Network |
| SSH | OpenSSH server installed |

## Progress tracker

| Phase | Milestone | Status |
|---|---|---|
| 1 | Create UTM virtual machine | ✅ Complete |
| 2 | Install Ubuntu Server | ✅ Complete |
| 3 | Inspect OS, CPU, memory, disk, hostname | ✅ Complete |
| 3B | Back up VM and expand LVM root filesystem | ✅ Complete |
| 4A | Verify gateway, external IP, DNS, and SSH | ✅ Complete |
| 4B | Investigate local/external routing and ARP | 🔄 In progress |
| 4C | DNS and DHCP troubleshooting | ⬜ Planned |
| 5 | Linux networking toolkit | ⬜ Planned |
| 6 | Remote SSH administration | ⬜ Planned |
| 7 | Multi-VM connectivity, subnets, routing | ⬜ Planned |
| 8 | Firewalls and network access controls | ⬜ Planned |
| 9 | Packet capture and performance testing | ⬜ Planned |
| 10 | Prometheus/Grafana monitoring | ⬜ Planned |
| 11 | Simulated incidents and runbooks | ⬜ Planned |
| 12 | Final documentation and demo | ⬜ Planned |

## Phase 1 — Create the VM

1. Download the Ubuntu Server **ARM64** ISO.
2. Open UTM → **Create a New Virtual Machine** → **Virtualize** → **Linux**.
3. Select the Ubuntu ISO.
4. Allocate **2 CPU cores**, **2048 MB RAM**, and a **20 GB virtual disk**.
5. Name the VM `dc-network-01`.
6. In VM settings, confirm **Shared Network** is selected.
7. Save and start the VM.

**Checkpoint:** The Ubuntu Server installer appears.

## Phase 2 — Install Ubuntu

1. Select **Try or Install Ubuntu Server**.
2. Choose language and keyboard layout.
3. Keep automatic **DHCP** configuration; do not set a static address yet.
4. Leave proxy blank and use the default package mirror.
5. Install onto the **virtual** disk. The installer may use LVM by default.
6. Set the server hostname to `dc-network-01` and create a local user.
7. Select **Install OpenSSH server**.
8. Finish installation and reboot.
9. If the installer menu returns, **stop the VM and detach the installation ISO** from the virtual CD/DVD drive; do not delete the virtual hard disk. Start again.

**Checkpoint:** Log in and see a prompt similar to `kevin@dc-network-01:~$` (your username may differ).

## Phase 3 — Inspect and prepare the server

Run each command and record what it means:

```bash
cat /etc/os-release  # Distribution and version
hostnamectl         # Hostname, architecture, virtualization
free -h             # RAM and swap usage
df -h               # Filesystem usage
uptime              # Uptime and load averages
ip -br addr         # Interface state and addresses
ip route            # Routing table and default gateway
```

**Observed results:** Ubuntu 26.04.1 LTS, ARM64, interface `enp0s1`, DHCP IPv4 `192.168.64.3/24`, gateway `192.168.64.1`. The root filesystem was initially approximately **9.8 GiB** with **4.1 GiB available**.

**What to learn:** Distinguish virtual disk capacity, partitions, LVM logical volumes, filesystem capacity, and memory utilization.

## Phase 3B — Expand LVM storage

**Reason:** A 20 GB virtual disk was provisioned, but the Ubuntu root logical volume initially occupied only about 10 GB. LVM showed about 7.32 GiB free in its volume group.

### Step 1 — Back up the VM

1. Shut down Ubuntu cleanly:

   ```bash
   sudo shutdown -h now
   ```

2. In Finder, locate the VM's `.utm` package (UTM's **Show in Finder** option, if available).
3. **Copy**, rather than move, the package to a backup directory, for example `Documents/UTM-Backups/`.
4. Confirm the copy completed and preserve the original VM. Do not modify the backup.
5. Start the original VM again.

### Step 2 — Inspect storage

```bash
lsblk
sudo vgs
sudo lvs
df -h /
```

Verify the volume group is `ubuntu-vg`, the logical volume is `ubuntu-lv`, and free extents are available **before** proceeding. Names may differ on other installations.

### Step 3 — Expand the root logical volume and filesystem

> **Only run this against the confirmed volume on your own lab VM, after a backup.**

```bash
sudo lvextend -l +100%FREE -r /dev/ubuntu-vg/ubuntu-lv
```

- `lvextend`: expands a logical volume.
- `-l +100%FREE`: uses the remaining free extents in its volume group.
- `-r`: also resizes the filesystem.

### Step 4 — Verify

```bash
df -h /
sudo vgs
sudo lvs
```

**Observed result:** Root filesystem increased to approximately **17 GiB**, available space to approximately **11 GiB**, and utilization fell from **57% to 33%**. Volume group free space became approximately zero.

**Interview takeaway:** Inspect → identify unused capacity → back up → make an approved change → verify.

## Phase 4A — Establish a network baseline

### Step 1 — Identify network settings

```bash
ip -4 addr show enp0s1
ip route
```

### Step 2 — Test the gateway

```bash
ping -c 4 192.168.64.1
```

**Observed:** 4/4 replies, 0% packet loss. A responding gateway demonstrates local IP reachability; it does not alone prove internet access.

### Step 3 — Test an external IP

```bash
ping -c 4 1.1.1.1
```

**Observed:** 4/4 replies, 0% packet loss. A failed ping may reflect ICMP filtering, so corroborate with other tests.

### Step 4 — Verify DNS

```bash
getent ahostsv4 google.com
```

**Observed:** Hostname resolved to IPv4 addresses.

### Step 5 — Check SSH

```bash
sudo systemctl status ssh
sudo ss -tlnp | grep ':22'
```

**Observed:** SSH service active and listening on TCP port 22. **Remote login from the Mac has not yet been verified.** Press `q` to exit the service status pager.

### Incident record — INC-001 (baseline verification)

Document the system, date, commands, results, interpretation, and evidence. This is a **preventive verification exercise**, not a real outage.

## Phase 4B — IPv4, subnetting, routes, and ARP

**Status:** Exercise assigned; final command output not yet reviewed.

### Step 1 — Understand the current subnet

At the time of inspection:

| Property | Value |
|---|---|
| IPv4 address | `192.168.64.3` |
| Prefix | `/24` |
| Subnet mask | `255.255.255.0` |
| Network address | `192.168.64.0` |
| Broadcast | `192.168.64.255` |
| Conventional usable host range | `192.168.64.1–192.168.64.254` |
| Default gateway | `192.168.64.1` |

A `/24` leaves 8 host bits: `2^8 = 256` total addresses, conventionally **254 usable host addresses**. DHCP may assign only a subset.

### Step 2 — Inspect addressing and routes

```bash
ip -4 addr show enp0s1
ip route
```

If DHCP changed the address or gateway, use the **new values** in subsequent exercises.

### Step 3 — Compare a local and remote route

```bash
ip route get 192.168.64.10
ip route get 8.8.8.8
```

For the local `/24` destination, expect an on-link route via `enp0s1` with **no `via` gateway**. For the external destination, expect `via 192.168.64.1` (if the gateway is unchanged). `ip route get` shows a routing decision; it does **not** prove the destination exists or is reachable.

### Step 4 — Inspect the neighbor table

```bash
ping -c 4 192.168.64.1
ip neigh show
```

Look for the gateway's neighbor entry and its link-layer (MAC) address. States such as `REACHABLE` and `STALE` are normal.

### Step 5 — Answer these questions in your notes

1. What is the difference between an IP address and a MAC address?
2. Why does `192.168.64.10` use a local route but `8.8.8.8` use the gateway?
3. What does `/24` mean?
4. What can `ip route get` prove, and what can it **not** prove?
5. How does ARP help IPv4 communication on a local Ethernet segment?

### Incident record — INC-002 (routing baseline)

Record the route lookup outputs, gateway ping, neighbor table, and explanation. **Do not deliberately break your network settings yet.**

## Remaining phases

These are **planned exercises**, not completed achievements.

### Phase 4C — DNS and DHCP

- Inspect resolver configuration using `resolvectl status` and `getent`/`dig`.
- Inspect DHCP-assigned address, lease information, and default route.
- Compare IP connectivity against DNS name resolution.
- Document a controlled DNS failure and recovery once console access is confirmed.

### Phase 5 — Linux networking toolkit

- Install and learn `tcpdump`, `traceroute`, `iperf3`, `curl`, and `htop`.
- Explain what each tool measures and when to use it.

### Phase 6 — Remote SSH administration

- Connect from macOS Terminal to the VM's current IP.
- Practice key-based SSH authentication and log inspection.
- Confirm console recovery before firewall or SSH changes.

### Phase 7 — Multi-server networking

- Add `dc-server-01` and `dc-monitor-01` as resources permit.
- Test VM-to-VM connectivity.
- Design management, server, and monitoring subnets.
- Configure appropriate isolated interfaces and routing; verify rather than assume UTM topology behavior.

### Phase 8 — Firewall and access control

- Use UFW or nftables to allow required services and deny unwanted access.
- Reproduce and resolve a **controlled** blocked-port incident.

### Phase 9 — Packet analysis and performance

- Capture ARP, ICMP, DNS, and TCP traffic with `tcpdump`.
- Inspect captures in Wireshark.
- Measure VM-to-VM throughput using `iperf3`.

### Phase 10 — Monitoring

- Deploy Node Exporter, Prometheus, and Grafana.
- Monitor server reachability, CPU, memory, disk, and network metrics.
- Trigger a safe test alert and document the response.

### Phase 11 — Incident response

- Write simulated tickets for SSH failure, DNS failure, unreachable host, blocked service, and performance degradation.
- For each: symptoms → scope → diagnostics → root cause → resolution → verification → escalation considerations.

### Phase 12 — Portfolio presentation

- Publish network diagrams, screenshots, configuration examples, incident reports, and lessons learned.
- Record a short walkthrough showing actual commands and verified outcomes.
- Clearly distinguish **completed work** from **planned work**.

## Troubleshooting

| Symptom | First checks |
|---|---|
| VM boots back into installer | Stop VM, detach Ubuntu ISO, boot virtual disk |
| No IPv4 address | `ip -br addr`; verify UTM network mode and DHCP |
| Gateway unreachable | `ip route`; `ip link`; verify gateway and network configuration |
| External IP works but domain fails | `resolvectl status`; `getent ahostsv4 ubuntu.com` |
| SSH connection refused | `systemctl status ssh`; `ss -tlnp`; firewall checks |
| Root filesystem unexpectedly small | `lsblk`; `vgs`; `lvs`; `df -h /` |

## Evidence and screenshots

Store original screenshots in the repository using consistent descriptive filenames. Suggested organization:

```text
screenshots/
├── 01-server-configuration/
│   ├── ubuntu-os-version.jpeg
│   ├── ubuntu-hostname-system-info.jpeg
│   ├── ubuntu-memory-usage.jpeg
│   ├── ubuntu-disk-usage-before.jpeg
│   ├── ubuntu-server-uptime.jpeg
│   ├── ubuntu-network-interfaces.jpeg
│   └── ubuntu-routing-table.jpeg
├── 02-network-connectivity/
│   ├── gateway-connectivity-test.jpeg
│   ├── internet-connectivity-test.jpeg
│   ├── dns-resolution-test.jpeg
│   └── ssh-service-status.jpeg
└── 03-storage-management/
    ├── linux-storage-layout.jpeg
    ├── storage-lvm-initial-inspection.jpeg
    ├── lvm-filesystem-expansion.jpeg
    ├── lvm-volume-verification.jpeg
    └── storage-expansion-final-results.jpeg
```

Add images to this README **after** placing the corresponding files at those paths. For example:

```markdown
![Ubuntu network interfaces](screenshots/01-server-configuration/ubuntu-network-interfaces.jpeg)
![LVM expansion](screenshots/03-storage-management/lvm-filesystem-expansion.jpeg)
```

Do not publish credentials, private SSH keys, or sensitive configuration. A few representative screenshots in the README are easier to review than an unfiltered gallery.

## Suggested repository layout

```text
datacenter-network-operations-lab/
├── README.md
├── architecture/
│   ├── network-topology.md
│   └── ip-address-plan.md
├── documentation/
│   ├── day-01-server-setup.md
│   ├── ipv4-and-subnetting.md
│   └── linux-storage-management.md
├── incidents/
│   ├── INC-001-connectivity-baseline.md
│   └── INC-002-routing-baseline.md
├── configurations/
├── screenshots/
└── lessons-learned.md
```

## Interview talking points

- **Provisioning:** Installed Ubuntu Server ARM64 in UTM, selected DHCP networking, and enabled OpenSSH.
- **Networking:** Verified default gateway connectivity, external IP reachability, and DNS resolution; inspected network interfaces and routes.
- **Linux troubleshooting:** Used `lsblk`, `vgs`, `lvs`, and `df` to identify unallocated LVM space; backed up the VM and expanded the root filesystem from about 10 GiB to 17 GiB.
- **Operational discipline:** Captured before/after evidence, verified outcomes, and documented procedures.

**Next checkpoint:** Complete Phase 4B commands and add actual route/ARP screenshots. Then proceed to DNS and DHCP.

---

*This repository documents a self-directed educational lab. Any incidents described are simulations unless explicitly labeled otherwise.*
