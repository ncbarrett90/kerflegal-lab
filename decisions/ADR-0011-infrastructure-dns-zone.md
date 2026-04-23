# ADR-0011: Infrastructure DNS Zone

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-04-22 |
| Supersedes | [ADR-0007](ADR-0007-dns-architecture.md) |
| Superseded By | - |

## Context

ADR-0007 established `kerflegal.com` (public) and `corp.kerflegal.com` (internal AD) but did not name a zone for non domain joined infrastructure.

## Decision

Add `infra.kerflegal.com` as a third internal zone. Unbound on fw01 is authoritative, implemented as host overrides. Domain controllers conditionally forward queries for `infra.kerflegal.com` to fw01.

All other elements of ADR-0007 are retained.

| Zone | Authority | Purpose |
|---|---|---|
| kerflegal.com | GoDaddy | Public |
| corp.kerflegal.com | DCs (AD-integrated) | Domain-joined |
| infra.kerflegal.com | Unbound on fw01 | Non-domain-joined infra |

## Alternatives Considered

- **Infra devices under `corp.kerflegal.com`** - Mixes static and dynamic record management in one zone. Rejected.
- **No DNS for infra, IPs only** - Loses name resolution for admin work and log readability. Rejected.

## Consequences

- Manually-managed records isolated from AD-managed records.
- Drift between Unbound overrides and actual IPs is a manual discipline.
- Firewall policy, not DNS, controls what can actually reach infra.
- **Deviation:** Enterprise practice uses IPAM/DDI (Infoblox) to sync DNS and IP allocation. Lab uses manual overrides plus the device registry.

## References

