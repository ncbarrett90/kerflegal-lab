# ADR-0015: Hypervisor Trunk Port Tagging Policy

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-05-01 |
| Supersedes | - |
| Superseded By | - |

## Context

Hypervisor and firewall uplink trunks carry management plus VM zone VLANs. The native VLAN style of trunking (management on PVID untagged, other VLANs tagged) is the Proxmox installer default and a documented anti-pattern in production. CIS Cisco and Juniper hardening guides recommend tagging the native VLAN or pinning PVID to an unused VLAN to mitigate 802.1Q double-tagging.

## Decision

Tagged-only trunks. Every production VLAN is tagged. PVID is VLAN 1, intentionally unused. Acceptable Frame Types is Tagged Only so untagged ingress is dropped. Hypervisor hosts use explicit tagged sub-interfaces (e.g., `vmbr0.10`) for management.

Applies to sw01 port 1 (fw01 uplink) and ports 4-7 (pve01-04). Single-VLAN endpoints (pbs01, wazuh01) remain access ports.

## Alternatives Considered

- **Hybrid trunk, native VLAN for management** - PVID untagged on management, other VLANs tagged. Rejected. Enables 802.1Q double-tagging and lands misconfigured devices on management.
- **Tagged-only on every port** - Apply to single-VLAN endpoints too. Rejected. No security gain, adds host config burden.

## Consequences

- VLAN hopping via double-tagging is structurally blocked.
- Stray untagged frames on a tagged-only trunk land on unused VLAN 1, not management.
- Hypervisor hosts must configure a tagged management sub-interface during baseline. Procedure 06 step 2 codifies this.
- **Deviation:** Single-NIC lab hardware cannot mirror enterprise out-of-band management on a separate physical NIC. Tagged-on-trunk is the closest defensible approximation.
- **Deviation:** Port 2 (ap01) keeps management untagged on VLAN 10 because the UAP-AC-Pro binds management to its native VLAN. Re-adoption with tagged management is deferred to the hardening pass.
