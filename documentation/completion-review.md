# Lab checklist

Use this checklist alongside the README. Work through the phases in order, verify each result, and keep notes on what you learned. Screenshots are optional; concise command output and written observations are useful for troubleshooting.

## Guide coverage

The README covers Phases 1–4B directly and links detailed step-by-step guides for Phases 4C–12. Those guides include prerequisites, command locations, expected results, verification, and recovery. Configuration examples support the routed networks and monitoring stack.

Use this table to navigate the walkthrough and check your own run. Keep your actual addresses, OS version, and observations in your notes; example results are comparisons, not newly collected VM output.

## Exercise checkpoints

For your own run, mark each exercise complete when its checkpoint passes. Record your actual results, including failures and recovery steps.

| Phase | Walkthrough | What to check and record |
|---|---|---|
| 1 — VM creation | [README](../README.md#phase-1--create-the-vm) | QEMU backend, ARM64, CPU/RAM/disk, Shared Network, and installer boot. |
| 2 — OS installation | [README](../README.md#phase-2--install-ubuntu) | Hostname, user, OpenSSH, and booting the installed disk after detaching the ISO. |
| 3 — Server inspection | [README](../README.md#phase-3--inspect-and-prepare-the-server) | OS, CPU, available memory, filesystems, interfaces, default route, and package access. |
| 3B — LVM | [README](../README.md#phase-3b--expand-lvm-storage) | Completed backup, correct root LV, free extents, and before/after filesystem/LVM sizes; skip resizing if already expanded. |
| 4A — Connectivity | [README](../README.md#phase-4a--establish-a-network-baseline) | Gateway/external IP, DNS, and local SSH listener checked separately. |
| 4B — Routing/ARP | [README](../README.md#phase-4b--ipv4-subnetting-routes-and-arp) | Local/external route selection, gateway neighbor entry, and subnetting answers. |
| 4C — DNS/DHCP | [Detailed guide](phase-04c-dns-dhcp.md) | Lease/resolver inspection, DNS failure and recovery, DHCP renewal, and working baseline. |
| 5 — Toolkit | [Detailed guide](phase-05-networking-toolkit.md) | Interpreted traceroute, curl, ss/htop, ICMP capture, and loopback iperf3 test. |
| 6 — SSH | [Detailed guide](phase-06-remote-ssh.md) | Verified host key, Mac login, public-key-only authentication, logs, and console access. |
| 7 — Multi-server | [Detailed guide](phase-07-multi-server-networking.md) | Three VMs, two separate Host Networks, static addresses/return routes, persistent forwarding, bidirectional private connectivity, and NGINX. |
| 8 — Firewall | [Detailed guide](phase-08-firewall-controls.md) | UFW active, management SSH permitted, routed traffic permitted, HTTP deny/recovery drill cleaned up. |
| 9 — Packets/performance | [Detailed guide](phase-09-packet-performance.md) | Four interpreted captures, forward/reverse VM throughput results, temporary listener/rule removed. |
| 10 — Monitoring | [Detailed guide](phase-10-monitoring.md) | Three healthy node targets, HTTP probe, six dashboard panels, loopback UIs, and alert firing/resolution. |
| 11 — Incidents | [Detailed guide](phase-11-incident-response.md) | Five fault/diagnosis/recovery records using the template; no temporary faults remain. |
| 12 — Final checks | [Detailed guide](phase-12-final-verification.md) | Baseline/service checks repeated, settings verified after one-at-a-time reboots, notes saved, clean shutdown if finished. |

## Exercise notes

For every exercise, record:

- Observation date and time, including timezone.
- VM hostname, current address, and relevant software version.
- Command or action and its actual output, or a screenshot that shows it.
- What the result means, any limitations, and a link to related troubleshooting notes.
- For changes: initial condition, recovery method, change made, and post-change verification.

Use your actual observations. If a result differs from the example, investigate the difference and note what you found. Leave an exercise open when its checkpoint has not passed.

## Next steps

1. Start with the VM setup and record your installed OS version and network settings.
2. Work through the exercises in phase order, using the checkpoints above.
3. Mark each exercise complete when its checks pass, and save useful troubleshooting notes.
4. Update the README, architecture notes, and incident records to match the setup you actually used.

You can pause after the single-VM baseline and return to the later exercises when ready. Keep unfinished phases open in your notes so you know where to resume.
