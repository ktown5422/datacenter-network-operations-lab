# Phase 10 — Prometheus, Node Exporter, and Grafana

[Previous: packet analysis](phase-09-packet-performance.md) · [Next: incident drills](phase-11-incident-response.md)

**Run on:** all VMs for Node Exporter; `dc-monitor-01` for Prometheus, Blackbox Exporter, and Grafana. **Start with:** routed HTTP and UFW working. The dashboard and Prometheus UI are accessed through an SSH tunnel from the Mac.

**Goal:** Graph host metrics, distinguish scrape health from HTTP availability, and observe a safe alert fire and resolve. This exercise uses local alert pages; it does not configure email or other external notifications.

## Step 1 — Install Node Exporter on all three VMs

On **each Ubuntu VM**:

```bash
sudo apt update
sudo apt install -y software-properties-common
sudo add-apt-repository -y universe
sudo apt update
sudo apt install -y prometheus-node-exporter
sudo systemctl enable --now prometheus-node-exporter
systemctl status prometheus-node-exporter --no-pager
curl -fsS --max-time 5 http://127.0.0.1:9100/metrics -o /tmp/dc-lab-node-metrics.txt
head -n 12 /tmp/dc-lab-node-metrics.txt
```

The package selects the VM's architecture automatically. Inspect `systemctl cat prometheus-node-exporter` and `ss -lntp` to confirm the service and listener. The examples expect TCP 9100; if you already customized its bind address, reconcile it before continuing.

## Step 2 — Allow only the monitor to scrape the private interfaces

On **`dc-network-01`**, use its actual monitoring NIC:

```bash
LAB_MON_IF=enp0s3
sudo ufw allow in on "$LAB_MON_IF" from 10.10.30.10 to any port 9100 proto tcp
```

On **`dc-server-01`**, use its actual application NIC:

```bash
LAB_APP_IF=enp0s2
sudo ufw allow in on "$LAB_APP_IF" from 10.10.30.10 to any port 9100 proto tcp
```

The monitor scrapes its own exporter through loopback. Do not add inbound 9100 rules on the management NICs. From **`dc-monitor-01`**:

```bash
curl -fsS --max-time 5 http://10.10.30.1:9100/metrics -o /tmp/router-metrics.txt
curl -fsS --max-time 5 http://10.10.20.10:9100/metrics -o /tmp/server-metrics.txt
curl -fsS --max-time 5 http://127.0.0.1:9100/metrics -o /tmp/monitor-metrics.txt
```

**Checkpoint:** All three metric endpoints respond from the monitor. If only the remote server fails, check the routed path and its firewall rather than changing the exporter on the monitor.

## Step 3 — Install the monitoring packages

On **`dc-monitor-01`**:

```bash
sudo apt install -y prometheus prometheus-blackbox-exporter
command -v prometheus promtool prometheus-blackbox-exporter
systemctl cat prometheus
systemctl cat prometheus-blackbox-exporter
LAB_MON_BACKUP="/root/dc-lab-monitoring-$(date +%Y%m%d-%H%M%S)"
sudo mkdir -p "$LAB_MON_BACKUP"
sudo cp -a /etc/prometheus "$LAB_MON_BACKUP/"
printf 'Monitoring backup: %s\n' "$LAB_MON_BACKUP"
```

Record that backup path. These steps use Ubuntu's `/usr/bin/prometheus` and `/usr/bin/prometheus-blackbox-exporter` binaries and the `prometheus` service account. If your installation has different unit names, binaries, or an existing monitoring configuration, reconcile them before applying the examples.

## Step 4 — Install the scrape, probe, and alert configurations

Copy the repository's examples to the monitor using [the configuration transfer instructions](../configurations/README.md#copy-examples-to-a-vm), then on **`dc-monitor-01`**:

```bash
sudo install -m 644 ~/dc-lab-files/configurations/prometheus/prometheus.yml /etc/prometheus/prometheus.yml
sudo install -m 644 ~/dc-lab-files/configurations/prometheus/lab-alerts.yml /etc/prometheus/lab-alerts.yml
sudo install -m 644 ~/dc-lab-files/configurations/prometheus/blackbox.yml /etc/prometheus/blackbox.yml
promtool check rules /etc/prometheus/lab-alerts.yml
promtool check config /etc/prometheus/prometheus.yml
```

The [scrape configuration](../configurations/prometheus/prometheus.yml) collects three node endpoints every 15 seconds and uses a local Blackbox Exporter to probe the server's HTTP URL. The [alert rules](../configurations/prometheus/lab-alerts.yml) detect failed node scrapes and failed HTTP probes after 30 seconds.

