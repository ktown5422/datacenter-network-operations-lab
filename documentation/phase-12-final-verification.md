# Phase 12 — Final verification and lab notes

[Previous: incident drills](phase-11-incident-response.md) · [Return to README](../README.md)

**Run on:** all VMs and the Mac, as labeled. **Goal:** Repeat the checks from a clean state, confirm recovery and persistence, and leave notes you can use to rebuild the lab.

## Step 1 — Confirm the three VM identities

On **each Ubuntu VM**:

```bash
date -Is
hostnamectl
cat /etc/os-release
free -h
df -h /
ip -br -4 addr
ip -4 route
sudo ufw status verbose
```

Update the [address plan](../architecture/ip-address-plan.md) with actual management addresses and private NIC names. Confirm no duplicate static addresses and only management's default route. On the router, also run `sysctl net.ipv4.ip_forward` and expect 1.

## Step 2 — Repeat management, DNS, and SSH checks

On **each VM**, use its actual management gateway:

```bash
ping -c 4 192.168.64.1
ping -c 4 1.1.1.1
getent ahostsv4 ubuntu.com
systemctl is-active systemd-resolved
sudo ss -lntp '( sport = :22 )'
```

In **macOS Terminal**, repeat the key-based SSH command from Phase 6 for each management address and run `hostname` in the remote session. Compare any failed ping with TCP/DNS tests before classifying it as a loss of internet access.

## Step 3 — Verify the routed application path

On **`dc-server-01`**:

```bash
ip route get 10.10.30.10
ping -c 4 10.10.30.10
sudo nginx -t
systemctl is-active nginx
```

On **`dc-monitor-01`**:

```bash
ip route get 10.10.20.10
ping -c 4 10.10.20.10
curl --connect-timeout 3 --max-time 5 http://10.10.20.10/
```

**Checkpoint:** Both private routes use the Linux router, and the HTTP page identifies the server. UTM's Shared Network is still used for administration/downloads, not these cross-subnet tests.

## Step 4 — Check monitoring and its exposure

On **`dc-monitor-01`**:

```bash
systemctl is-active prometheus prometheus-blackbox-exporter grafana-server
promtool check rules /etc/prometheus/lab-alerts.yml
promtool check config /etc/prometheus/prometheus.yml
curl -fsS http://127.0.0.1:9090/-/ready
curl -fsS http://127.0.0.1:3000/api/health
curl -fsS -G http://127.0.0.1:9090/api/v1/query --data-urlencode 'query=up{job="node"}'
curl -fsS -G http://127.0.0.1:9090/api/v1/query --data-urlencode 'query=probe_success{job="http"}'
sudo ss -lntp '( sport = :3000 or sport = :9090 or sport = :9115 )'
```

Expect three healthy node scrapes, a successful HTTP probe, and loopback-only UI/probe listeners. Use the Phase 10 SSH tunnel to review the dashboard and confirm no drill alert remains active. On each VM, check `systemctl is-active prometheus-node-exporter`.

## Step 5 — Remove drill leftovers

On the **monitor**:

```bash
ip route show exact 10.10.20.10/32
```

The injected unreachable route should be absent. On the **server**, use the actual application NIC:

```bash
LAB_APP_IF=enp0s2
sudo ufw status numbered
sudo tc qdisc show dev "$LAB_APP_IF"
sudo ss -lntp '( sport = :5201 )'
```

Confirm the drill's HTTP deny and temporary 5201 allow rule are gone, there is no netem queue, and no iperf3 listener remains. Use the corresponding phase's recovery steps if needed; do not remove unrelated rules or queues.

## Step 6 — Verify persistence with controlled reboots

Do this after all transient faults are removed and configuration files are saved. Reboot **one VM at a time** from its console with `sudo reboot`:

1. Reboot `dc-server-01`; wait for it to return and repeat its identity, route, UFW, NGINX, exporter, and Mac SSH checks.
2. Reboot `dc-network-01`; expect brief cross-subnet interruption. After it returns, verify private NICs, forwarding = 1, UFW routed rules, and both routed pings.
3. Reboot `dc-monitor-01`; reopen the SSH tunnel, verify its return route, and repeat monitoring health queries.

DHCP management addresses may change, so confirm them at the console before reconnecting. Allow time for Prometheus scrapes/rate graphs to resume and temporary reboot alerts to resolve. If configuration is not persistent, return to the relevant Netplan, sysctl, UFW, or service-enablement step before continuing.

## Step 7 — Save useful lab notes

1. Keep the actual UTM network assignments, NIC/MAC mapping, IP plan, and route diagram together.
2. Save reviewed copies of your final Netplan, router sysctl, UFW rules, and monitoring configuration. Exclude passwords/private keys.
3. Keep the ARP/ICMP/DNS/TCP captures, directional throughput results, and the five completed incident records.
4. Add a short explanation for each tool and the results you can interpret.
5. Note any installation/version-specific differences so you can follow the walkthrough again without guessing.
6. Update [lessons learned](../lessons-learned.md) with what changed your understanding.

Screenshots are optional. The [lab checklist](completion-review.md) is a compact reference for repeating the verification checks.

## Step 8 — Finish or shut down

After successful checks, leave the VMs in their healthy baseline state or shut them down cleanly with `sudo shutdown -h now`. If making final VM package backups, copy the packages only after shutdown completes, as in Phase 3B. Close any Mac SSH tunnel with Ctrl+C.

**Final checkpoint:** You can follow the full sequence, explain the addressing/routing and service behavior, diagnose each controlled fault, restore the baseline, and repeat the checks after reboot.
