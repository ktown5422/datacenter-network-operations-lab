# INC-002 — Routing baseline

- Type: preventive routing check; no outage or injected fault.
- Use: reference for your own route/neighbor notes in Phase 4B.

## Summary

Compare local and external route selection and inspect the gateway's ARP neighbor entry on `dc-network-01`. Record your actual addresses, output, and observation time.

## Commands

```bash
ip route
ip route show default
ip route get 192.168.64.10
ip route get 8.8.8.8
ping -c 4 192.168.64.1
ip neigh show
ip -4 addr show enp0s1
```

## Expected findings for the example subnet

- The local `/24` destination used the directly connected network path.
- The external destination used the default gateway route.
- The gateway responded successfully to ICMP requests.
- ARP entries resolved the gateway's link-layer address on the local segment.

## Interpretation

This behavior is consistent with a connected `/24` route and a default gateway. A route lookup shows route selection, not destination reachability. [Phase 7](../documentation/phase-07-multi-server-networking.md) demonstrates routing between the private lab subnets.

## Verification

Compare both route lookups with actual connectivity tests and explain the next-hop MAC choice. Repeat the checks after network changes and use [Phase 11](../documentation/phase-11-incident-response.md) for controlled faults.
