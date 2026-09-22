# Network Diagnostic Runbook

Symptom-based troubleshooting for Linux hosts, DHCP, DNS, VLANs and directional network failures. Maintained by **Duval Analytics**.

The emphasis is on choosing a useful observation point, preserving evidence, testing competing explanations, and verifying a repair. Ten symptom plans link to reusable pinpoint tests and a change log.

The runbook grew from a wireless outage involving an access point's management path and a switch-port PVID/tagging mismatch. The [reviewed incident retrospective](incident-wireless-dhcp.md) follows the diagnostic evidence, configuration repair and recovery checks. It distinguishes recorded observations, operator-reported actions and diagnostic inference, and states the limits of recovery verification.

## Read in this order

| Document | Purpose |
|---|---|
| [Runbook](network-diagnostic-runbook.md#test-plan-index) | Enter by symptom; follow tests and decision branches |
| [Incident retrospective](incident-wireless-dhcp.md) | Follow the diagnostic sequence, repair and verification limits |
| [Evidence/change-log template](evidence-change-log.md) | Record observations, hypotheses, changes and verification |
| [References](references.md) | Check protocol behavior and command semantics against primary sources |

## Audience and prerequisites

For infrastructure operators, homelab maintainers and technical reviewers familiar with IPv4 addressing, VLANs and Linux administration. Use only on systems you are authorized to inspect. You need the relevant topology/address plan, service ownership, appropriate privileges, and a private evidence location outside this repository. Configuration changes require recovery access and a rollback plan.

Examples target Bash/Linux with iproute2, iputils, tcpdump/libpcap, ethtool, BIND dig, an OpenBSD-compatible netcat, and systemd where noted. Firewall inspection depends on the installed backend. Install only tools needed for the selected test and consult the local version's manual. [Placeholder instructions](network-diagnostic-runbook.md#scope-and-command-conventions) explain substitution; examples have no default live targets.

## Tested versus untested

This revision received documentation review, local Markdown/link-target checks and offline Bash syntax/placeholder-guard checks. **No diagnostic command was run against a network, and no system configuration was changed.** These checks do not establish runtime compatibility or successful troubleshooting on particular devices.

The scope is IPv4-first Ethernet/Wi-Fi troubleshooting with Linux command examples. IPv6 internals, RF surveys, vendor-specific recovery, and a device/OS compatibility matrix are outside the validated scope. The original incident does not imply every procedure was field-tested.

Review on 2026-09-21 covered all 11 Bash blocks, local Markdown links and anchors, placeholder consistency, sanitization, and the 22 external URLs in the [primary references](references.md). External-link availability was checked at that time; command syntax and placeholder guards were checked offline. The case study identifies the remaining historical evidence and recovery-verification limits.

Public content contains sanitized prose and templates only. Resumes, raw logs, packet captures, live configuration and identifying infrastructure details are excluded.

## License

Copyright © 2026 Duval Analytics. **All rights reserved.** This is a proprietary portfolio repository, not an open-source project. See [LICENSE](LICENSE) for terms; reuse beyond applicable legal or platform permissions requires written permission.
