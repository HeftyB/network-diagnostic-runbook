# Evidence and change log

Copy this template to a **private location outside the repository** for operational use. Keep source artifacts private; publish a separately reviewed, sanitized summary. Use stable role labels and the runbook's [placeholder convention](network-diagnostic-runbook.md#scope-and-command-conventions). Blank fields mean unknown until filled; they are not negative findings.

## Intake and time basis

- Incident ID / owner / authorized scope:
- Expected operation / observed symptom:
- Affected clients, services, address families / unaffected controls:
- First observed / last known good (original time and UTC offset):
- UTC correlation basis / clock offsets and uncertainty:
- Relative timeline origin / source supporting it:
- Topology roles and observation points:
- Relevant timers and measured values / search window and rationale:
- Probe/capture scope, start, stop and privacy constraints:

## Evidence register

Separate what a source shows from its interpretation. Give each item an ID used by hypotheses and changes.

| Evidence ID | Original time/offset and UTC | Source/placement/namespace | Exact observation or private artifact reference | Filter/window/drop statistics | Coverage limits/confidence |
|---|---|---|---|---|---|
| | | | | | |

Record artifact integrity hashes privately where useful. Do not embed credentials, raw traffic or configuration here for publication. Missing logs require a logging/retention/permission check; missing packets require a capture-coverage check.

## Hypothesis register

| ID | Hypothesis and predicted observation | Test/control and action class | Evidence IDs and result | Status: open / weakened / supported / rejected within scope | Remaining alternatives |
|---|---|---|---|---|---|
| | | | | | |

## Direction and control

| Flow/transaction ID | Sender observation | Receiver observation | Reply observation | Delivery/acceptance observation | NAT/relay/tagging and visibility limits |
|---|---|---|---|---|---|
| | | | | | |

- Matched control: same destination, protocol, operation and time window:
- Differences in VLAN, source, identity, pool, policy, family or route:
- Supported interval of failure / unresolved ambiguity:

## Port and endpoint matrix

| Port/link role | Endpoint sends/expects | Admission policy | Untagged ingress PVID | VLAN membership | Egress tag action | Source and confidence |
|---|---|---|---|---|---|---|
| | | | | | | |

## Change register

Include pre-incident changes and diagnostic changes. Mark retroactively reconstructed entries and uncertain times.

| Change ID | Time/offset and UTC | Owner/authorization | Exact before/after values or private reference | Hypothesis/evidence IDs | Affected dependencies and expected impact | Result/retained/reverted |
|---|---|---|---|---|---|---|
| | | | | | | |

For **each planned change**, complete before execution:

- Change ID, prediction and success criteria:
- Prerequisites, saved state/configuration, lease data if relevant:
- Coupled settings / why they must change together:
- Maintenance window / affected users / communications owner:
- Recovery access and verified recovery procedure:
- Exact rollback sequence, trigger, decision deadline and responsible person:
- Irrecoverable effects (lost volatile evidence, interrupted sessions):
- Actual execution/result/rollback evidence IDs:
- Temporary capture/logging settings and removal confirmation:

## Closure and publication summary

- Supported conclusion and evidence IDs:
- Actual repair (distinguish from proposed changes):
- Original user operation retested / client and family / result:
- DHCP allocation/renewal and options checked, if relevant:
- Adjacent services and expected denials checked / results:
- Observation duration versus relevant timer / recurrence:
- Unresolved facts, residual risk, follow-up owner and deadline:
- Monitoring improvement and what it actually detects:
- Publication review: replace identities/addresses/IDs, omit private artifacts and secrets, retain uncertainty, verify every causal claim:
