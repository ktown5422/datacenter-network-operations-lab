# Phase 7 — Multi-server networking and routed subnets

[Previous: remote SSH](phase-06-remote-ssh.md) · [Next: firewall controls](phase-08-firewall-controls.md)

**Run on:** UTM/macOS and all three Ubuntu VMs, as labeled. **Start with:** the first VM's working baseline, console access, and backups. **Goal:** Route traffic between two isolated lab networks without changing the management/internet path.

## Step 1 — Follow one address plan

Every VM keeps a **Shared Network** management NIC with DHCP for SSH, DNS, and package downloads. Application and monitoring traffic uses two extra **Host Networks** in UTM's **QEMU backend**. QEMU can still use hardware virtualization for ARM64; this is distinct from UTM's Apple Virtualization backend.

| VM | Management NIC | Application NIC (`lab-app`) | Monitoring NIC (`lab-monitor`) |
|---|---|---|---|
| `dc-network-01` | DHCP, e.g. `enp0s1` | `10.10.20.1/24`, e.g. `enp0s2` | `10.10.30.1/24`, e.g. `enp0s3` |
| `dc-server-01` | DHCP, e.g. `enp0s1` | `10.10.20.10/24`, e.g. `enp0s2` | None |
| `dc-monitor-01` | DHCP, e.g. `enp0s1` | None | `10.10.30.10/24`, e.g. `enp0s2` |

The router forwards between `10.10.20.0/24` and `10.10.30.0/24`. The other VMs have a specific route to the opposite lab subnet through that router. Their default routes remain on management DHCP. No NAT is needed for this lab path.

If either subnet overlaps a VPN or an existing network on your setup, choose two unused private `/24` ranges and change **all** addresses, routes, rules, and monitoring targets consistently before continuing.

## Step 2 — Create the two additional VMs

In **UTM**, repeat [Phases 1 and 2](../README.md#phase-1--create-the-vm) for `dc-server-01` and `dc-monitor-01` using fresh installations, not copies of the running VM's disk.

1. Use ARM64 Linux virtualization with the QEMU backend and Shared Network.
2. Allocate **2 vCPUs, 2 GB RAM, and 20 GB disk per VM**. Three such VMs use 6 GB configured guest RAM; allow additional host and hypervisor memory and disk space.
3. Set the distinct hostname during each install and install OpenSSH.
4. Detach the installer ISO and boot the installed system.
5. Repeat Phase 3's inspection and [Phase 6's key-based SSH setup](phase-06-remote-ssh.md) on each VM.
6. Record each management address. Do not assume they are `.4` and `.5`.

From the **Mac**, connect to each management address with SSH. From each **Ubuntu VM**, confirm its hostname, gateway, and DNS work. Install Phase 5's tools on both new VMs.

**Checkpoint:** Three distinct VMs boot, have unique NIC MAC addresses, and accept management SSH connections.

## Step 3 — Create two isolated UTM Host Networks

Shut down all three **Ubuntu VMs** cleanly:

```bash
sudo shutdown -h now
```

In **UTM on macOS**:

1. Open **UTM → Settings → Network → Host Networks**. Create two host-network entries and label them `lab-app` and `lab-monitor` where your version allows labels. Record each entry's UUID so you can distinguish them.
2. Edit `dc-network-01`. Keep the original Shared Network NIC. Add two `virtio-net-pci` network devices in **Host Only** mode: select the `lab-app` Host Network on one and `lab-monitor` on the other.
3. Edit `dc-server-01`. Keep Shared Network and add one Host Only NIC assigned to the **same `lab-app` Host Network UUID** used by the router.
4. Edit `dc-monitor-01`. Keep Shared Network and add one Host Only NIC assigned to the **same `lab-monitor` Host Network UUID** used by the router.
5. Record the MAC address and network assignment of every added device. Save the settings and start the VMs.

UTM's named Host Networks have no DHCP and are isolated from the Mac; you will configure static guest addresses. Selecting Host Only without selecting a common Host Network can leave VMs isolated from one another. Separate Emulated VLAN devices also do not create a shared VM-to-VM segment. See [UTM Host Networks](https://docs.getutm.app/preferences/macos/#host-networks) and [network-device settings](https://docs.getutm.app/settings-qemu/devices/network/network/).

If the Host Networks controls are absent, use a UTM version/backend that supports this documented feature before proceeding. Do not substitute a home/work LAN bridge for these isolated networks.

## Step 4 — Map Linux interfaces to the UTM devices

On **each Ubuntu VM**, at the console:

```bash
hostname
ip -br link
ip -br -4 addr
ip -4 route
sudo netplan get
```

Match MAC addresses with Step 3. The `enp0s2`/`enp0s3` names in this guide are examples, not guaranteed device names. Record the management, application, and monitoring interfaces before editing any configuration.

The guides assume `renderer: networkd`. Inspect existing `/etc/netplan/*.yaml`: management DHCP should match only the management NIC. If an existing stanza matches all Ethernet devices, narrow it to the management NIC first; otherwise it can also configure the private NICs. Do this through the console with a backup.

## Step 5 — Back up and install the private-interface configuration

