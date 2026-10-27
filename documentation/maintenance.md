# Keeping the lab current

[Return to README](../README.md) · [Final verification](phase-12-final-verification.md)

**Reference review date:** October 26, 2026. This is a review of the walkthrough and official references. Installed VM package versions and available updates must be checked on the VMs using the steps below.

## Current reference baseline

| Component | Reference checked | Walkthrough approach |
|---|---|---|
| Ubuntu Server ARM64 | [Ubuntu ARM downloads](https://ubuntu.com/download/server/arm) lists 26.04.1 LTS | Keep the 26.04 LTS baseline and apply its package updates. |
| UTM | [Official stable release](https://github.com/utmapp/UTM/releases/latest) points to 5.0.8 at the review date | Keep the QEMU backend and named Host Networks used in Phase 7. Check release notes before updating. |
| Netplan | [YAML reference](https://netplan.readthedocs.io/en/stable/netplan-yaml/) and [trial behavior](https://netplan.readthedocs.io/en/stable/netplan-try/) | Static private addresses, explicit routes, existing management DHCP, and console recovery. |
| Prometheus/exporters | [Ubuntu Prometheus package](https://packages.ubuntu.com/en/resolute/prometheus), [Blackbox Exporter package](https://packages.ubuntu.com/en/resolute/devel/prometheus-blackbox-exporter), and [Prometheus configuration](https://prometheus.io/docs/prometheus/latest/configuration/configuration/) | Use Ubuntu's APT package candidates and validate the examples with the installed `promtool`. |
| Grafana | [Official Debian/Ubuntu installation](https://grafana.com/docs/grafana/latest/setup-grafana/installation/debian/) | Stable APT repository with its signed keyring; loopback UI and a local Prometheus data source. |
| Wireshark | [Downloads](https://www.wireshark.org/download.html) and [display filters](https://www.wireshark.org/docs/wsug_html_chunked/ChWorkDisplayFilterSection.html) | Use the Mac application to inspect the saved lab captures. |

The Ubuntu repository's package version can differ from the newest upstream release. This walkthrough follows the Ubuntu-packaged Prometheus installation method. Switching to upstream binaries, another Ubuntu release, or another UTM backend changes that baseline and needs a separate compatibility review.

## Step 1 — Establish a healthy starting point

1. Remove the temporary faults from Phase 11 and verify the relevant Phase 12 checks.
2. Confirm console access and a recent VM backup using Phase 3B's shutdown-and-copy procedure.
3. Record VM hostnames, current management addresses, and the date. Work on one VM at a time.

## Step 2 — Check versions and available updates

On **each Ubuntu VM**:

```bash
date -Is
cat /etc/os-release
uname -r
dpkg --print-architecture
sudo apt update
apt list --upgradable
apt-cache policy openssh-server netplan.io ufw tcpdump iperf3 curl
```

On **`dc-monitor-01`**, after Phase 10, also run:

```bash
apt-cache policy prometheus prometheus-node-exporter prometheus-blackbox-exporter grafana
prometheus --version
promtool --version
```

Compare **Installed** and **Candidate** versions. `apt update` refreshes the package index; it does not install upgrades. An installed release name such as 26.04.1 does not, by itself, show whether every package update has been applied.

## Step 3 — Apply package updates

On the **current Ubuntu VM**, after reviewing the proposed package changes:

```bash
sudo apt upgrade
```

Read the summary and confirm when ready. Keep local configuration backups and review any configuration-file prompts rather than replacing your lab settings blindly. For major Grafana or Prometheus version changes, read their official upgrade/migration notes before proceeding. This maintenance step updates packages within the configured release; it does not require a distribution release upgrade.

If packages are deferred because of phased updates, check [Ubuntu's explanation](https://ubuntu.com/server/docs/explanation/software/about-apt-upgrade-and-phased-updates/) before attempting to override that behavior.

## Step 4 — Check whether a reboot is required

On the **updated Ubuntu VM**:

```bash
if [ -f /var/run/reboot-required ]; then
  cat /var/run/reboot-required
  if [ -f /var/run/reboot-required.pkgs ]; then
    cat /var/run/reboot-required.pkgs
  fi
fi
```

If requested, reboot through the console with `sudo reboot`, wait for the VM to return, and confirm its current management IP before reconnecting. Update/reboot the next VM only after the current VM's checks pass.

## Step 5 — Check the Mac applications

1. In **UTM → About UTM**, record the installed version and compare it with the official release page above. Read the release notes for compatibility changes.
2. Shut down all VMs cleanly before updating UTM; its release notes warn that suspended VMs may not resume correctly after an update. Keep the existing QEMU/Host Network assignments.
3. In **Wireshark → About Wireshark**, record its version. Use its update mechanism or official download page for application updates.
4. Reopen a saved capture after the update and confirm the display filters still work.

## Step 6 — Verify after maintenance

Repeat [Phase 12](phase-12-final-verification.md) for each affected VM: management SSH, DNS, private routes, forwarding/UFW, NGINX, and monitoring health. On **`dc-monitor-01`**, also run:

```bash
promtool check rules /etc/prometheus/lab-alerts.yml
promtool check config /etc/prometheus/prometheus.yml
curl -fsS http://127.0.0.1:9090/-/ready
curl -fsS http://127.0.0.1:3000/api/health
```

Allow monitoring targets and rate graphs to settle after reboots. Record package/application versions, any changes needed, and the successful checks in your notes. Close the maintenance work when the baseline is healthy again.

References: [Ubuntu package-management tutorial](https://ubuntu.com/server/docs/tutorial/managing-software/), [Grafana upgrade guide](https://grafana.com/docs/grafana/latest/upgrade-guide/), [Prometheus migration guide](https://prometheus.io/docs/prometheus/latest/migration/).
