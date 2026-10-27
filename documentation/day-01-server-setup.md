# Day 01 — Server setup

## Objective

Provision and validate the Ubuntu server VM for networking and troubleshooting exercises.

## Walkthrough order

Use [Phases 1–4B in the README](../README.md#phase-1--create-the-vm) for the full setup instructions. This page is a quick reference for your first session.

1. Create an ARM64 Ubuntu VM in UTM's QEMU backend with Shared Network.
2. Install the OS, create a user, enable OpenSSH, detach the ISO, and boot the installed system.
3. Inspect CPU, memory, storage, hostname, addresses, and routes.
4. Back up the VM, inspect LVM, and expand the root LV only if free extents remain and the exercise applies.
5. Verify gateway/external IP connectivity, DNS, the SSH listener, route selection, and ARP.

## Original example values

- Hostname: `dc-network-01`
- Interface: `enp0s1`
- IPv4: `192.168.64.3/24`
- Gateway: `192.168.64.1`
- Root filesystem: approximately 17 GiB after the example expansion

## Commands used

```bash
cat /etc/os-release
hostnamectl
free -h
df -h
ip -br addr
ip route
```

## Outcome

Record your actual results rather than assuming the example values apply. The [lab checklist](completion-review.md) lists the checkpoints. Continue with [Phase 4C](phase-04c-dns-dhcp.md), then test remote SSH in Phase 6.

## Lessons learned

- Always verify the correct volume group before resizing storage.
- Back up before any LVM change.
- Capture outputs and interpret them before moving on.
