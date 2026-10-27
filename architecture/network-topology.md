# Network topology

## Overview

This walkthrough uses three Ubuntu ARM64 VMs in UTM's QEMU backend. Build the first VM in Phases 1–6, then follow [Phase 7](../documentation/phase-07-multi-server-networking.md) to add the private routed networks and two additional VMs.

## Starting environment

- Host: Apple Silicon Mac
- Hypervisor: UTM
- VM: `dc-network-01`
- OS: Ubuntu Server 26.04.1 LTS (ARM64)
- Network: Shared Network
- Interface: `enp0s1`
- Gateway: `192.168.64.1`
- Subnet: `192.168.64.0/24`

## Full walkthrough topology

```text
Mac -- UTM Shared Network / management DHCP -- all three VMs

dc-server-01           dc-network-01                 dc-monitor-01
10.10.20.10/24 -- lab-app -- 10.10.20.1/24
                          10.10.30.1/24 -- lab-monitor -- 10.10.30.10/24
NGINX / exporter       IPv4 router / exporter        Prometheus / Grafana
```

## Network assignments

| UTM network | Members | Purpose |
|---|---|---|
| Shared Network | Management NIC on each VM; Mac host | DHCP, DNS, package downloads, management SSH |
| Host Network `lab-app` | Router application NIC + server private NIC | `10.10.20.0/24` application segment |
| Host Network `lab-monitor` | Router monitoring NIC + monitor private NIC | `10.10.30.0/24` monitoring segment |

The two private segments must use different Host Network UUIDs. The two NICs on a given segment must select the same UUID. These named Host Networks have no DHCP or direct Mac connection; administration stays on Shared Network. See [UTM's Host Networks reference](https://docs.getutm.app/preferences/macos/#host-networks).

## Traffic paths

- Mac → management DHCP address: SSH for administration or UI tunneling.
- Monitor `10.10.30.10` → router → server `10.10.20.10`: HTTP, exporter scrape, and temporary throughput tests.
- Server → router → monitor: return traffic and reverse-direction routing tests.
- Each VM → its management default gateway: internet access and upstream DNS.
- Grafana → local Prometheus; Prometheus → local Blackbox Exporter: loopback connections on the monitor.

## Notes

- Separate management paths keep SSH available during private-network faults.
- Only the router enables IPv4 forwarding. Specific private routes preserve management's default gateway, and no NAT is needed between the lab subnets.
- UFW limits private service access to the monitor. Grafana, Prometheus, and the probe API bind to loopback and are accessed through SSH.
- All guest workloads share the Mac's resources; virtual throughput is not a physical-link benchmark.