`up{job="node"}` describes whether Prometheus could scrape an exporter. `probe_success{job="http"}` describes whether the HTTP probe succeeded. `up{job="http"}` alone only confirms that Prometheus contacted the probe exporter, not that NGINX was reachable.

## Step 5 — Bind the monitoring APIs to loopback

On **`dc-monitor-01`**, create service overrides so the web APIs are available only locally:

```bash
sudo install -d -o prometheus -g prometheus /var/lib/prometheus/metrics2
sudo mkdir -p /etc/systemd/system/prometheus.service.d
sudo mkdir -p /etc/systemd/system/prometheus-blackbox-exporter.service.d
sudoedit /etc/systemd/system/prometheus.service.d/dc-lab.conf
```

Put this in the Prometheus override:

```ini
[Service]
ExecStart=
ExecStart=/usr/bin/prometheus --config.file=/etc/prometheus/prometheus.yml --storage.tsdb.path=/var/lib/prometheus/metrics2 --storage.tsdb.retention.time=7d --web.listen-address=127.0.0.1:9090
```

Edit the Blackbox Exporter override:

```bash
sudoedit /etc/systemd/system/prometheus-blackbox-exporter.service.d/dc-lab.conf
```

```ini
[Service]
ExecStart=
ExecStart=/usr/bin/prometheus-blackbox-exporter --config.file=/etc/prometheus/blackbox.yml --web.listen-address=127.0.0.1:9115
```

Then start the services:

```bash
sudo systemctl daemon-reload
sudo systemctl enable prometheus prometheus-blackbox-exporter
sudo systemctl restart prometheus-blackbox-exporter prometheus
systemctl status prometheus prometheus-blackbox-exporter --no-pager
sudo ss -lntp '( sport = :9090 or sport = :9115 )'
curl -fsS --max-time 5 http://127.0.0.1:9090/-/ready
curl -fsS --max-time 10 -G http://127.0.0.1:9115/probe \
  --data-urlencode 'target=http://10.10.20.10/' --data-urlencode 'module=http_2xx'
```

Wait briefly and retry readiness if Prometheus is still starting. Expect loopback listeners and `probe_success 1`. If a service fails, inspect `journalctl -u prometheus -u prometheus-blackbox-exporter -b --no-pager -n 80` and validate the configuration before restarting again.

## Step 6 — Install Grafana from its official APT repository

On **`dc-monitor-01`**:

```bash
sudo apt install -y apt-transport-https wget gnupg
sudo mkdir -p /etc/apt/keyrings
sudo wget -O /etc/apt/keyrings/grafana.asc https://apt.grafana.com/gpg-full.key
sudo chmod 644 /etc/apt/keyrings/grafana.asc
printf '%s\n' 'deb [signed-by=/etc/apt/keyrings/grafana.asc] https://apt.grafana.com stable main' \
  | sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt update
sudo apt install -y grafana
sudo cp -a /etc/grafana/grafana.ini "$LAB_MON_BACKUP/grafana.ini"
sudoedit /etc/grafana/grafana.ini
```

In the existing `[server]` section, uncomment/set these keys. Do not add a duplicate section:

```ini
[server]
http_addr = 127.0.0.1
http_port = 3000
```

Install the [local Prometheus data source](../configurations/grafana/datasources.yml):

```bash
sudo install -m 644 ~/dc-lab-files/configurations/grafana/datasources.yml \
  /etc/grafana/provisioning/datasources/dc-lab.yml
sudo systemctl enable --now grafana-server
sudo systemctl restart grafana-server
systemctl status grafana-server --no-pager
curl -fsS --max-time 5 http://127.0.0.1:3000/api/health
sudo ss -lntp '( sport = :3000 )'
```

**Checkpoint:** Grafana's health endpoint responds and TCP 3000 is bound to `127.0.0.1`. If the local port is already used by another service, resolve that conflict before continuing. No inbound UFW rule for 3000, 9090, or 9115 is required.

## Step 7 — Open the dashboards through an SSH tunnel

In **macOS Terminal**:

```bash
LAB_USER=your_ubuntu_username
LAB_MONITOR_MGMT_IP=192.168.64.5
ssh -i ~/.ssh/id_ed25519_dc_lab -o ExitOnForwardFailure=yes -N \
  -L 127.0.0.1:3000:127.0.0.1:3000 \
  -L 127.0.0.1:9090:127.0.0.1:9090 "$LAB_USER@$LAB_MONITOR_MGMT_IP"
```

