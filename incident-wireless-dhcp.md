# Restoring an access point's management path after a VLAN tagging mismatch

**Case study draft for review** | Duval Analytics

## Summary

A controller-managed access point became unreachable at its expected management address during a wireless outage. The investigation connected repeated DHCP activity, controller adoption failures, and the switch uplink's VLAN configuration to a management-path mismatch. The port classified untagged ingress into the management VLAN, while sending that VLAN back toward the AP with a tag.

The operator reported that changing the management VLAN to untagged on the AP uplink restored communication. A related SSID setting was then corrected, and the AP was assigned a static management address. Controller connectivity and one affected SSID were confirmed working; post-repair DHCP behavior and every other SSID were not verified in the available account.

This account is reconstructed from a retrospective report and verification notes drawn from the original diagnostic conversation. It distinguishes summarized records from operator-reported actions and diagnostic inference. Raw artifacts remain private. The focus is the diagnostic sequence and repair; no precise cause-to-symptom interval is asserted.

## Symptom and scope

The operator reported that all but one SSID stopped broadcasting. Clients could associate with the remaining SSID but had no usable connectivity. The controller showed failed adoption and later disconnection, displaying a fallback address outside the configured networks instead of the expected management address.

Other wired management-network clients remained operational. The DHCP service logged responses, and the controller was running and receiving discovery traffic. Those observations narrowed the problem toward the AP's path without proving every server, policy or shared component healthy.

The affected DHCP client in this investigation was the **AP's management interface**. Its management connectivity and the forwarding of wireless-client traffic were related symptoms but required separate checks. The exact mechanism behind the disappearance of SSIDs was not established.

## Diagnostic sequence

| Stage | Evidence or action | What it established |
|---|---|---|
| Establish actual addressing | Controller state and a discovery capture showed the fallback address; the lease table still listed the expected address. | The lease record did not establish what the AP was currently using. |
| Check the controller path | Listener output showed management listeners. Controller logs recorded discovery followed by adoption failures. | The controller was receiving discovery; the management exchange was not completing. |
| Inspect DHCP behavior | Server logs recorded repeated REQUEST/ACK activity, followed by DISCOVER/OFFER activity with no further REQUEST recorded in the reviewed window. | The server was processing AP requests, but successful client acceptance was not demonstrated. |
| Inspect forwarding configuration | Switch records showed a management-VLAN PVID and tagged management-VLAN egress on the AP uplink. | Ingress classification and return tagging needed comparison with AP expectations. |
| Restore the management path | The operator reported changing management-VLAN egress to untagged and recovering communication. | The intervention supported the uplink mismatch as the leading explanation. |
| Resolve remaining client traffic failure | After uplink correction, the management-network SSID still failed until its explicit VLAN tag was cleared. | AP management recovery alone was insufficient to establish wireless-client recovery. |

## Competing explanations

Controller uptime information did not indicate a reboot, but was not independent proof of uninterrupted AP operation. Controller event records contained no provisioning event explaining the failure; incomplete event coverage remained a limitation.

Hypervisor firewall configuration and a guest-chain check were reviewed. The absence of the checked guest chain did not eliminate every possible host, bridge or forwarding policy. Likewise, responses logged for the AP and other clients weakened a general DHCP-service-outage hypothesis without establishing delivery to this AP.

An alternate DHCP service was shown disabled with no selected interfaces. That addressed the particular service inspected, not the possibility of every other responder. Cable and power-source changes also occurred during troubleshooting; their timing was not sufficiently established to treat the incident as a controlled single-change experiment.

## Evidence for the directional diagnosis

Captures were taken on a hypervisor bridge, not at the AP-facing switch egress or on the AP. No SPAN or TAP was used. The bridge exposed relevant broadcasts and controller-guest traffic, but could not establish which replies reached or were accepted by the AP.

The strongest evidence was the combination of AP-originated traffic reaching services, failed completion of management exchanges, the recorded uplink tagging configuration, and reported recovery after the egress change. This supports a return-path tagging mismatch; it is not direct packet-level proof of the AP discarding tagged replies.

Some DHCP capture output had been filtered by client MAC text, losing packet context. It cannot establish that replies were absent. An OFFER without a subsequent REQUEST also does not alone prove nonreceipt. See the runbook's [capture placement](network-diagnostic-runbook.md#pt-c) and [directional testing](network-diagnostic-runbook.md#pt-h) guidance.

## Configuration and repair

The table uses role names instead of live identifiers. “Recorded” refers to the retrospective source's description of a record; the original configuration artifacts are not reproduced here.

| Setting | Before | Repair or resulting state | Evidence boundary |
|---|---|---|---|
| AP uplink PVID | Management VLAN | No PVID change reported as part of the repair | Before-state described in switch records |
| Management-VLAN egress on uplink | Tagged | Untagged | Before-state recorded; change and recovery operator-reported; no after-state switch image |
| AP management tagging expectation | Untagged was the working interpretation | No management-tagging change confirmed | Controller settings were ambiguous; exact effective mode remains unconfirmed |
| Management-network SSID | Explicit management-VLAN tag | Explicit tag cleared | Operator reported restored client connectivity |
| AP management addressing | DHCP | Static management address | Post-repair controller state showed the AP connected at that address |

PVID equality with a tagged VLAN is not inherently defective. The supported explanation is a mismatch between the port's egress treatment and the AP's expected management traffic. The SSID correction aligned that traffic with the repaired path; it does not establish a universal rule that a VLAN can never have mixed tagging behavior. See [VLAN verification](network-diagnostic-runbook.md#pt-d).

The reconstruction places management recovery after the uplink change, followed by the SSID correction. Static addressing was also applied, but its exact position relative to the first successful management exchange is not established. These actions should not be compressed into a claim that one isolated change proved the entire causal chain.

## Verification and limits

The controller showed the AP connected at a static management address. The operator reported that the affected management-network SSID worked after its tag was cleared. Other SSIDs were not individually retested in the available record, and no observation period was documented.

This establishes reported management recovery and a successful client check on one SSID. It does **not** establish successful post-repair DHCP allocation or renewal: the AP was using static addressing, and no post-repair DHCP capture was available. Static addressing also does not repair a VLAN forwarding mismatch by itself.

## Lessons incorporated into the runbook

1. Verify a device's observed address rather than treating a lease table as current client state.
2. Separate request arrival, response generation, return delivery and client acceptance.
3. Compare ingress policy, egress tagging and endpoint expectations independently.
4. Preserve capture context and document what each observation point cannot see.
5. Record changes in order and verify management and client-data paths separately.

The reusable [runbook](network-diagnostic-runbook.md) and [evidence/change-log template](evidence-change-log.md) turn those lessons into procedures. A fuller validation would document the effective AP management mode, switch after-state, per-SSID results and an observation window; a claim of repaired DHCP would additionally require a DHCP-based check. Such checks are follow-up work, not reconstructed incident actions.
