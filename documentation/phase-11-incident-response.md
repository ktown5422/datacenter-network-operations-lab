# Phase 11 — Five controlled incident drills

[Previous: monitoring](phase-10-monitoring.md) · [Next: final verification](phase-12-final-verification.md)

**Run on:** the dedicated lab VMs and Mac, as labeled. **Start with:** all Phase 10 targets healthy, HTTP working, and UTM console login available. Run **one drill at a time** and restore its baseline before the next.

**Goal:** Practice symptom → scope → diagnostics → root cause → resolution → verification. The procedures below describe simulations; fill records with your own observed results.

## Step 1 — Prepare five records and a baseline

Copy [the incident template](../incidents/TEMPLATE.md) into your notes five times, using these filenames:

| Record | Drill | Fault target |
|---|---|---|
| `INC-003-ssh-failure.md` | SSH unavailable | Server's SSH listener |
| `INC-004-dns-failure.md` | DNS lookup failure | Server's local resolver |
| `INC-005-unreachable-host.md` | Host unreachable on the lab path | Monitor's route to the server |
| `INC-006-blocked-service.md` | HTTP blocked | Server's UFW input rule |
| `INC-007-performance-degradation.md` | Reduced throughput | Server's private egress queue |

On **`dc-server-01`**:

```bash
date -Is
hostname
ip -br -4 addr
sudo ufw status numbered
sudo ss -lntp
```

On **`dc-monitor-01`**:

```bash
ping -c 4 10.10.20.10
curl --connect-timeout 3 --max-time 5 http://10.10.20.10/
curl -fsS -G http://127.0.0.1:9090/api/v1/query --data-urlencode 'query=up{job="node"}'
```

Record the time, current management addresses, and healthy starting results. Use `date -Is` at each failure/recovery to build a timeline.

## Step 2 — INC-003: SSH listener failure

### Inject at the server's UTM console

Record whether socket activation is active before stopping anything:

```bash
systemctl is-active ssh.socket
systemctl is-active ssh.service
sudo ss -lntp '( sport = :22 )'
```

An inactive/nonexistent unit can return a nonzero status; record it. Stop the socket if installed, then stop the service:

```bash
if systemctl cat ssh.socket >/dev/null 2>&1; then
  sudo systemctl stop ssh.socket
fi
sudo systemctl stop ssh.service
sudo ss -lntp '( sport = :22 )'
```

Do this at the console because existing SSH sessions can end. Stopping only `ssh.service` can allow an active socket to start it again, which is why both are considered.

### Diagnose from the Mac

In **macOS Terminal**, use the server's current management address:

```bash
LAB_USER=your_ubuntu_username
LAB_SERVER_MGMT_IP=192.168.64.4
ssh -i ~/.ssh/id_ed25519_dc_lab -o ConnectTimeout=5 \
  "$LAB_USER@$LAB_SERVER_MGMT_IP" hostname
```

Expect failure. On the server console, inspect `systemctl status ssh.service ssh.socket --no-pager`, the missing TCP 22 listener, and `sudo journalctl -u ssh.service --since '5 minutes ago' --no-pager`. From the monitor, verify HTTP still works: this is a service-specific failure, not proof that the VM is down.

### Recover

If `ssh.socket` was active before the drill, run **`sudo systemctl start ssh.socket`** at the console. Otherwise run **`sudo systemctl start ssh.service`**. Restore the original activation method rather than starting both blindly.

Repeat the same Mac SSH command and confirm the server hostname. Record the listener and successful login. Do not change authentication policy for this drill.

## Step 3 — INC-004: DNS resolver failure

