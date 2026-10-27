# IPv4 and subnetting

## Core concepts

- IPv4 uses dotted-decimal notation.
- CIDR notation identifies the prefix length, such as `/24`.
- A `/24` network contains 256 addresses, but only 254 are typically usable for hosts.
- A gateway can use any suitable host address on the subnet; `.1` is a convention, not a requirement.

## Example address plan

- Network: `192.168.64.0/24`
- Usable host range: `192.168.64.1 - 192.168.64.254`
- Gateway example: `192.168.64.1`
- Host example: `192.168.64.3`

## Useful commands

```bash
ip -4 addr show
ip route
ping -c 4 192.168.64.1
ip route get 192.168.64.10
ip route get 8.8.8.8
ip neigh show
```

## Outcomes

For this example, expect the connected route to select the local subnet and the default route to select the gateway for external traffic. Record your actual route/neighbor results using [INC-002](../incidents/INC-002-routing-baseline.md) as a reference. [Phase 7](phase-07-multi-server-networking.md) extends the exercise to two routed private subnets.

## Exercise answers

1. An IP address identifies an interface at the network layer and supports routing. A MAC address identifies a link-layer interface on the local segment.
2. In the reported route table, `192.168.64.10` matches the connected `/24` route; `8.8.8.8` uses the default route through `192.168.64.1` when no more specific route matches.
3. `/24` means 24 network-prefix bits and 8 remaining host bits: 256 total addresses and conventionally 254 usable hosts on this subnet.
4. `ip route get` shows the kernel's selected route, outgoing interface, and source address. It does not test whether the destination exists, responds, or accepts a service connection.
5. For IPv4 over Ethernet, ARP maps an on-link next-hop IPv4 address to a MAC address. For an off-subnet destination, the VM resolves the gateway's MAC, not the remote destination's MAC.

## Notes

- Subnetting is a way to divide one network into smaller networks.
- Routing prefers the longest matching destination prefix; the default route is used when no more specific route matches.
- ARP resolves on-link IPv4 next hops to MAC addresses on the same Layer 2 domain.
