# Data Center Network Operations & Troubleshooting Lab

> **Walkthrough status:** Complete — all phases have instructions, explanations, and verification checkpoints. Phases 4C–12 include linked guides and configuration examples.

**Reference review:** October 26, 2026. See [keeping the lab current](documentation/maintenance.md) for package updates, version checks, and verification after maintenance.

A hands-on, step-by-step networking and Linux infrastructure lab using **UTM** on an Apple Silicon Mac (16 GB RAM). Follow the phases in order to build an Ubuntu server, learn networking fundamentals, troubleshoot controlled failures, and practice Linux administration.

This walkthrough targets **Ubuntu Server 26.04.1 LTS (ARM64)**, available from [Ubuntu's ARM download page](https://ubuntu.com/download/server/arm). Check your installed version with `cat /etc/os-release`; addresses, interface names, and output may differ on your VM.

**No Azure or other cloud subscription required.** The exercises run in local virtual machines.

## Contents

1. [How to use this walkthrough](#how-to-use-this-walkthrough)
2. [Learning objectives](#learning-objectives)
3. [Lab architecture](#lab-architecture)
4. [Prerequisites](#prerequisites)
5. [Exercise progress](#exercise-progress)
6. [Phase 1 — Create the VM](#phase-1--create-the-vm)
7. [Phase 2 — Install Ubuntu](#phase-2--install-ubuntu)
8. [Phase 3 — Inspect and prepare the server](#phase-3--inspect-and-prepare-the-server)
9. [Phase 3B — Expand LVM storage](#phase-3b--expand-lvm-storage)
10. [Phase 4A — Establish a network baseline](#phase-4a--establish-a-network-baseline)
11. [Phase 4B — IPv4, subnetting, routes, and ARP](#phase-4b--ipv4-subnetting-routes-and-arp)
12. [Phase 4C — DNS and DHCP](#phase-4c--dns-and-dhcp)
13. [Phase 5 — Linux networking toolkit](#phase-5--linux-networking-toolkit)
14. [Phase 6 — Remote SSH administration](#phase-6--remote-ssh-administration)
15. [Phase 7 — Multi-server networking](#phase-7--multi-server-networking)
16. [Phase 8 — Firewall and access control](#phase-8--firewall-and-access-control)
17. [Phase 9 — Packet analysis and performance](#phase-9--packet-analysis-and-performance)
18. [Phase 10 — Monitoring](#phase-10--monitoring)
19. [Phase 11 — Incident response](#phase-11--incident-response)
20. [Phase 12 — Final verification and lab notes](#phase-12--final-verification-and-lab-notes)
21. [Keeping the lab current](#keeping-the-lab-current)
22. [Troubleshooting](#troubleshooting)
23. [Lab notes and screenshots](#lab-notes-and-screenshots)
24. [Lab checklist](documentation/completion-review.md)

## How to use this walkthrough

1. Start with the prerequisites, then work through the phases in order.
2. Run Ubuntu commands inside the VM. Steps for UTM, Finder, or macOS Terminal explicitly name the host application.
3. Replace example usernames, IP addresses, interfaces, and volume names with the values on your own VM.
4. Read each command's explanation, run it, and compare your result with the checkpoint before continuing.
5. Keep short notes on what you observed, what failed, and how you resolved it. Screenshots are optional learning aids.
6. Use the [lab checklist](documentation/completion-review.md) to track your exercises. The progress recorded below describes the original lab, not your own progress.
7. From Phase 7 onward, check the guide's **Run on** label before each command. Keep the management NIC available while changing private-network settings. Variables such as `LAB_APP_IF` are local to the current terminal; set them again in a new terminal.

The commands below and in the linked guides are instructions for you to run. Expected results describe the checkpoints; they are not newly collected VM output.

## Learning objectives

- Provision and administer an Ubuntu Server VM.
- Explain IPv4 addresses, CIDR notation, subnets, gateways, DHCP, DNS, ARP, TCP/UDP, and routing.
- Use `ip`, `ping`, `getent`, `ss`, `systemctl`, `journalctl`, `tcpdump`, and `iperf3` to investigate problems.
- Build a multi-server test environment, then add controlled firewall and routing exercises.
- Monitor infrastructure and write clear incident records with evidence and verification.

## Lab architecture

**Starting setup for Phases 1–6:**

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

**Three-VM setup used in Phases 7–12:**

```text
    Shared Network / DHCP (Mac SSH, DNS, package downloads)
            |                 |                  |
      dc-server-01      dc-network-01       dc-monitor-01
      10.10.20.10       Linux router        10.10.30.10
            |          /            \             |
            +---------+              +------------+
            lab-app                   lab-monitor
         10.10.20.0/24              10.10.30.0/24
         router: .1                 router: .1
      UTM Host Network A        UTM Host Network B
```

> Each VM retains a Shared Network management NIC. Two separate UTM Host Networks carry the application and monitoring subnets; `dc-network-01` routes between them. [Phase 7](documentation/phase-07-multi-server-networking.md) shows the exact NIC assignments, static addresses, return routes, and connectivity tests. UTM's QEMU backend is required for this Host Networks setup.

## Prerequisites

- Apple Silicon Mac with **16 GB RAM** (this lab's host).
- A current stable [UTM](https://mac.getutm.app/) release installed. Compare your version with the [official releases](https://github.com/utmapp/UTM/releases/latest); the walkthrough uses the QEMU backend's Host Networks feature.
- Ubuntu Server **26.04.1 LTS ARM64** installation ISO from [Ubuntu](https://ubuntu.com/download/server/arm).
- A 20 GB virtual disk per VM: 60 GB total provisioned capacity for three VMs, plus host space for backups and installation media.
- UTM's QEMU backend with **Host Networks** support for Phase 7 onward. The [UTM networking preferences](https://docs.getutm.app/preferences/macos/#host-networks) describe this feature.
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

These are the starting specifications; use the same CPU/RAM/disk allocation for the two additional VMs in Phase 7. Record your installed OS version and actual configuration in your lab notes.

## Exercise progress

| Phase | Milestone | Status |
|---|---|---|
| 1 | Create UTM virtual machine | ✅ Complete |
| 2 | Install Ubuntu Server | ✅ Complete |
| 3 | Inspect OS, CPU, memory, disk, hostname | ✅ Complete |
| 3B | Back up VM and expand LVM root filesystem | ✅ Complete |
| 4A | Verify gateway, external IP, DNS, and local SSH service | ✅ Complete |
| 4B | Investigate local/external routing and ARP | ✅ Complete |
| 4C | DNS and DHCP troubleshooting | ✅ Complete |
| 5 | Linux networking toolkit | ✅ Complete |
| 6 | Remote SSH administration | ✅ Complete |
| 7 | Multi-VM connectivity, subnets, routing | ✅ Complete |
| 8 | Firewalls and network access controls | ✅ Complete |
| 9 | Packet capture and performance testing | ✅ Complete |
| 10 | Prometheus/Grafana monitoring | ✅ Complete |
| 11 | Simulated incidents and runbooks | ✅ Complete |
| 12 | Final verification and lab notes | ✅ Complete |

All listed lab tasks have been completed. If you are following this walkthrough, use the [lab checklist](documentation/completion-review.md) to track your own progress through each phase.

## Phase 1 — Create the VM

1. Download the Ubuntu Server **ARM64** ISO.
2. Open UTM → **Create a New Virtual Machine** → **Virtualize** → **Linux**.
   Use the **QEMU backend** for this walkthrough; if offered an Apple Virtualization option, leave it unselected. Keep ARM64 hardware virtualization enabled.
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

### Step 1 — Identify the operating system and host

```bash
cat /etc/os-release  # Distribution and version
hostnamectl         # Hostname, architecture, virtualization
lscpu               # CPU architecture and virtual CPU count
```

Confirm the hostname, ARM64/aarch64 architecture, installed release, and expected CPU count.

### Step 2 — Inspect resources and storage

```bash
free -h             # RAM and swap usage
df -h               # Filesystem usage
lsblk -f            # Disks, partitions, and filesystems
uptime              # Uptime and load averages
```

Distinguish configured RAM from current use and virtual disk size from the root filesystem size. A low free-memory figure alone does not prove a problem; inspect available memory.

### Step 3 — Record network settings

```bash
ip -br addr         # Interface state and addresses
ip route            # Routing table and default gateway
```

Identify the management interface, IPv4 address/prefix, and default gateway. Substitute your actual values for the examples used below.

### Step 4 — Prepare the basic tools

```bash
sudo apt update
apt list --upgradable
sudo apt upgrade
sudo apt install -y openssh-server curl dnsutils
mkdir -p ~/dc-lab-notes
```

**Example from the original lab:** Ubuntu 26.04.1 LTS, ARM64, interface `enp0s1`, DHCP IPv4 `192.168.64.3/24`, gateway `192.168.64.1`. The root filesystem initially showed approximately **9.8 GiB** with **4.1 GiB available**. Your output can differ.

`apt update` refreshes package metadata; `apt upgrade` installs available package updates after you review the proposed changes. If a reboot is required, reboot through the console and recheck the VM's address before continuing. The [maintenance guide](documentation/maintenance.md) explains the checks.

**Checkpoint:** You can identify the OS, CPU, memory, root filesystem, management address, and default route, and the package index refresh succeeds.

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
findmnt -no SOURCE,FSTYPE /
sudo vgs
sudo lvs
df -h /
```

Verify the volume group is `ubuntu-vg`, the logical volume is `ubuntu-lv`, and free extents are available **before** proceeding. Names may differ on other installations.

If the root is not an LVM logical volume, this specific expansion procedure does not apply. If the root already uses the available capacity and `vgs` shows no free extents, skip Step 3 and record the current layout; do not repeat a completed expansion.

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

**Example result from the original lab:** Root filesystem increased to approximately **17 GiB**, available space to approximately **11 GiB**, and utilization fell from **57% to 33%**. Volume group free space became approximately zero. Compare your own before/after output and see the [storage notes](documentation/linux-storage-management.md).

**Checkpoint:** The root filesystem is larger, and the volume group shows the expected reduction in free space. The workflow is inspect → identify unused capacity → back up → expand → verify.

## Phase 4A — Establish a network baseline

### Step 1 — Identify network settings

The examples use `enp0s1` and gateway `192.168.64.1`. Read your current output and replace those values if different.

```bash
ip -4 addr show enp0s1
ip route
```

### Step 2 — Test the gateway

```bash
ping -c 4 192.168.64.1
```

**Expected result:** Four replies with no loss on a healthy path. A responding gateway demonstrates local IP reachability; it does not alone prove internet access.

### Step 3 — Test an external IP

```bash
ping -c 4 1.1.1.1
```

**Expected result:** External IP replies if ICMP is allowed. A failed ping may reflect ICMP filtering, so corroborate with other tests rather than assuming the entire connection is down.

### Step 4 — Verify DNS

```bash
getent ahostsv4 google.com
```

**Expected result:** The hostname resolves to one or more IPv4 addresses. This tests name resolution separately from the IP-only checks.

### Step 5 — Check SSH

```bash
sudo systemctl status ssh
systemctl status ssh.socket --no-pager
sudo ss -tlnp | grep ':22'
```

**Expected result:** TCP 22 is listening. Depending on Ubuntu's activation method, the listener may be owned by `ssh.service` or `ssh.socket`. A local listener alone does not prove remote SSH access; [Phase 6](documentation/phase-06-remote-ssh.md) tests that path. Press `q` to exit the service status pager.

### Incident record — INC-001 (baseline verification)

Use the [INC-001 baseline record](incidents/INC-001-connectivity-baseline.md) as a reference for your notes. Record the date, commands, results, and interpretation. This is a preventive check; the fault/recovery simulations are in Phase 11.

**Checkpoint:** You can distinguish gateway reachability, external IP reachability, DNS resolution, and the local SSH listener.

## Phase 4B — IPv4, subnetting, routes, and ARP

**Goal:** Explain how the VM selects an on-link destination versus a default gateway, and identify the gateway's MAC address.

### Step 1 — Understand the current subnet

For the example address `192.168.64.3/24`:

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

Use the [INC-002 routing record](incidents/INC-002-routing-baseline.md) to organize the route lookups, gateway ping, neighbor table, date, and explanation. Compare your answers with the [subnetting reference](documentation/ipv4-and-subnetting.md#exercise-answers).

**Checkpoint:** The local destination has an on-link route, the remote destination uses the default gateway, and you can explain the difference. A route lookup does not prove that either destination responds.

## Phase 4C — DNS and DHCP

Follow the [DNS and DHCP walkthrough](documentation/phase-04c-dns-dhcp.md).

1. Identify the management interface and inspect its DHCP lease, gateway, and resolver settings.
2. Compare IP-only connectivity with `getent`, `resolvectl`, and `dig` lookups.
3. Stop the local resolver at the console, diagnose the failure, restore it, and retest.
4. Renew DHCP and record the resulting address.

**Checkpoint:** DNS is restored, and you can distinguish a resolver failure from an address or routing failure.

## Phase 5 — Linux networking toolkit

Follow the [toolkit walkthrough](documentation/phase-05-networking-toolkit.md).

1. Install and inspect `tcpdump`, `traceroute`, `iperf3`, `curl`, and `htop`.
2. Trace a path, inspect HTTP headers/timings, and identify service listeners.
3. Capture gateway ICMP traffic and run a bounded loopback throughput test.

**Checkpoint:** You can explain what each tool measures and what its output cannot establish.

## Phase 6 — Remote SSH administration

Follow the [remote SSH walkthrough](documentation/phase-06-remote-ssh.md).

1. Check the server listener and host-key fingerprint through the console.
2. Connect from macOS Terminal and confirm the remote hostname.
3. Generate a dedicated client key, install its public key, and test public-key-only login.
4. Inspect authentication logs and confirm console recovery.

**Checkpoint:** Both Mac-to-VM key-based SSH and UTM console login work.

## Phase 7 — Multi-server networking

Follow the [multi-server and routed-subnet walkthrough](documentation/phase-07-multi-server-networking.md).

1. Create `dc-server-01` and `dc-monitor-01`, each with a management DHCP NIC.
2. Create two isolated UTM Host Networks and attach the private NICs according to the address plan.
3. Back up Netplan, install the per-VM static interface/route examples, and try the configuration.
4. Enable IPv4 forwarding on `dc-network-01` and test both routed directions.
5. Install NGINX on the server and request its page from the monitor.

**Checkpoint:** Traffic between `10.10.20.10` and `10.10.30.10` crosses the Linux router while management access remains available.

## Phase 8 — Firewall and access control

Follow the [UFW walkthrough](documentation/phase-08-firewall-controls.md).

1. Inspect existing rules and allow management SSH before enabling UFW.
2. Configure router forwarding rules and the server's HTTP allow rule.
3. Verify normal access, insert a narrow HTTP deny, and compare local success with remote failure.
4. Remove the injected rule and retest.

**Checkpoint:** UFW remains active, intended traffic works, and the temporary blocked-service rule is gone.

## Phase 9 — Packet analysis and performance

Follow the [capture and throughput walkthrough](documentation/phase-09-packet-performance.md).

1. Capture ARP, ICMP, DNS, and HTTP/TCP on the relevant private or management NIC.
2. Transfer the captures to the Mac and interpret them in Wireshark.
3. Run bounded VM-to-VM iperf3 tests in both directions and save results.
4. Remove the temporary throughput-test allow rule and listener.

**Checkpoint:** You can interpret each protocol capture and explain the limits of a same-host VM throughput result.

## Phase 10 — Monitoring

Follow the [monitoring walkthrough](documentation/phase-10-monitoring.md).

1. Install Node Exporter on each VM and permit scrapes from the monitor.
2. Configure Prometheus and a local HTTP probe using the supplied examples.
3. Install Grafana, provision its data source, and open both UIs through an SSH tunnel.
4. Build CPU, memory, disk, network, scrape-health, and HTTP panels.
5. Stop/restart one exporter and observe its alert fire and resolve.

**Checkpoint:** All metrics and the HTTP probe are healthy, and the test alert has resolved.

## Phase 11 — Incident response

Follow the [five incident drills](documentation/phase-11-incident-response.md) and use the [incident template](incidents/TEMPLATE.md).

1. Record a healthy baseline and open a record for each drill.
2. Simulate SSH failure, DNS failure, an unreachable route, blocked HTTP, and reduced throughput one at a time.
3. Diagnose each fault, restore the baseline, and repeat the original failing test.
4. Record the timeline, cause, fix, verification, and escalation considerations.

**Checkpoint:** All five simulations have recovery records, and no injected fault remains.

## Phase 12 — Final verification and lab notes

Follow the [final verification walkthrough](documentation/phase-12-final-verification.md).

1. Repeat identity, storage, addressing, DNS, SSH, routing, firewall, service, and monitoring checks.
2. Confirm all drill rules, routes, queues, and listeners were cleaned up.
3. Reboot one VM at a time and recheck persistent settings and recovery.
4. Save final configuration notes, captures, results, and incident records; shut down cleanly if finished.

**Checkpoint:** The lab can be rebuilt and verified from your notes, and each simulated fault has a clear recovery path.

## Keeping the lab current

Follow the [maintenance walkthrough](documentation/maintenance.md) to inspect installed/candidate versions, apply Ubuntu and Grafana package updates, check UTM/Wireshark releases, and repeat the relevant service checks. Update one VM at a time and retain the walkthrough's LTS and QEMU network design.

## Troubleshooting

| Symptom | First checks |
|---|---|
| VM boots back into installer | Stop VM, detach Ubuntu ISO, boot virtual disk |
| No IPv4 address | `ip -br addr`; verify UTM network mode and DHCP |
| Gateway unreachable | `ip route`; `ip link`; verify gateway and network configuration |
| External IP works but domain fails | `resolvectl status`; `getent ahostsv4 ubuntu.com` |
| SSH connection refused | `systemctl status ssh`; `ss -tlnp`; firewall checks |
| Root filesystem unexpectedly small | `lsblk`; `vgs`; `lvs`; `df -h /` |

## Lab notes and screenshots

Keep your own notes, captures, and test output as you follow the walkthrough. `screenshots/` is ready for optional images; [configurations/](configurations/README.md) contains the Netplan, router forwarding, Prometheus, probe, alert, and Grafana examples used in the later phases.

Use notes or screenshots to help compare results and retrace troubleshooting steps. Screenshots are optional. `.gitignore` allows `screenshots/` to be tracked. Suggested organization (these image files do not exist yet):

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

Keep passwords, private SSH keys, and other secrets out of saved output and screenshots.

For each exercise, record the date, VM/hostname, command or action, actual result, and what you learned. Add a relative file link if you saved output or a screenshot. Compare your actual output with the examples rather than copying example results into your notes.

## Suggested repository layout

```text
datacenter-network-operations-lab/
├── README.md
├── architecture/
│   ├── network-topology.md
│   └── ip-address-plan.md
├── documentation/
│   ├── day-01-server-setup.md
│   ├── completion-review.md
│   ├── ipv4-and-subnetting.md
│   ├── linux-storage-management.md
│   ├── maintenance.md
│   └── phase-04c-… through phase-12-… (detailed guides)
├── incidents/
│   ├── INC-001-connectivity-baseline.md
│   ├── INC-002-routing-baseline.md
│   └── TEMPLATE.md
├── configurations/ (Netplan, sysctl, Prometheus, Grafana)
├── screenshots/
└── lessons-learned.md
```

---

*This repository documents a self-directed educational lab. Any incidents described are simulations unless explicitly labeled otherwise.*
