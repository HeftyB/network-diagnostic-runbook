# Network Diagnostic Runbook

**Symptom-based test plans for infrastructure fault isolation**

Revision 1.1 | Maintained by Duval Analytics

Start with [intake](#intake), select a [test plan](#test-plan-index), and use the linked pinpoint tests. Record findings in the [evidence/change-log template](evidence-change-log.md). The [incident retrospective](incident-wireless-dhcp.md) explains the origin; it is not a test transcript for these procedures.

## Scope and command conventions

This is an IPv4-first Ethernet/Wi-Fi and Linux troubleshooting guide. Commands assume Bash, iproute2, iputils, systemd where indicated, and separately installed tools. DNS AAAA checks help detect address-family differences; IPv6 RA/DHCPv6/NDP, routing-protocol debugging, RF surveys, and vendor recovery need separate procedures. Do not apply ARP or DHCPv4 conclusions to IPv6.

Examples were reviewed against documentation, not run on a production or lab network for this revision. See [tested versus untested scope](README.md#tested-versus-untested).

All operator-supplied values use single-quoted `REPLACE_UPPER_SNAKE_CASE` tokens. Replace each entire token with an authorized value, not shell syntax or untrusted text. Each parameterized Bash block runs in a subshell and stops while a token remains, preventing reuse of ambient variables. Read and select commands before execution. Guards are editing aids, not input validators.

| Placeholder | Substitute |
|---|---|
| `REPLACE_IFACE` | Exact local interface in the correct namespace |
| `REPLACE_TARGET_IPV4` | Approved destination's literal IPv4 address |
| `REPLACE_NEXT_HOP_IPV4` | On-link next hop selected by the route lookup |
| `REPLACE_PEER_IPV4` | Other endpoint's literal IPv4 address as visible at this capture point |
| `REPLACE_RESOLVER_IPV4` | Approved resolver's literal IPv4 address |
| `REPLACE_FQDN` | Approved fully qualified DNS name |
| `REPLACE_TCP_PORT` | One authorized TCP port, numeric 1–65535 |
| `REPLACE_SERVICE_UNIT` | Exact local systemd service unit |
| `REPLACE_START_UTC`, `REPLACE_END_UTC` | UTC strings in `YYYY-MM-DD HH:MM:SS UTC` form |
| `REPLACE_GUEST_ID` | Guest identifier in private notes; verify actual interface mappings |

`0.0.0.0`, `127.0.0.1`, and `::1` describe protocol/binding semantics, not example targets. No default live target or public resolver is supplied.

**Action labels:** **READ** inspects local state; **PROBE** sends traffic and can populate caches, create connections, or trigger logs; **CAPTURE** collects potentially sensitive traffic and may enable promiscuous mode; **CHANGE** modifies configuration or disrupts state. Every CHANGE uses the [change procedure](#change-procedure). Keep original evidence privately outside this repository.

## Rules of engagement

1. **Corroborate evidence.** Captures show what the collection point exposes; logs show what a service emits; lease tables show server state. None alone proves end-to-end delivery. Record placement, filters, clock offset and blind spots.
2. **Test both directions.** Separate request delivery, response generation, response delivery and application acceptance. [PT-H](#pt-h) narrows an interval without prematurely naming a culprit.
3. **Use a control.** A working peer narrows scope but does not clear a server or shared path: client policies, pools, load balancing, VLANs and timing can differ. See [PT-J](#pt-j).
4. **Search before symptom onset.** Cached state and timers can conceal an earlier change. Use measured settings and [timer guidance](#timers), then widen the search. A timer match is a hypothesis.
5. **Preserve state before changing it.** Log the change and prediction first. Treat coupled settings as one documented change set and acknowledge reduced attribution.
6. **Qualify silence.** Missing logs can reflect logging level, retention, permissions, rate limits, wrong instance, clock error or a failed logger. Missing packets can reflect placement, filters or drops. Absence constrains a hypothesis only when coverage is established.
7. **Normalize time.** Retain original timestamps/offsets, correlate in UTC, and record clock uncertainty. Do not manufacture precision from memory.
8. **Read the underlying error.** Preserve exact messages and context; UI labels may summarize stale or incomplete state. If logs are unavailable, record that limitation and seek another observation point.
9. **Separate observation, hypothesis and conclusion.** “OFFER seen at server” is an observation; “return-path loss” is a hypothesis. Conclusions need correlated evidence and verification. Retire hypotheses only as far as evidence permits.

## Intake

1. Record expected operation, symptom, first observed time, last known good time, affected clients/services, and what still works. Reproduce within the approved scope. Intermittent or no longer reproducible → [TP-08](#tp-08).
2. Sketch the relevant client/AP/switch/VLAN/relay/router/server path and capture points. Record IPv4 and IPv6 behavior separately. Failed ping is not proof a host is down.
3. Preserve private logs and configuration before rebooting, flushing caches, releasing leases or resetting devices. Start the [template](evidence-change-log.md).
4. Review manual and automated changes across switching, firewall, hypervisor, DNS, DHCP, certificates, software, cabling and power. Start with an available recent window (for example 72 hours), then extend for relevant timers and last known good evidence.
5. Set probe/capture scope and a stop time. Reconstruct any fixes already attempted; mark uncertain times.

## Test plan index

| Symptom | Entry |
|---|---|
| No connectivity | [TP-01](#tp-01) |
| No expected IPv4 address or DHCP failure | [TP-02](#tp-02) |
| IP works but name-based access fails | [TP-03](#tp-03) |
| Partial or directional failure | [TP-04](#tp-04) |
| Service works locally, fails remotely | [TP-05](#tp-05) |
| Managed device offline or provisioning fails | [TP-06](#tp-06) |
| Wireless association but no usable traffic | [TP-07](#tp-07) |
| Intermittent or degraded service | [TP-08](#tp-08) |
| VLAN isolation or routing differs from policy | [TP-09](#tp-09) |
| Failure following a change | [TP-10](#tp-10) |

Return to the calling plan after each pinpoint test. If evidence is insufficient, improve observation coverage or escalate with the evidence log; do not force a diagnosis. Every repair ends with [verification](#verification).

<a id="tp-01"></a>
## TP-01: No connectivity

1. Run [PT-A](#pt-a). No carrier/unexpected speed → examine physical, power, negotiation and administrative state at both ends. Carrier present does not prove a healthy path.
2. Inspect addressing/routes with [PT-F](#pt-f). Missing/unexpected address with DHCP intended → [TP-02](#tp-02). Static addressing intended → compare with the approved plan. Missing default route matters for destinations without a more specific route, not all local communication.
3. Test the selected next hop with [PT-B](#pt-b). Failed resolution → check addressing, link, endpoint availability and [PT-D](#pt-d). Resolved → [PT-H](#pt-h).
4. Request seen only near sender → inspect the forward interval. Reply seen only near responder → [TP-04](#tp-04). No request observed → verify application action, capture, namespace, route and [PT-E](#pt-e). Bidirectional exchange but failed operation → [TP-03](#tp-03) or [TP-05](#tp-05).

<a id="tp-02"></a>
## TP-02: DHCPv4 failure

Confirm DHCP is intended. Use [PT-C](#pt-c) at suitable client/server/relay points before inducing a transaction. Initial allocation commonly uses DISCOVER → OFFER → REQUEST → ACK; renewal or reuse can start with REQUEST. Do not require DISCOVER in every window. See [RFC 2131](https://www.rfc-editor.org/rfc/rfc2131.html).

| Correlated observation | Next test and conclusion limit |
|---|---|
| No client message here | Validate coverage and client state; inspect logs/interface, then [PT-A](#pt-a)/[PT-D](#pt-d). Client may be bound, waiting or using another interface. |
| DISCOVER at client, absent at server | Inspect VLAN, relay, DHCP snooping/security policy and forwarding interval with [PT-H](#pt-h). |
| DISCOVER at server, no OFFER observed | Inspect service/interface, pool selection/capacity, relay information, client classification and authorization; verify server egress coverage. |
| OFFER observed, no REQUEST | Compare client capture/logs. Consider delivery loss, rejected parameters, another selected offer, client state or incomplete capture. OFFER alone proves neither receipt nor nonreceipt. |
| REQUEST observed, no ACK/NAK | Compare server reception, response policy and return path; check client identity/requested address against server state. |
| NAK observed | Inspect responding server, requested network/address and client state. This alone diagnoses neither duplicate servers nor address conflict. |
| ACK observed, no usable configuration | Verify client/transaction match and delivery; inspect installation errors, duplicate-address detection/DECLINE, prefix, router, DNS and routing. Then [TP-01](#tp-01). |

**READ**, on a systemd-managed server: inspect the actual DHCP unit and relevant time window. Appliances/non-systemd services need the documented equivalent.

```bash
(
  SERVICE_UNIT='REPLACE_SERVICE_UNIT'
  START_UTC='REPLACE_START_UTC'
  END_UTC='REPLACE_END_UTC'
  case "$SERVICE_UNIT $START_UTC $END_UTC" in *REPLACE_*) echo 'Substitute all placeholders first.' >&2; exit 2;; esac
  systemctl --no-pager status "$SERVICE_UNIT"
  journalctl --utc --no-pager -u "$SERVICE_UNIT" --since "$START_UTC" --until "$END_UTC"
)
```

Several responding servers → compare server identifiers, scopes, options, relay destinations and the approved redundancy design with the owner. Multiple authorized servers can be intentional. Investigate conflicting configuration; do not disable a server based on count alone. Active DHCP discovery scans are not required by default: synthetic clients can affect server state and receive different policy.

Use [PT-J](#pt-j) to compare an actual peer transaction. Supported mismatch → scoped [CHANGE](#change-procedure), then verify client configuration and the original operation.

<a id="tp-03"></a>
## TP-03: Name resolution or name-based access failure

**READ:** inspect resolver/search/routing domains with `resolvectl status` on systemd-resolved hosts, or read `/etc/resolv.conf` otherwise. A stub address can represent a local resolver; VPNs, browsers and applications may use other paths.

**PROBE:** query an approved name/resolver. Requests can populate caches and disclose names to that resolver.

```bash
(
  NAME='REPLACE_FQDN'
  RESOLVER='REPLACE_RESOLVER_IPV4'
  case "$NAME $RESOLVER" in *REPLACE_*) echo 'Substitute all placeholders first.' >&2; exit 2;; esac
  dig @"$RESOLVER" "$NAME" A +time=2 +tries=1
  dig @"$RESOLVER" "$NAME" AAAA +time=2 +tries=1
  dig @"$RESOLVER" "$NAME" A +tcp +time=2 +tries=1
  getent ahosts "$NAME"
)
```

- Timeout → check resolver reachability, UDP/TCP policy and health via [PT-H](#pt-h)/[PT-E](#pt-e). It is not an authoritative “no record” response.
- NXDOMAIN, NOERROR with no requested data, SERVFAIL or REFUSED → preserve status, flags, answer and authority sections; inspect resolver/zone/view, DNSSEC and policy. Compare an approved authority/resolver for the same DNS view.
- Different answers → check split DNS, forwarding, TTL and cache state before calling either wrong. Two failures can share a local path fault.
- Direct query works, application fails → compare application resolver/search suffixes, NSS/hosts, cache, A/AAAA connectivity and TLS/SNI behavior. `getent` uses system NSS, not necessarily the application's resolver.

`dig @resolver` does **not** bypass that resolver's cache. `+trace` makes iterative queries and does not reproduce internal recursive/split-DNS behavior; reserve it for approved public names. Avoid `+short` during diagnosis because it hides status. See [BIND dig documentation](https://bind9.readthedocs.io/en/latest/manpages.html) and [timers](#timers).

<a id="tp-04"></a>
## TP-04: Partial or directional failure

1. [PT-H](#pt-h): locate the first observation boundary where the same exchange disappears. Broadcast success does not prove unicast forwarding; inspect ARP, forwarding/security policy and [PT-D](#pt-d).
2. Same subnet works, another fails → [PT-F](#pt-f)/[PT-E](#pt-e), including return route. Only some ports fail → [PT-G](#pt-g) and policy. These differences narrow scope, not uniquely identify a layer.
3. Small IPv4 packets work, larger ones fail → test a path-MTU hypothesis against an approved ICMP target. **PROBE:**

```bash
(
  TARGET_IP='REPLACE_TARGET_IPV4'
  case "$TARGET_IP" in *REPLACE_*) echo 'Substitute all placeholders first.' >&2; exit 2;; esac
  ping -4 -n -c 3 -W 2 -M do -s 1472 "$TARGET_IP"
  ping -4 -n -c 3 -W 2 -M do -s 1372 "$TARGET_IP"
)
```

These yield 1500/1400-byte IPv4 packets with ordinary 20-byte IP and 8-byte ICMP headers. Tunnels reduce usable MTU; loss, filtering or local MTU errors can also explain failure. Capture ICMP errors and test the affected application before concluding PMTU failure. See [iputils ping](https://raw.githubusercontent.com/iputils/iputils/master/doc/ping.xml).

<a id="tp-05"></a>
## TP-05: Service works locally, fails remotely

1. [PT-G](#pt-g): inspect process, port, family, namespace, bind address and proxy/published-port mappings. Loopback-only listeners cannot directly accept remote connections. If remote access is intended, choose the least-exposed intended bind address through a [CHANGE](#change-procedure), not an automatic wildcard bind.
2. Test the **same service destination** from its host, appropriate parent host, same-subnet client and routed client. Parent-to-guest traffic can traverse different hooks. TCP connection success does not prove application health.
3. First failing vantage point → compare route, NAT, firewall, exposure and captures via [PT-E](#pt-e)/[PT-F](#pt-f)/[PT-H](#pt-h). No listener → service owner. Transport works, operation fails → application/TLS/authentication evidence. Local success alone does not identify a host firewall fault.

<a id="tp-06"></a>
## TP-06: Managed device offline or provisioning fails

1. Preserve controller/device errors, timestamps and versions. Verify the current device address via local state, correlated capture and approved inventory. ARP/lease/UI records can be stale.
2. Wrong/missing address → [TP-01](#tp-01)/[TP-02](#tp-02). Correct → identify documented protocol, initiator, destination, transport and ports. Some systems use one device-initiated session; do not assume a separate inbound provisioning port.
3. Test the actual flow with [PT-H](#pt-h)/[PT-G](#pt-g)/[PT-E](#pt-e). Discovery success proves neither provisioning reachability nor a reverse-path fault.
4. Transport works → check trust, clock, certificates, credentials, compatibility and application logs. UDP `nc` success/silence is not a health check; use a protocol-valid request and correlated response/error.
5. Unresolved → escalate with evidence and recovery options. Reset only with a restore/re-adoption procedure and [CHANGE](#change-procedure) authorization; successful reachability does not justify erasing state.

<a id="tp-07"></a>
## TP-07: Wireless association but no usable traffic

1. Verify association **and authorization**, including 802.1X/NAC, assigned role and captive-portal state where applicable. “Connected” does not establish forwarding. Authorization fails → wireless/authentication logs; succeeds → addressing.
2. Missing intended DHCP address → [TP-02](#tp-02). Valid address, failed traffic → [TP-01](#tp-01)/[TP-05](#tp-05). Track AP management and client data as separate flows.
3. Map SSID/role to effective VLAN, AP uplink tagging, switch membership and server/relay using [PT-D](#pt-d). Sharing a management/client VLAN is not inherently invalid; AP expectations and switch behavior determine compatibility.
4. Compare clients, SSIDs and APs independently; check isolation/ACLs. Wired forwarding/addressing works but wireless remains unreliable → [TP-08](#tp-08) and platform-specific RF investigation. An uplink capture does not establish over-the-air delivery.

<a id="tp-08"></a>
## TP-08: Intermittent or degraded service

**PROBE:** use a bounded timestamped baseline alongside an application check; ICMP alone does not measure the application.

```bash
(
  TARGET_IP='REPLACE_TARGET_IPV4'
  case "$TARGET_IP" in *REPLACE_*) echo 'Substitute all placeholders first.' >&2; exit 2;; esac
  ping -4 -n -D -i 1 -c 60 -W 2 "$TARGET_IP"
)
```

Correlate with before/after counters, link events, load, power/PoE, scheduled jobs, lease events and wireless channel/DFS events. Counter increments support a link hypothesis but do not uniquely prove duplex mismatch. Timer alignment → [TP-10](#tp-10) with measured [timers](#timers). Sustained failure → matching symptom plan. No recurrence → preserve evidence and define a bounded monitoring window/escalation threshold; do not declare a fix.

<a id="tp-09"></a>
## TP-09: VLAN isolation or reachability fault

1. Define expected allowed/denied client/VLAN/service pairs. VLANs separate Layer 2 domains; routing can intentionally connect them.
2. Build [PT-D](#pt-d)'s matrix for ports, APs and virtual bridges. Ingress classification/admission, membership, egress action and endpoint expectations must agree.
3. Supported mismatch → smallest coherent [CHANGE](#change-procedure) with recovery access. Settings consistent → [PT-F](#pt-f)/[PT-E](#pt-e) and effective role assignment.
4. Verify allowed service operations and expected denials from representative clients. Failed ping alone does not establish segmentation. Unresolved unexpected access → preserve evidence and escalate to the policy owner.

<a id="tp-10"></a>
## TP-10: Failure following a change

1. Search backward using relevant measured timers and last known good evidence. Include automation/adjacent systems. Unknown timers justify widening the window, not choosing a convenient default.
2. Rank changes by mechanism and testable prediction. Correlate time and [PT-H](#pt-h) evidence; sequence alone is not causality.
3. Appropriate targeted revert → [CHANGE](#change-procedure). Recovery supports causality but can coincide with independent recovery or cached-state reset. Failed revert does not eliminate the change: persistent state, incomplete rollback and multiple faults remain possible.
4. Update hypotheses, then return to the symptom plan or [verification](#verification).

## Pinpoint tests

<a id="pt-a"></a>
### PT-A: Link and physical evidence

**READ**, on the affected Linux host:

```bash
(
  IFACE='REPLACE_IFACE'
  case "$IFACE" in *REPLACE_*) echo 'Substitute all placeholders first.' >&2; exit 2;; esac
  ip -br link show dev "$IFACE"
  ip -s link show dev "$IFACE"
  ethtool "$IFACE"
)
```

Compare administrative state, carrier, speed/duplex, counter deltas, switch events and power at both ends. Virtual/wireless interfaces may not expose Ethernet negotiation. Read kernel journal events in UTC; humanized kernel uptime timestamps can mislead after clock changes. Bad evidence → inspect power/cabling/port state, with disruptive work logged as CHANGE. Normal evidence → caller; intermittency remains possible.

<a id="pt-b"></a>
### PT-B: IPv4 next-hop adjacency

Use the **on-link next hop from PT-F**, not a remote host reached through a router. **READ**, then **PROBE**, then READ:

```bash
(
  IFACE='REPLACE_IFACE'
  NEXT_HOP='REPLACE_NEXT_HOP_IPV4'
  case "$IFACE $NEXT_HOP" in *REPLACE_*) echo 'Substitute all placeholders first.' >&2; exit 2;; esac
  ip -4 neigh show to "$NEXT_HOP" dev "$IFACE"
  ping -4 -n -I "$IFACE" -c 3 -W 2 "$NEXT_HOP"
  ip -4 neigh show to "$NEXT_HOP" dev "$IFACE"
)
```

INCOMPLETE/FAILED indicates unresolved neighbour discovery, not its cause. Check requests/replies, VLAN/prefix, availability and filtering. STALE is normal cache state; a stored MAC does not prove current delivery. “No route to host” can arise from neighbour failure, routing or administrative rejection. Ping timeout may reflect ICMP policy. See [iproute2 neighbour documentation](https://raw.githubusercontent.com/iproute2/iproute2/main/man/man8/ip-neighbour.8).

<a id="pt-c"></a>
### PT-C: DHCP capture and correlation

**Choose placement before filters.** Capture on client/server/relay interfaces or use approved SPAN/mirroring or a TAP. A third host on an ordinary switched port cannot see all other hosts' unicast; promiscuous mode does not change switch forwarding. Mirror the relevant port/VLAN and directions; oversubscription can lose packets. A breakout TAP may need both outputs. Configuring a mirror or inserting a TAP is a [CHANGE](#change-procedure). See [Wireshark Ethernet capture guidance](https://wiki.wireshark.org/CaptureSetup/Ethernet).

**CAPTURE**, where DHCP is presented without an 802.1Q header:

```bash
(
  IFACE='REPLACE_IFACE'
  case "$IFACE" in *REPLACE_*) echo 'Substitute all placeholders first.' >&2; exit 2;; esac
  sudo tcpdump -ni "$IFACE" -e -vvv -s 0 -c 100 'udp and (port 67 or port 68)'
)
```

Stop with Ctrl-C at the approved time; `-c` limits packets, not duration. Retain drop statistics. For tagged links, inspect [PT-D](#pt-d)'s short unfiltered Ethernet view first and construct a filter for observed encapsulation. Port-only filters may miss tagged traffic. Do not filter solely on the client Ethernet MAC: broadcast replies and relay-side packets may lack it in their Ethernet headers.

Correlate transaction ID, client identifier/chaddr, message type, server identifier, requested/offered address, relay information, flags and time across points/logs. Source address alone does not identify DHCP state. Return to [TP-02](#tp-02) with visibility limits. Command/filter basis: [tcpdump](https://raw.githubusercontent.com/the-tcpdump-group/tcpdump/master/tcpdump.1.in), [libpcap](https://raw.githubusercontent.com/the-tcpdump-group/libpcap/master/pcap-filter.manmisc.in).

<a id="pt-d"></a>
### PT-D: VLAN ingress, egress and endpoint expectations

| Link/port role | Endpoint sends/expects | Admitted frame types | Untagged ingress PVID | Membership | Egress tagged/untagged | Evidence/unknowns |
|---|---|---|---|---|---|---|
| Fill privately | | | | | | |

PVID commonly classifies admitted untagged ingress. Admission/ingress-filtering rules decide acceptance; membership and egress policy govern forwarding/tag treatment. **PVID equal to a tagged member alone is not proof of a fault.** Establish endpoint expectations and effective policy. Accepted untagged ingress plus tagged return egress can fail for an endpoint expecting untagged replies; verify both directions before concluding that occurred.

Native/hybrid port, priority-tagged frame and CPU/bridge behavior are platform-specific. Do not impose a universal “same VLAN tagged and untagged is impossible” rule. Check effective state on the actual version. [MikroTik VLAN documentation](https://help.mikrotik.com/docs/spaces/ROS/pages/28606465/Bridge%2BVLAN%2BTable) illustrates separate controls; it does not identify incident hardware.

**CAPTURE:** brief approved inspection on physical Ethernet or validated mirror/TAP. Unfiltered traffic can contain unrelated sensitive data.

```bash
(
  IFACE='REPLACE_IFACE'
  case "$IFACE" in *REPLACE_*) echo 'Substitute all placeholders first.' >&2; exit 2;; esac
  sudo tcpdump -ni "$IFACE" -e -nn -s 0 -c 30
)
```

Stop at the approved time even if fewer packets arrive. VLAN subinterfaces, NIC offload, virtual switches and mirrors can remove/alter visible tags. `-e` shows available headers, not missing ones. A `vlan`-only filter excludes untagged frames and changes subsequent filter offsets. No visible tag is not proof of untagged wire traffic. Compare endpoint captures, validated wire observations and configuration. Offload changes are CHANGE actions. See [Wireshark VLAN capture](https://wiki.wireshark.org/CaptureSetup/VLAN) and [offloading](https://wiki.wireshark.org/CaptureSetup/Offloading).

<a id="pt-e"></a>
### PT-E: Effective firewall path

**READ:** identify backend, namespace, address family and traffic path. For nftables inspect `sudo nft list ruleset`; for iptables inspect `sudo iptables-save -c` and, when relevant, `sudo ip6tables-save -c`. These display rules/counters. Do not restore, flush or reset counters to diagnose a path.

Map input/output versus forwarding, NAT, bridge/guest policy, hooks/priorities, default policies and connection tracking; include switch ACLs and DHCP snooping. `iptables -L` of only the default table and a failed search for a guessed guest chain are incomplete evidence. `REPLACE_GUEST_ID` is an inventory placeholder, not a portable interface-name formula. See [Netfilter architecture](https://www.netfilter.org/documentation/HOWTO/netfilter-hacking-HOWTO-3.html) and [iptables documentation](https://www.netfilter.org/projects/iptables/index.html).

Compare counters around one scoped probe. Movement shows matching, not necessarily the final verdict for that exact flow. Some nftables rules lack counters; offload may bypass observation. Correlate tuple, order/verdict, route, NAT and captures. No match → validate backend/hooks/coverage before clearing the firewall. Adding logging/tracing is a CHANGE: narrow match, rate limit, private destination, stop time and exact removal plan. See the current [nft manual](https://netfilter.org/projects/nftables/manpage.html) for families, hooks, priorities and verdicts.

<a id="pt-f"></a>
### PT-F: Addressing and route selection

**READ:**

```bash
(
  IFACE='REPLACE_IFACE'
  TARGET_IP='REPLACE_TARGET_IPV4'
  case "$IFACE $TARGET_IP" in *REPLACE_*) echo 'Substitute all placeholders first.' >&2; exit 2;; esac
  ip -br addr show dev "$IFACE"
  ip -4 route show table all
  ip -4 rule show
  ip -4 route get "$TARGET_IP"
)
```

Compare source, interface, gateway and policy table with the application's actual namespace/source/marks. Basic route lookup may not reproduce policy-bound traffic. Check the other endpoint's return route. Asymmetry is not inherently broken; inspect stateful policy/reverse-path filtering. Traceroute/mtr are optional active probes: protocol, ECMP and ICMP rate limiting affect results; intermediate nonresponses do not prove transit loss. Wrong route → scoped CHANGE; consistent route → [PT-B](#pt-b)/[PT-H](#pt-h). See the [iproute2 route manual](https://raw.githubusercontent.com/iproute2/iproute2/main/man/man8/ip-route.8.in).

<a id="pt-g"></a>
### PT-G: Listener and TCP reachability

**READ**, in the service's namespace: `sudo ss -lntup`. Check process, port, family and binding. Wildcard listeners may expose multiple interfaces; IPv6 wildcard behavior depends on socket/platform settings. See [ss documentation](https://raw.githubusercontent.com/iproute2/iproute2/main/man/man8/ss.8).

**PROBE**, from failing client or matched control, using OpenBSD-compatible netcat:

```bash
(
  TARGET_IP='REPLACE_TARGET_IPV4'
  PORT='REPLACE_TCP_PORT'
  case "$TARGET_IP $PORT" in *REPLACE_*) echo 'Substitute all placeholders first.' >&2; exit 2;; esac
  nc -4 -n -v -z -w 3 "$TARGET_IP" "$PORT"
)
```

Success → actual application test. Refusal → inspect responder/reset and listener; intermediaries can reject. Timeout → [PT-H](#pt-h)/[PT-E](#pt-e), not an automatic firewall verdict. Other netcat implementations differ. UDP needs protocol-aware validation; [OpenBSD nc](https://man.openbsd.org/nc.1) documents UDP scanning limitations.

<a id="pt-h"></a>
### PT-H: Directional isolation

Arrange simultaneous observations near both endpoints; synchronize clocks and account for NAT, relays and encapsulation. **CAPTURE:** run separately at each untagged IPv4 endpoint, substituting the peer as visible there. DHCP pre-address broadcasts need [PT-C](#pt-c); tagged links need [PT-D](#pt-d).

```bash
(
  IFACE='REPLACE_IFACE'
  PEER_IP='REPLACE_PEER_IPV4'
  case "$IFACE $PEER_IP" in *REPLACE_*) echo 'Substitute all placeholders first.' >&2; exit 2;; esac
  sudo tcpdump -ni "$IFACE" -nn -tttt -s 0 -c 100 "host $PEER_IP"
)
```

Generate one approved **PROBE**, record its tuple/transaction ID and stop time, then stop both captures. Use this table only after validating coverage/drop statistics:

| Correlated observations | Supported next step |
|---|---|
| A outgoing request, no B incoming request | Inspect forward interval, not yet a specific device. |
| B incoming request/outgoing reply, no A incoming reply | Inspect return interval, translations and tagging. |
| B incoming request, no reply observed | Inspect B service, policy, reply route and alternate interfaces. |
| No A outgoing request observed | Check application action, local stack, route/namespace and capture point. |
| Both directions observed, operation fails | Inspect protocol contents, client acceptance and application logs. |

Host-side outgoing capture can precede wire transmission. Packet appearance is not application acceptance; timestamps alone do not match transactions.

<a id="pt-j"></a>
### PT-J: Matched control

From a working client, repeat the same operation against the **same server, port, protocol and DNS view** during the failure window. Record differences in subnet, VLAN, identity, route, family, pool, policy and time. For TCP use [PT-G](#pt-g); for DHCP compare transactions, not an existing lease.

Peer works → investigate differences while retaining server-side per-client/intermittent hypotheses. Peer fails → investigate shared dependencies and possible independent failures. No comparable control → record the limitation and improve [PT-H](#pt-h) coverage. Testing a different service on the peer does not control the original server.

## Timers

Use effective settings and observed state transitions, not defaults as an incident clock. A delay between a change and a symptom does not itself identify a timer.

| Mechanism | Interpretation |
|---|---|
| DHCP lease | T1/T2 may be supplied; otherwise RFC 2131 uses 0.5/0.875 of lease duration. Renewal failure need not immediately remove a valid address; expiry matters. No universal lease duration. |
| DHCP retries | Initial retransmission uses randomized exponential backoff. Renewal/rebinding retries can use half the remaining time to T2/expiry, with a 60-second minimum in RFC 2131. Shrinking intervals alone prove neither reply loss nor cause time. |
| Positive DNS cache | Remaining TTL, resolver and application caches matter. TTL is not a global propagation deadline; serve-stale can extend use. |
| Negative DNS cache | RFC 2308 uses the smaller of SOA TTL and SOA MINIMUM; inspect response and policy. |
| Neighbour/MAC aging | Read actual platform settings/transitions. Neighbour reachability and switch MAC aging are distinct; neither has a universal fault-delay value. |
| TCP keepalive | Must be enabled per socket. Linux documents 7200 seconds idle by default, followed by probes; idle interval alone is not detection time. Applications can override it. |
| Conntrack/firewall | Linux documents 432000 seconds for established TCP conntrack by default. Refresh, rule order, offload and policy govern whether old connections survive; this is not a grace-period guarantee. |
| Certificates, heartbeats, routing adjacencies | Inspect actual expiry, reconnect behavior and protocol configuration. OSPF dead intervals and BGP hold timers differ; no universal “three times hello” rule. |

Sources: [DHCP](https://www.rfc-editor.org/rfc/rfc2131.html), [negative DNS caching](https://www.rfc-editor.org/rfc/rfc2308.html), [serve-stale](https://www.rfc-editor.org/rfc/rfc8767.html), [Linux TCP settings](https://www.kernel.org/doc/html/latest/networking/ip-sysctl.html), [conntrack](https://www.kernel.org/doc/html/latest/networking/nf_conntrack-sysctl.html).

## Change procedure

Before a **CHANGE**, record owner/authorization, evidence/prediction, exact old/new values, dependencies/users affected, window, saved private configuration, recovery access, success criteria and rollback trigger/sequence in the [template](evidence-change-log.md). Confirm recovery access before risking management connectivity. Apply one isolated change or explicitly coupled set, verify, then retain or roll back. Do not improvise vendor commands from this guide.

| Action | Prerequisites and impact | Rollback/recovery |
|---|---|---|
| VLAN/PVID/AP role change | Map endpoints/management path; console/out-of-band access or supported timed revert. May disconnect clients/admins. | Restore coherent port/endpoint settings through recovery access; verify management and data. |
| DHCP service/scope/relay change | Confirm authorization, redundancy, scope interactions and lease state. May stop new leases/renewals. | Restore service/configuration; verify new leases and renewals. Configuration restore may not recover lost lease state. |
| Release/renew, static address, cache flush | Preserve state; use a recoverable test client. May terminate sessions/erase evidence. Static tests need allocated conflict-free address, correct prefix/gateway/DNS and pool exclusion. | Restore addressing method/settings; reacquire and validate service. Flushed evidence cannot be restored. |
| Bind/route/firewall/logging change | Limit exposure/match scope; preserve effective state and access. Logs can expose data/fill storage. | Restore exact settings, remove temporary logging, verify allowed/denied flows. Do not broadly flush firewall/conntrack. |
| SPAN/TAP/offload/cabling change | Validate capacity, directions, tag preservation and link impact; plan hardware insertion interruption. | Restore settings/cabling and verify link; remove collection access. |
| Restart/reboot/reset | Preserve evidence; establish restore, trust/credentials and re-adoption procedure. Interrupts service and may erase configuration. | Follow recovery plan; volatile evidence cannot be recreated. |

Static addressing is a design choice, not a universal repair. Reservations retain DHCP dependence; static addresses need coordinated IPAM, pool exclusion and maintenance. Neither fixes broken VLAN forwarding. Select resilience and recovery for the actual dependency chain.

## Verification

1. Repeat the original operation from an affected client with intended family, DNS path and service. Record result/evidence, not just “ping works.”
2. Test adjacent clients/SSIDs/VLANs and expected isolation. For DHCP, validate address/options and return traffic; cover new allocation and renewal as applicable on an approved test client. Avoid mass lease release.
3. Observe across the suspected trigger/timer when feasible. A shorter window requires an explicit limitation and follow-up, not a durable-recovery claim.
4. Record repair/rollback status, coupled settings, remaining risks and supported conclusions. Remove temporary collection/logging changes.
5. Choose bounded monitoring for the failed function, such as lease acquisition or an application transaction, with an owner. Reachability alone cannot detect every DHCP/tagging failure.
6. Preserve private originals; publish reviewed summaries only. Use the [template](evidence-change-log.md) for closure and [references](references.md) for platform checks.
