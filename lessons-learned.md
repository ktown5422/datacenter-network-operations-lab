# Lessons learned

## Operational principles

- A VM must be validated before being treated as a reliable network node.
- Local connectivity, gateway reachability, external IP reachability, and DNS resolution should all be confirmed separately.
- Backups are essential before Linux storage changes, especially when using LVM.
- Route and neighbor information should be reviewed before making any changes to networking.
- Save useful command output and explain what it means so you can retrace your troubleshooting steps.

## What to take away from the exercises

Use the [lab checklist](documentation/completion-review.md) to record your own results and add the observations that helped you understand each exercise.

- A working baseline makes later fault comparisons meaningful.
- LVM free space and filesystem capacity are different from the virtual disk's provisioned size.
- A local SSH listener, remote TCP access, and successful authentication are separate checks.
- Static routes need a valid return path; use private addresses to prove the routed path instead of bypassing it over management.
- Firewall input rules and forwarding rules apply to different traffic paths.
- An exporter scrape failure does not prove a VM or its application is down; compare metrics, probes, and service checks.
- Packet captures and directional throughput tests answer different questions about network behavior.

## Questions for your own notes

- Which diagnostic check narrowed the scope fastest in each incident?
- Which results differed from the examples, and why?
- Did your recovery restore the persistent configuration as well as the current running state?
- Which temporary rules, routes, queues, and listeners needed cleanup?
- What would you change to make the next run easier to reproduce?
