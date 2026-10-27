# Configuration examples

These files support the walkthrough. Netplan interface names are examples; match them to the NICs on your VM before installing them. The management NIC keeps its existing DHCP configuration. Private lab interfaces have no default route or DNS server.

| Files | Destination | Used in |
|---|---|---|
| `netplan/dc-network-01.yaml` | Router: `/etc/netplan/60-dc-lab.yaml` | [Phase 7](../documentation/phase-07-multi-server-networking.md) |
| `netplan/dc-server-01.yaml` | Server: `/etc/netplan/60-dc-lab.yaml` | Phase 7 |
| `netplan/dc-monitor-01.yaml` | Monitor: `/etc/netplan/60-dc-lab.yaml` | Phase 7 |
| `sysctl/90-dc-lab-router.conf` | Router: `/etc/sysctl.d/90-dc-lab-router.conf` | Phase 7 |
| `prometheus/*.yml` | Monitor: `/etc/prometheus/` | [Phase 10](../documentation/phase-10-monitoring.md) |
| `grafana/datasources.yml` | Monitor: `/etc/grafana/provisioning/datasources/dc-lab.yml` | Phase 10 |

## Copy examples to a VM

After [Phase 6](../documentation/phase-06-remote-ssh.md), open **macOS Terminal** in this repository's root directory. Set the username and current management IP for the destination VM:

```bash
LAB_USER=your_ubuntu_username
LAB_IP=192.168.64.3
ssh -i ~/.ssh/id_ed25519_dc_lab "$LAB_USER@$LAB_IP" 'mkdir -p ~/dc-lab-files'
scp -r -i ~/.ssh/id_ed25519_dc_lab configurations "$LAB_USER@$LAB_IP:dc-lab-files/"
```

Repeat for each VM using its own management address. The examples will be under `~/dc-lab-files/configurations/` inside that VM. Copying these files does not apply them; the phase guides show how to inspect, back up, install, and validate each configuration.

If you prefer, create the destination files with `sudoedit` and paste the corresponding example. Do not copy a different VM's Netplan file onto your current VM.
