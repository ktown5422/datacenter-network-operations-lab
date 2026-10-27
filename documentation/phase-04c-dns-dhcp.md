# Phase 4C — DNS and DHCP troubleshooting

[Previous: routing and ARP](../README.md#phase-4b--ipv4-subnetting-routes-and-arp) · [Next: networking toolkit](phase-05-networking-toolkit.md)

**Run on:** `dc-network-01`, inside Ubuntu. Use the UTM console for the failure and renewal exercises. **Start with:** Phases 1–4B complete and working IP connectivity.

**Goal:** Separate address assignment, IP routing, and name resolution, then recover from a local DNS resolver failure.

## Step 1 — Identify the management interface

```bash
ip -br addr
ip -4 route show default
LAB_MGMT_IF=enp0s1
networkctl status "$LAB_MGMT_IF" --no-pager
```

Replace `enp0s1` with the interface carrying the default route. In `networkctl`, identify the assigned address, gateway, DNS servers, and DHCP information. These guides assume Ubuntu Server's `systemd-networkd` renderer; if the interface is unmanaged, check `sudo netplan get` and use the matching network manager before continuing.

## Step 2 — Inspect DHCP and the lease

```bash
sudo netplan get
LAB_IFINDEX=$(cat "/sys/class/net/$LAB_MGMT_IF/ifindex")
sudo cat "/run/systemd/netif/leases/$LAB_IFINDEX"
journalctl -u systemd-networkd -b --no-pager -n 60
```

The runtime lease file is specific to `systemd-networkd`; if absent, use the status and journal rather than inventing a lease value. Record the DHCP server, address, router, DNS, and lease timing where available. DHCP provides configuration; it does not forward your internet traffic.

**Checkpoint:** Explain which values came from DHCP and which interface owns the default route.

## Step 3 — Inspect and test DNS

```bash
sudo apt update
sudo apt install -y dnsutils
resolvectl status
readlink -f /etc/resolv.conf
getent ahostsv4 ubuntu.com
resolvectl query ubuntu.com
dig ubuntu.com A +time=2 +tries=1
```

`getent` follows the system's name service configuration, including `/etc/hosts`. `dig` makes DNS queries. A `127.0.0.53` server is the local systemd-resolved stub; the upstream DNS servers are shown by `resolvectl`.

Compare with IP-only tests, using the actual gateway from Step 1:

```bash
ping -c 4 192.168.64.1
ping -c 4 1.1.1.1
```

**Checkpoint:** IP connectivity and DNS resolution both work. ICMP filtering can affect ping independently of DNS.

## Step 4 — Create a controlled DNS failure

Do this at the **UTM console**, before other workloads depend on this VM. First confirm `/etc/resolv.conf` uses the systemd-resolved stub and that the service is active:

```bash
systemctl is-active systemd-resolved
readlink -f /etc/resolv.conf
sudo resolvectl flush-caches
```

If the file does not resolve to `/run/systemd/resolve/stub-resolv.conf`, stop here and document the different resolver setup; this drill depends on the stub. Keep the recovery command in Step 5 ready.

```bash
sudo systemctl mask --runtime --now systemd-resolved.service
timeout 10 getent ahostsv4 ubuntu.com
timeout 10 resolvectl query ubuntu.com
ping -c 4 1.1.1.1
systemctl status systemd-resolved --no-pager
```

The temporary runtime mask stops the resolver and prevents a client request from activating it again during the drill. Expect DNS through the stopped stub to fail while the previously working IP-only path remains available. A cached or `/etc/hosts` result can still satisfy some lookups; use the resolver service status and `resolvectl` error to interpret that result. This simulates a **local resolver service outage**, not an upstream DNS or DHCP outage.

## Step 5 — Recover and retest

```bash
sudo systemctl unmask --runtime systemd-resolved.service
sudo systemctl start systemd-resolved
sudo resolvectl flush-caches
systemctl is-active systemd-resolved
resolvectl status "$LAB_MGMT_IF"
getent ahostsv4 ubuntu.com
dig ubuntu.com A +time=2 +tries=1
```

**Checkpoint:** The runtime mask is removed, the resolver is active, and the same name resolves again. If the drill is interrupted, run both recovery commands before continuing. Record symptom → IP-only comparison → service diagnosis → recovery → retest.

## Step 6 — Renew DHCP without changing persistent settings

At the **UTM console**, record the current address, then request renewal:

```bash
ip -4 addr show dev "$LAB_MGMT_IF"
sudo networkctl renew "$LAB_MGMT_IF"
networkctl status "$LAB_MGMT_IF" --no-pager
ip -4 route
getent ahostsv4 ubuntu.com
```

A renewal may retain the same address. Wait for network configuration to settle, then record the resulting address and gateway. If it changes, use the new address for SSH. If renewal fails, inspect UTM's Shared Network setting and `journalctl -u systemd-networkd -b`; do not assign a guessed address on the management network.

**Finish:** DNS is restored, DHCP configuration is understood, and baseline connectivity still works. Reuse these observations for the DNS incident in Phase 11.

References: [systemd networkctl](https://www.freedesktop.org/software/systemd/man/latest/networkctl.html), [systemd resolvectl](https://www.freedesktop.org/software/systemd/man/latest/resolvectl.html), [systemctl service control and runtime masks](https://www.freedesktop.org/software/systemd/man/latest/systemctl.html).
