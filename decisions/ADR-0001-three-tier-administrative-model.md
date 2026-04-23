# ADR-0001: Adopt Microsoft Three Tier Administrative Model

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-04-14 |
| Supersedes | - |
| Superseded By | - |

## Context

The most common authentication pattern I see in small business Active Directory environments is a single admin account that logs into domain controllers, member servers, and workstations all with the same credentials. Nothing stops a Tier 0 credential from being typed into a compromised endpoint. The path from one infected workstation to the DC is usually a short one. The lab is the place to close that gap with a model that is enforceable.

## Decision

Adopt Microsoft's three-tier model:

- **Tier 0** - Identity and control plane (DCs, CA, hypervisors, backup, PAW)
- **Tier 1** - Servers (member servers, file servers, app servers)
- **Tier 2** - Workstations and endpoints

**Core rule:** credentials never authenticate to a system at a lower tier than themselves. Each tier has its own admin accounts.

## Alternatives Considered

- **No tier model** - The exact MSP pattern I am trying to avoid. Rejected.
- **Enterprise Access Model** - Microsoft's successor framework, built around cloud identity and hybrid environments. Too far ahead of where this lab sits. Revisit once Entra ID is integrated.

## Consequences

- Forces explicit credential separation across tiers.
- Multiple admin accounts per administrator. Adds overhead and that is the point.
- Requires dedicated PAW devices for Tier 0 administration.

## References

- [Microsoft - PAM environment tier model (legacy)](https://learn.microsoft.com/en-us/microsoft-identity-manager/pam/tier-model-for-partitioning-administrative-privileges)
- [Microsoft - Planning a bastion environment](https://learn.microsoft.com/en-us/microsoft-identity-manager/pam/planning-bastion-environment)
- [Microsoft - Enterprise access model (successor)](https://learn.microsoft.com/en-us/security/privileged-access-workstations/privileged-access-access-model)