On the **server console**, inspect `readlink -f /etc/resolv.conf` and confirm the systemd-resolved stub setup described in [Phase 4C](phase-04c-dns-dhcp.md#step-4--create-a-controlled-dns-failure). With the resolver initially active:

```bash
sudo resolvectl flush-caches
sudo systemctl mask --runtime --now systemd-resolved.service
timeout 10 getent ahostsv4 ubuntu.com
timeout 10 resolvectl query ubuntu.com
ping -c 4 1.1.1.1
systemctl status systemd-resolved --no-pager
```

The temporary runtime mask prevents client-triggered activation from undoing the drill. Compare failed DNS lookups with the previously working IP-only path. Inspect the resolver's service state and logs:

```bash
sudo journalctl -u systemd-resolved --since '5 minutes ago' --no-pager
```

Recover on the **server**:

```bash
sudo systemctl unmask --runtime systemd-resolved.service
sudo systemctl start systemd-resolved
sudo resolvectl flush-caches
getent ahostsv4 ubuntu.com
dig ubuntu.com A +time=2 +tries=1
```

Record the actual failure and recovery. If a cached lookup succeeds, explain why instead of claiming a failure you did not see. Confirm the runtime mask is removed and resolver state is active before proceeding. Use the same recovery commands if interrupted.

## Step 4 — INC-005: Unreachable host from a bad route

This drill affects only the **monitor's IPv4 route to `10.10.20.10`**, leaving management SSH and other addresses intact. On **`dc-monitor-01`**:

```bash
ip route get 10.10.20.10
ip route show exact 10.10.20.10/32
```

Proceed only if no existing `/32` route must be preserved. Add a more specific unreachable route:

```bash
sudo ip route add unreachable 10.10.20.10/32
ping -c 4 10.10.20.10
curl --connect-timeout 3 --max-time 5 http://10.10.20.10/
ip route get 10.10.20.10
ip route show
ping -c 4 10.10.30.1
```

The explicit `/32` takes precedence over the valid `/24` route. Expect the server path to fail while the monitor's local router address still responds. At the **server console**, verify `ip -br addr` and local HTTP to show the server is still running.

Recover on the **monitor**:

```bash
sudo ip route del unreachable 10.10.20.10/32
ip route get 10.10.20.10
ping -c 4 10.10.20.10
curl --connect-timeout 3 --max-time 5 http://10.10.20.10/
```

Record the selected next hop before/after. This models a client-side route problem, not a VM power failure. The temporary route is not written into Netplan.

## Step 5 — INC-006: Blocked HTTP service

On **`dc-server-01`**:

```bash
LAB_APP_IF=enp0s2
sudo ufw insert 1 deny in on "$LAB_APP_IF" from 10.10.30.10 to any port 80 proto tcp
sudo ufw status numbered
sudo ss -lntp '( sport = :80 )'
curl --max-time 5 http://127.0.0.1/
```

Use the actual application NIC. On **`dc-monitor-01`**:

```bash
curl --connect-timeout 3 --max-time 5 http://10.10.20.10/
```

Compare remote failure with the still-working server listener/local HTTP. Use `sudo journalctl -k --since '5 minutes ago' --no-pager` on the server and inspect rule order. If monitoring has run long enough, the HTTP alert may also fire; record that separately from exporter health.

Recover on the **server**:

```bash
sudo ufw delete deny in on "$LAB_APP_IF" from 10.10.30.10 to any port 80 proto tcp
```

Repeat the monitor's HTTP request. Confirm both HTTP and any resulting alert recover. See [Phase 8](phase-08-firewall-controls.md) for the full rule-order explanation.

## Step 6 — INC-007: Reduced throughput

Use only the **server's private application NIC**, never its management NIC. On **`dc-server-01`**:

```bash
LAB_APP_IF=enp0s2
sudo tc qdisc show dev "$LAB_APP_IF"
sudo ufw allow in on "$LAB_APP_IF" from 10.10.30.10 to any port 5201 proto tcp
iperf3 -s -1 -B 10.10.20.10
```

Record the existing qdisc. Continue only on a default, uncustomized lab queue; preserve any custom traffic-control configuration instead of replacing it.

On **`dc-monitor-01`**, establish the reverse-direction baseline (the server sends):

```bash
iperf3 -c 10.10.20.10 -B 10.10.30.10 -t 10 -R
```

After the one-off server exits, inject a bounded rate/latency constraint on **`dc-server-01`**:

```bash
sudo tc qdisc add dev "$LAB_APP_IF" root netem delay 40ms rate 10mbit
sudo tc -s qdisc show dev "$LAB_APP_IF"
iperf3 -s -1 -B 10.10.20.10
```

If adding the qdisc fails, inspect the existing queue; do not use `replace` to overwrite an unknown setup. Repeat the same `iperf3 -R` command on the **monitor**. Record throughput, latency/retransmission changes, `tc -s` statistics, and guest/host load. Expect reduced server-to-monitor throughput; do not expect an exact 10 Mbit/s application result because of protocol overhead and scheduling.

Recover on the **server**, immediately after the measurement:

```bash
sudo tc qdisc del dev "$LAB_APP_IF" root
sudo tc qdisc show dev "$LAB_APP_IF"
iperf3 -s -1 -B 10.10.20.10
```

Repeat the monitor's reverse test to compare recovery with the baseline. The deleted test queue returns to the device's default queue behavior. If a drill is interrupted, use the console to remove the netem root queue before any other exercise.

After the final one-off server exits, on the **server**:

```bash
sudo ufw delete allow in on "$LAB_APP_IF" from 10.10.30.10 to any port 5201 proto tcp
sudo ss -lntp '( sport = :5201 )'
curl --max-time 5 http://127.0.0.1/
```

## Step 7 — Close the records

For each simulation, complete the template with observed commands/results, the cause supported by those results, recovery, and a repeated client test. Include what would need escalation if your fix failed. Link any capture or metric graph that clarifies the diagnosis.

**Checkpoint:** Five records are filled, each fault has been reversed, SSH/DNS/HTTP work, monitoring is healthy, no temporary `/32` route or deny rule remains, and no netem queue or iperf3 listener remains.

References: [UFW manual](https://manpages.ubuntu.com/manpages/resolute/man8/ufw.8.html), [Linux IP route manual](https://man7.org/linux/man-pages/man8/ip-route.8.html), [netem manual](https://man7.org/linux/man-pages/man8/tc-netem.8.html).
