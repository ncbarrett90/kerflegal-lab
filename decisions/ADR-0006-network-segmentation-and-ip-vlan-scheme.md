# ADR-0006: Network Segmentation and IP/VLAN Scheme

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-04-17 |
| Supersedes | - |
| Superseded By | - |

## Context

The three tier model requires that the network enforce tier boundaries rather than leaving them to host configuration alone. Flat networks collapse the model; segmentation that leaves PAWs on the same subnet as the systems they administer does as well. The SIEM does not belong on the management plane, since it needs to ingest logs and agent traffic from every zone. Identity systems (DCs and CA) are the highest-value targets in the environment and warrant isolation from general member servers for tighter firewall scoping.

## Decision

Six lab zones plus a declared HOME zone, each on its own VLAN and subnet. VLAN IDs start at 10 and increment by 10 through the lab zones. HOME uses VLAN 100 to leave room for future lab expansion. Subnets follow the pattern `10.0.<VLAN>.0/24` so the VLAN ID is readable from the IP.

| Zone | VLAN | Subnet | Gateway | Tier | Purpose |
|---|---|---|---|---|---|
| INFRA | 10 | 10.0.10.0/24 | 10.0.10.1 | 0 | Hypervisor admin, PBS, network gear management |
| SECURITY | 20 | 10.0.20.0/24 | 10.0.20.1 | 0 | Wazuh SIEM, future security tooling |
| PAW | 30 | 10.0.30.0/24 | 10.0.30.1 | 0 | Privileged Access Workstations |
| IDENTITY | 40 | 10.0.40.0/24 | 10.0.40.1 | 0 | DCs, CA |
| SERVERS | 50 | 10.0.50.0/24 | 10.0.50.1 | 1 | File servers, member servers |
| CLIENTS | 60 | 10.0.60.0/24 | 10.0.60.1 | 2 | Domain-joined Windows workstations |
| HOME (out of scope) | 100 | 10.0.100.0/24 | 10.0.100.1 | 0 | Personal workstation, access point |

INFRA, SECURITY, PAW, and IDENTITY are all Tier 0 but separated by traffic profile and trust boundary: INFRA carries the hypervisor and infrastructure management plane, SECURITY accepts inbound log/agent traffic from every zone, PAW holds the Privileged Access Workstations, and IDENTITY holds the authentication and certificate authorities. Keeping these separate prevents a single compromise from giving broad Tier 0 access.

Default deny between all zones at the firewall. Specific allow rules (domain service dependencies, log ingestion into SECURITY, administrative sessions from PAW into the correct target tier, DC authentication from SERVERS and CLIENTS into IDENTITY) are maintained in [network-design.md](../design/network-design.md).

**VLAN constraints:**
- VLAN 1 is the switch default and cannot be reassigned.
- VLAN 2 is hardcoded on the Netgear distribution switch and cannot be removed.
- Trunk ports allow VLANs 10/20/30/40/50/60/100 and explicitly exclude VLANs 1 and 2. Per-trunk VLAN membership is defined in [network-design.md](../design/network-design.md); not every trunk carries every VLAN.

## Alternatives Considered

- **Flat network** - All devices on one subnet. Cannot enforce the tier model. Rejected.
- **Collapsed Tier 0 (single management zone)** - SIEM, PAWs, and identity systems share one zone. Opens ingest ports on the management plane and puts admin workstations on the same subnet as the systems they administer. Rejected.
- **IDENTITY collapsed into SERVERS** - DCs, CA, file servers, and member servers on one Tier 1 subnet. Works, but identity systems are the highest-value targets and warrant their own firewall scoping and Tier 0 classification. Rejected.
- **VLAN ID in second octet (10.\<VLAN\>.0.0/24)** - Second octet is conventionally reserved for distinguishing sites in multi-site deployments. Rejected.
- **Default VLAN for lab infrastructure** - The default VLAN is where misconfigured or forgotten devices land by accident. All lab devices should sit on purpose-built VLANs. Rejected.

## Consequences

- Tier separation is enforced at layer 2/3, not just at the host.
- Four Tier 0 zones (INFRA, SECURITY, PAW, IDENTITY) reduce blast radius within the tier: compromise of one Tier 0 zone does not grant broad Tier 0 access.
- IDENTITY on its own zone provides tight firewall scoping for authentication and PKI traffic and isolates the highest-value systems.
- Consistent IP layout (`10.0.<VLAN>.0/24`) makes firewall rules, DHCP scopes, and troubleshooting predictable.
- Additional switch and firewall configuration is required to support the trunking and inter-VLAN policy.
- **Deviation:** The HOME VLAN is declared in the table for completeness but is outside the lab's zone model. Personal devices on HOME are not tracked in lab artifacts. HOME is labeled Tier 0 because the personal workstation is the launch point for PAW sessions; in a production environment the admin's entry point would itself be a Tier 0 PAW.

## References

- [NIST SP 800-207 - Zero Trust Architecture](https://csrc.nist.gov/pubs/sp/800/207/final)