Use [the configuration copy steps](../configurations/README.md#copy-examples-to-a-vm) to put the examples under `~/dc-lab-files/configurations/` on each VM.

On **each Ubuntu VM**:

```bash
LAB_NET_BACKUP="/root/dc-lab-netplan-$(date +%Y%m%d-%H%M%S)"
sudo mkdir -p "$LAB_NET_BACKUP"
sudo cp -a /etc/netplan/. "$LAB_NET_BACKUP/"
printf 'Netplan backup: %s\n' "$LAB_NET_BACKUP"
```

Record the backup path. Copy **only the file matching the current hostname**. For example, on **`dc-network-01`**:

```bash
cp ~/dc-lab-files/configurations/netplan/dc-network-01.yaml ~/dc-lab-private.yaml
nano ~/dc-lab-private.yaml
sudo install -m 600 ~/dc-lab-private.yaml /etc/netplan/60-dc-lab.yaml
```

On `dc-server-01`, use `dc-server-01.yaml`; on `dc-monitor-01`, use `dc-monitor-01.yaml`. In `nano`, replace interface names with your actual mapping; save with **Ctrl+O**, Enter, then **Ctrl+X**. Read the resulting file before installing it.

The [router example](../configurations/netplan/dc-network-01.yaml) adds two connected subnets. The [server example](../configurations/netplan/dc-server-01.yaml) adds a route to `10.10.30.0/24` through `10.10.20.1`. The [monitor example](../configurations/netplan/dc-monitor-01.yaml) supplies the return route to `10.10.20.0/24` through `10.10.30.1`.

Existing management DHCP stays in its original file. Do not add a second default gateway or DNS configuration to the private interfaces.

## Step 6 — Try the configuration and verify both paths

On **each VM at the UTM console**:

```bash
sudo netplan generate
sudo netplan try --timeout 120
```

Fix any generation errors before continuing. While the trial prompt is open, use a second console/SSH session to check:

```bash
ip -br -4 addr
ip -4 route
getent ahostsv4 ubuntu.com
```

Test Mac-to-VM management SSH as well. Confirm the trial only when the expected addresses, management connectivity, and routes are correct. Then validate that the private settings persist after a controlled reboot.

Do not rely on trial timeout alone for recovery. If settings fail, use the console, move the new file out of `/etc/netplan/`, restore any existing files you edited from the recorded backup, and run `sudo netplan apply`. Verify the files on disk before rebooting. See [Netplan's current trial behavior and caveats](https://netplan.readthedocs.io/en/stable/netplan-try/).

## Step 7 — Enable forwarding on the router only

On **`dc-network-01`**:

```bash
sysctl net.ipv4.ip_forward
sudo install -m 644 ~/dc-lab-files/configurations/sysctl/90-dc-lab-router.conf \
  /etc/sysctl.d/90-dc-lab-router.conf
sudo sysctl -p /etc/sysctl.d/90-dc-lab-router.conf
sysctl net.ipv4.ip_forward
```

Expect `net.ipv4.ip_forward = 1`. Record the previous value for rollback. Do not enable forwarding on the server or monitor. Phase 8 restricts forwarded traffic with UFW; before that phase, do not expose these dedicated VMs to other networks.

## Step 8 — Test directly connected neighbors first

On **`dc-server-01`**:

```bash
ping -c 4 10.10.20.1
ip neigh show
```

On **`dc-monitor-01`**:

```bash
ping -c 4 10.10.30.1
ip neigh show
```

If either fails, compare interface state, MAC/Host Network assignment, address/prefix, and any existing firewall rules. Fix that local segment before testing cross-subnet routing.

## Step 9 — Verify routing in both directions

On **`dc-server-01`**:

```bash
ip route get 10.10.30.10
ping -c 4 10.10.30.10
traceroute -n -I -m 5 -w 1 10.10.30.10
```

On **`dc-monitor-01`**:

```bash
ip route get 10.10.20.10
ping -c 4 10.10.20.10
traceroute -n -I -m 5 -w 1 10.10.20.10
```

Expect the private router address as the next hop, with the private source address selected. The routing tests must target the **10.10.x.x addresses**; successful tests between management DHCP addresses would bypass this routed path. Capture on the router if traceroute replies are filtered.

If cross-subnet pings fail after both local gateways work, check forwarding, both static routes, source address selection, and firewall forwarding rules. Never fix a missing return route by guessing at NAT.

## Step 10 — Install the test service

On the fresh **`dc-server-01`** VM:

```bash
sudo apt update
sudo apt install -y nginx
printf 'dc-server-01 lab service\n' | sudo tee /var/www/html/index.html
sudo nginx -t
sudo systemctl enable --now nginx
curl --max-time 5 http://127.0.0.1/
sudo ss -lntp '( sport = :80 )'
```

From **`dc-monitor-01`**:

```bash
curl --connect-timeout 3 --max-time 5 http://10.10.20.10/
```

**Checkpoint:** The response identifies `dc-server-01`, both routed directions work, and all VMs retain management SSH and package access. Update the [address plan](../architecture/ip-address-plan.md) with your actual NIC names and DHCP addresses.

## Recovery

For private-network changes, restore the affected Netplan backup through the UTM console. To undo router forwarding, move `/etc/sysctl.d/90-dc-lab-router.conf` out of that directory and restore the recorded value with `sudo sysctl -w net.ipv4.ip_forward=0` if it was originally zero. Removing a sysctl file alone does not reset the running value. Keep the management NIC unchanged throughout.

Reference: [Netplan addresses and static routes](https://netplan.readthedocs.io/en/stable/netplan-yaml/).
