# INC-001 — Connectivity baseline

- Type: preventive baseline check; no outage or injected fault.
- Use: reference for your own baseline notes in Phase 4A.

## Summary

Check gateway reachability, external IP access, DNS resolution, and the local SSH listener on `dc-network-01`. Record the date, current address, actual output, and interpretation in your own notes. Remote SSH authentication is tested separately in Phase 6.

## Commands

```bash
ip -4 addr show enp0s1
ip route
ping -c 4 192.168.64.1
ping -c 4 1.1.1.1
getent ahostsv4 google.com
sudo systemctl status ssh
sudo ss -tlnp | grep ':22'
```

## Expected baseline results

- Gateway responded successfully.
- External IPv4 address responded successfully.
- DNS resolved hostnames successfully.
- SSH service was active and listening on port 22.

## Interpretation

These checks establish gateway and external ICMP reachability, IPv4 name resolution, and a local TCP listener when their results match the expected baseline. They do not establish remote SSH login or firewall behavior.

## Verification

Repeat the checks after a relevant change and compare results with your baseline. Use [Phase 6](../documentation/phase-06-remote-ssh.md) for remote login and [Phase 11](../documentation/phase-11-incident-response.md) for controlled fault/recovery drills.