Use the monitor's **actual management IP**. Keep this terminal open. If a Mac port is occupied, change only the first port in that `-L` mapping (for example `3001`) and use the matching localhost URL.

1. In the Mac browser, open `http://127.0.0.1:9090/targets`. Expect three node targets, one Prometheus target, and one HTTP probe target to be up.
2. Open `http://127.0.0.1:9090` and use the **Query/Graph** page to evaluate `up{job="node"}`, then `probe_success{job="http"}`. Expect values of 1.
3. Open `http://127.0.0.1:3000`. On a fresh Grafana installation, sign in with its initial `admin`/`admin` login and set your own password when prompted. Do not record that password in the repository.
4. Open the data sources page (usually **Connections → Data sources**) and select **Lab Prometheus**. Use **Save & test** to confirm access to `http://127.0.0.1:9090` from the Grafana server.

## Step 8 — Build a small dashboard

In **Grafana**:

1. Choose **Dashboards → New → New dashboard → Add visualization**.
2. Select **Lab Prometheus**.
3. Switch the query editor to **Code**, paste a query from the table, and run it.
4. Set the panel title and unit, apply/save the panel, and repeat for the other rows. Select a time range such as the last 15 minutes; rate queries need multiple scrapes.
5. Save the dashboard as **Data Center Lab**. Export its JSON to your lab notes if you want to recreate it later.

| Panel | PromQL | Unit |
|---|---|---|
| Exporter scrape health | `up{job="node"}` | None; 1 means scrape succeeded |
| HTTP availability | `probe_success{job="http"}` | None; 1 means probe succeeded |
| CPU busy | `100 * (1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])))` | Percent (0–100) |
| Memory used | `100 * (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)` | Percent (0–100) |
| Root filesystem used | `100 * (1 - node_filesystem_avail_bytes{mountpoint="/",fstype!~"tmpfs\|overlay"} / node_filesystem_size_bytes{mountpoint="/",fstype!~"tmpfs\|overlay"})` | Percent (0–100) |
| Network receive rate | `8 * rate(node_network_receive_bytes_total{device!="lo"}[5m])` | Bits/sec |

Use the instance/host labels to distinguish VMs. A near-idle graph can be correct; generate a short lab HTTP or throughput test to observe a change, then let it settle.

## Step 9 — Trigger and recover a test alert

Keep `http://127.0.0.1:9090/alerts` open. On **`dc-server-01`**:

```bash
sudo systemctl stop prometheus-node-exporter
systemctl is-active prometheus-node-exporter
```

On **`dc-monitor-01`**, query scrape health:

```bash
curl -fsS -G http://127.0.0.1:9090/api/v1/query --data-urlencode 'query=up{job="node"}'
```

Allow about **45–75 seconds** for scraping and the rule's 30-second duration. Observe `LabNodeExporterDown` progress from pending to firing for the server.

Record the alert time and investigate service status, TCP 9100, and the firewall. Then recover on **`dc-server-01`**:

```bash
sudo systemctl start prometheus-node-exporter
systemctl is-active prometheus-node-exporter
```

Verify `up` returns to 1 and the alert becomes inactive after subsequent scrapes/evaluations. Also confirm HTTP availability remained 1 during this exporter-only failure; the VM and NGINX were still running.

**Checkpoint:** All three hosts have metrics, the HTTP probe succeeds, the dashboard shows CPU/memory/disk/network data, and a test alert fired and resolved. Close the Mac's SSH tunnel with Ctrl+C when finished.

## Recovery

If configuration changes break monitoring, restore the affected `/etc/prometheus` files from the recorded backup, validate with `promtool`, and restart the affected service. For a service-override problem, move only its new `dc-lab.conf` out of the corresponding `.service.d` directory, run `sudo systemctl daemon-reload`, and restart that service. Restore `grafana.ini` from the same backup if needed. Keep UFW active and use the UTM console or management SSH while troubleshooting.

References: [Node Exporter metrics](https://prometheus.io/docs/guides/node-exporter/), [Prometheus configuration](https://prometheus.io/docs/prometheus/latest/configuration/configuration/), [alert rules](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/), [Blackbox Exporter](https://github.com/prometheus/blackbox_exporter), [Grafana APT installation](https://grafana.com/docs/grafana/latest/setup-grafana/installation/debian/), [Grafana provisioning](https://grafana.com/docs/grafana/latest/administration/provisioning/).
