# INC-XXX — Exercise title

- Type: controlled lab simulation
- Date/time and timezone:
- Affected VM/service:
- Client VM/address:
- Starting baseline:
- Recovery access/method:

## Symptoms and scope

Record the actual error and which clients/services are affected. Note any checks that still work.

## Change used to simulate the fault

Record the exact command, target interface/service, and time. Keep this separate from the diagnosis so you can repeat the exercise later.

## Diagnostics

| Time | Command/check | Actual result | Interpretation |
|---|---|---|---|
| | | | |

## Root cause

Explain which evidence supports the cause. Distinguish a service failure, route problem, firewall drop, resolver failure, or performance constraint.

## Resolution

Record exactly what restored the intended state.

## Verification

- Original failing test repeated:
- Relevant local service/interface check:
- Client-to-service check:
- Temporary rules/listeners/traffic shaping removed:
- Monitoring recovered, if applicable:

## Escalation considerations

Record what you would check next if recovery failed: hypervisor network assignment, guest logs, service configuration, route/return path, host load, or another dependency.

## Lessons and supporting files

Link useful output/captures and note one improvement to the troubleshooting process. Fill this record with observed results; the procedure's expected result is a comparison, not an observation.
