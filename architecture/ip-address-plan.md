# IP address plan

## Management network

| Host | Role | Example interface | Address | Prefix | Gateway |
| --- | --- | --- | --- | --- | --- |
| dc-network-01 | Linux networking lab | `enp0s1` | `192.168.64.3` | `/24` | `192.168.64.1` |
| dc-server-01 | NGINX and test server | `enp0s1` | DHCP; record actual value | `/24` in this example | DHCP |
| dc-monitor-01 | Monitoring | `enp0s1` | DHCP; record actual value | `/24` in this example | DHCP |

Management addresses are assigned by UTM's Shared Network and can change. `192.168.64.3` and gateway `192.168.64.1` are original lab examples, not reservations. Record the current subnet, addresses, and gateway before using SSH or management firewall rules.

## Management subnet example

- Network: `192.168.64.0/24`
- Broadcast: `192.168.64.255`
- Usable host range: `192.168.64.1 - 192.168.64.254`
- Gateway: `192.168.64.1`
- Checks: gateway/external IP reachability, DNS resolution, and Mac-to-VM SSH in Phases 4A and 6.

## Private addresses configured in Phase 7

| VM | UTM Host Network | Example interface | Static address | Route to opposite subnet |
| --- | --- | --- | --- | --- |
| dc-network-01 | `lab-app` | `enp0s2` | `10.10.20.1/24` | Directly connected |
| dc-network-01 | `lab-monitor` | `enp0s3` | `10.10.30.1/24` | Directly connected |
| dc-server-01 | `lab-app` | `enp0s2` | `10.10.20.10/24` | `10.10.30.0/24 via 10.10.20.1` |
| dc-monitor-01 | `lab-monitor` | `enp0s2` | `10.10.30.10/24` | `10.10.20.0/24 via 10.10.30.1` |

For each private `/24`, `.0` is the network address and `.255` is the broadcast address. The router uses `.1` by design; this is a convention, not a requirement. Private interfaces have no default route or DNS server.

## Service ports

| Service | Destination | Allowed source/path |
|---|---|---|
| SSH, TCP 22 | Each management address | Management subnet |
| HTTP, TCP 80 | `10.10.20.10` | Monitor `10.10.30.10`, routed |
| Node Exporter, TCP 9100 | Router `10.10.30.1` and server `10.10.20.10` | Monitor `10.10.30.10` |
| Monitor's Node Exporter, TCP 9100 | `127.0.0.1` | Local Prometheus; no inbound UFW allow added |
| iperf3, TCP 5201 | `10.10.20.10` | Monitor, temporarily during Phases 9/11 |
| Prometheus, TCP 9090 | Monitor `127.0.0.1` | Local access / Mac SSH tunnel |
| Grafana, TCP 3000 | Monitor `127.0.0.1` | Mac SSH tunnel |
| Blackbox Exporter, TCP 9115 | Monitor `127.0.0.1` | Local Prometheus |

## Notes

- Match private NICs by MAC address rather than assuming their Linux names.
- Avoid overlap with host/VPN networks; if changing the example ranges, update all routes, UFW rules, and monitoring targets together.
- See [Phase 7](../documentation/phase-07-multi-server-networking.md) and the [Netplan examples](../configurations/README.md) for installation and recovery steps.
