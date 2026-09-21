# Primary references

Reviewed for this documentation revision on 2026-09-21. Consult the manual for the installed version before operational use; upstream documentation can change. Vendor documentation illustrates behavior, not the unidentified incident platform.

| Topic | Primary source and relevant material |
|---|---|
| DHCPv4 | [RFC 2131](https://www.rfc-editor.org/rfc/rfc2131.html), sections 3.1, 4.1 and 4.4: allocation, client state, retransmission, renewal/rebinding and expiry |
| Negative DNS caching | [RFC 2308](https://www.rfc-editor.org/rfc/rfc2308.html), sections 3 and 5: SOA-derived negative cache lifetime |
| DNS beyond TTL | [RFC 8767](https://www.rfc-editor.org/rfc/rfc8767.html): serving stale data under defined conditions |
| DNS commands | [ISC BIND manual pages](https://bind9.readthedocs.io/en/latest/manpages.html), dig: explicit server, response output, TCP and iterative trace |
| Switched capture placement | [Wireshark Ethernet capture setup](https://wiki.wireshark.org/CaptureSetup/Ethernet): endpoint, mirror and TAP placement and limitations |
| VLAN capture visibility | [Wireshark VLAN capture setup](https://wiki.wireshark.org/CaptureSetup/VLAN): physical versus VLAN interfaces and tag stripping |
| Offload artifacts | [Wireshark offloading guidance](https://wiki.wireshark.org/CaptureSetup/Offloading): checksum/segmentation interpretation |
| Capture commands | [tcpdump upstream manual source](https://raw.githubusercontent.com/the-tcpdump-group/tcpdump/master/tcpdump.1.in): interface, link headers, snapshot length, count and drop reporting |
| Capture filter semantics | [libpcap upstream filter manual](https://raw.githubusercontent.com/the-tcpdump-group/libpcap/master/pcap-filter.manmisc.in): host/port filters and VLAN offset changes |
| VLAN controls | [MikroTik bridge VLAN documentation](https://help.mikrotik.com/docs/spaces/ROS/pages/28606465/Bridge%2BVLAN%2BTable): PVID, frame admission, ingress filtering and egress membership; platform example only |
| Linux neighbour state | [iproute2 neighbour manual](https://raw.githubusercontent.com/iproute2/iproute2/main/man/man8/ip-neighbour.8): state definitions and selectors |
| Route lookup | [iproute2 route manual](https://raw.githubusercontent.com/iproute2/iproute2/main/man/man8/ip-route.8.in): tables, get, source/mark selectors and route errors |
| Listener inspection | [iproute2 ss manual](https://raw.githubusercontent.com/iproute2/iproute2/main/man/man8/ss.8): listening sockets, protocols and process display |
| IPv4 probes | [iputils ping manual source](https://raw.githubusercontent.com/iputils/iputils/master/doc/ping.xml): packet size, PMTU, count, wait and timestamp flags |
| TCP/UDP probe limits | [OpenBSD nc manual](https://man.openbsd.org/nc.1): zero-I/O TCP checks, timeouts and UDP scanning caveats |
| Service logs | [systemd journalctl manual source](https://raw.githubusercontent.com/systemd/systemd/main/man/journalctl.xml): unit filters, UTC and time windows |
| Firewall path | [Netfilter architecture](https://www.netfilter.org/documentation/HOWTO/netfilter-hacking-HOWTO-3.html): hooks and packet traversal; foundational architecture rather than a current platform configuration guide |
| Firewall tools | [Netfilter iptables documentation](https://www.netfilter.org/projects/iptables/index.html): official tool/manual entry points |
| Counter-preserving inspection | [Debian-packaged iptables-save manual](https://manpages.debian.org/trixie/iptables/iptables-save.8.en.html): stdout dump and counters option |
| Current nftables behavior | [Netfilter nft manual](https://netfilter.org/projects/nftables/manpage.html): list ruleset, families, hooks, priorities, counters and verdicts |
| TCP timing and reverse-path filtering | [Linux IP sysctl documentation](https://www.kernel.org/doc/html/latest/networking/ip-sysctl.html): keepalive settings and rp_filter |
| Connection tracking | [Linux conntrack sysctl documentation](https://www.kernel.org/doc/html/latest/networking/nf_conntrack-sysctl.html): protocol-specific state timeouts |

These references support protocol/command review. They do not establish which commands were executed or which packets were observed in the motivating incident.
