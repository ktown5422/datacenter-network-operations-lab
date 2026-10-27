# Phase 8 — Firewall and access control

[Previous: multi-server networking](phase-07-multi-server-networking.md) · [Next: packet analysis and performance](phase-09-packet-performance.md)

**Run on:** all three VMs as labeled. **Start with:** Phase 7's working routes and NGINX service. Keep a signed-in UTM console on each VM. These steps assume dedicated lab VMs without a pre-existing firewall policy to preserve.

**Goal:** Separate host input rules from router forwarding rules, allow required traffic, then diagnose a blocked HTTP connection.

## Step 1 — Inspect before changing rules

On **each VM**:

```bash
sudo apt update
sudo apt install -y ufw
sudo ufw status verbose
sudo nft list ruleset
sudo ss -lntup
```

Record the existing state. If another tool already manages an active ruleset, stop and reconcile that policy rather than layering UFW on top. Do not run `ufw reset` to discard unknown rules.

## Step 2 — Allow management SSH before enabling UFW

On **each VM**, set the actual management NIC and subnet from the Phase 7 address plan:

```bash
LAB_MGMT_IF=enp0s1
LAB_MGMT_NET=192.168.64.0/24
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow in on "$LAB_MGMT_IF" from "$LAB_MGMT_NET" to any port 22 proto tcp
```

This permits lab management sources on that interface. It does not expose SSH to every source/interface. Leave IPv6 handling enabled in UFW; the private routing exercises use IPv4, while the default incoming policy also protects unlisted IPv6 services.

## Step 3 — Allow the routed application path

On **`dc-network-01`**, substitute the actual private NIC names:

```bash
LAB_APP_IF=enp0s2
LAB_MON_IF=enp0s3
sudo ufw default deny routed
sudo ufw route allow in on "$LAB_MON_IF" out on "$LAB_APP_IF" \
  from 10.10.30.0/24 to 10.10.20.0/24
sudo ufw route allow in on "$LAB_APP_IF" out on "$LAB_MON_IF" \
  from 10.10.20.0/24 to 10.10.30.0/24
```

These forwarding rules allow the two isolated lab subnets to communicate. They do not permit forwarding to/from the management NIC. Host-service restrictions are still enforced by each endpoint's input rules.

In **`/etc/ufw/sysctl.conf` on the router**, ensure the IPv4 forwarding setting is enabled, consistent with Phase 7:

```bash
sudoedit /etc/ufw/sysctl.conf
```

Uncomment or set `net/ipv4/ip_forward=1`; leave unrelated settings intact. UFW reads this file when applying its firewall. Keep the Phase 7 sysctl file as the persistent router setting too.

On **`dc-server-01`**:

```bash
LAB_APP_IF=enp0s2
sudo ufw allow in on "$LAB_APP_IF" from 10.10.30.10 to any port 80 proto tcp
```

No private inbound application rule is needed on `dc-monitor-01` for the HTTP client: replies to its established connections are tracked by UFW. Monitoring and throughput ports are added in later phases.

## Step 4 — Enable and verify one VM at a time

On **each VM through its UTM console**, enable UFW and inspect the result:

```bash
sudo ufw enable
sudo ufw status verbose
```

Before moving to the next VM, open a **new SSH session from the Mac** to confirm management access. An already-open session alone is not a sufficient access test.

On the **router**:

```bash
sysctl net.ipv4.ip_forward
sudo ufw status numbered
```

On **`dc-monitor-01`**:

```bash
ping -c 4 10.10.20.10
curl --connect-timeout 3 --max-time 5 http://10.10.20.10/
```

**Checkpoint:** UFW is active on each VM, new management SSH connections work, the router still forwards between the lab subnets, and HTTP returns the lab service page. UFW's packaged before-rules normally permit selected ICMP messages; TCP port rules do not describe the entire ICMP policy.

## Step 5 — Block only the test HTTP flow

On **`dc-server-01`**, at the console:

```bash
sudo ufw logging low
sudo ufw insert 1 deny in on "$LAB_APP_IF" from 10.10.30.10 to any port 80 proto tcp
sudo ufw status numbered
sudo systemctl is-active nginx
curl --max-time 5 http://127.0.0.1/
```

The first matching deny precedes the normal allow rule. From **`dc-monitor-01`**, start a fresh HTTP request:

```bash
curl --connect-timeout 3 --max-time 5 http://10.10.20.10/
```

Expect failure while local HTTP on the server still succeeds. On the server:

```bash
sudo ss -lntp '( sport = :80 )'
sudo journalctl -k --since '5 minutes ago' --no-pager
```

Look for a UFW block with the expected source, destination, and port. Logging is rate-limited; no log line alone does not prove a packet passed. Compare listener state, route, and rule order.

## Step 6 — Remove the injected rule and retest

On **`dc-server-01`**:

```bash
sudo ufw delete deny in on "$LAB_APP_IF" from 10.10.30.10 to any port 80 proto tcp
sudo ufw status numbered
```

From **`dc-monitor-01`**:

```bash
curl --connect-timeout 3 --max-time 5 http://10.10.20.10/
```

**Checkpoint:** HTTP succeeds again, the temporary deny is gone, and UFW remains active. Record the failure and recovery for the blocked-service incident in Phase 11.

## Recovery

If management SSH is blocked, use the UTM console to check the actual NIC, source subnet, and TCP 22 allow rule. Correct that rule with `sudo ufw allow in on ...` using Step 2's full syntax, then test a new connection. Remove only the rule introduced by the drill; do not flush the complete firewall. Recheck router forwarding if a local service works but routed access fails.

References: [Ubuntu firewall guide](https://ubuntu.com/server/docs/how-to/security/firewalls/), [UFW rule and route syntax](https://manpages.ubuntu.com/manpages/resolute/man8/ufw.8.html).
