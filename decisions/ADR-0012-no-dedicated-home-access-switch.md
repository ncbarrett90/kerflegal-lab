# ADR-0012: No Dedicated HOME Access Switch

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-04-23 |
| Supersedes | - |
| Superseded By | - |

## Context

The original topology placed sw02 (TP-Link TL-SG108E) as an HTTP-only managed switch on the VLAN 10/100 trunk from sw01, serving the personal workstation on VLAN 100. HTTP management adjacent to a trunk carrying INFRA traffic is a credential-capture risk. The available mitigations (source-scoped fw01 rule, strong admin password) narrow the source scope but do not remove the risk class.

## Decision

Remove sw02. The personal workstation connects directly to sw01, which configures the port as VLAN 100 access.

## Alternatives Considered

- **Keep TL-SG108E with mitigations** - Source-scoped fw01 rule plus a strong admin password; document HTTP credential exposure on the risk register. Rejected: the risk class persists for the life of the device, and a single wired HOME client is not reason enough to carry a permanent Tier 0 exception.
- **Replace with a switch that supports HTTPS/SSH** - A Netgear GS-series or TP-Link JetStream unit. Rejected: a dedicated device for a single wired HOME client is not justified when sw01 already reaches the workstation.

## Consequences

- HTTP-management credential-capture risk on HOME-zone hardware is eliminated by design rather than mitigated.
- Future wired HOME capacity beyond the workstation becomes a separate decision if the need arises.

## References

- [ADR-0006](ADR-0006-network-segmentation-and-ip-vlan-scheme.md) - Network Segmentation and IP/VLAN Scheme
