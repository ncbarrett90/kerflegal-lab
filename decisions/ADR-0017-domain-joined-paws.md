# ADR-0017: Domain-Joined PAWs with Tier-Separated GPO Hardening

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-05-03 |
| Supersedes | - |
| Superseded By | - |

## Context

The lab plans three PAWs under the per-tier model in [ADR-0002](ADR-0002-per-tier-paw-model.md), placed per [ADR-0010](ADR-0010-dedicated-paw-hypervisor.md). The choice of how those PAWs are joined to AD shapes the buildout order and how admin credentials are handled day to day.

## Decision

Domain-join all three PAWs to the production AD forest. Place them in a dedicated PAW OU with tier-separated GPOs covering Credential Guard, Protected Users membership, AppLocker execution control, and logon restrictions that limit each tier's admin accounts to its own PAW. Outbound network policy stays as set in [ADR-0006](ADR-0006-network-segmentation-and-ip-vlan-scheme.md).

## Alternatives Considered

- **Workgroup PAWs with `runas`** - Rejected. No central management, manual hardening per box, and the trust-isolation argument is largely theoretical since admin credentials get typed into the PAW anyway.
- **Separate admin forest (ESAE)** - Rejected. Microsoft deprecated the model around 2020-2021 and a second AD just for admin identity is too much overhead at this scale.
- **Cloud-managed via Entra and Intune** - Rejected for an on-prem lab without M365 E5 licensing. Worth revisiting if hybrid identity is added later.

## Consequences

- DC01 with the PAW OU and core GPOs must come up before the PAW VMs.
- Central GPO is the management plane for hardening across paw01-03.
- Credential hygiene comes from Protected Users plus Credential Guard, not `runas` ergonomics.
- DC tier compromise reaches the PAWs by trust inheritance. Mitigation is hardening the DC tier rather than removing the trust.
- **Deviation:** Enterprise environments layer a JIT elevation product (CyberArk, BeyondTrust, Microsoft PIM). The lab uses static admin accounts with logon restrictions, no JIT.

## References

- [ADR-0001](ADR-0001-three-tier-administrative-model.md)
- [ADR-0002](ADR-0002-per-tier-paw-model.md)
- [ADR-0006](ADR-0006-network-segmentation-and-ip-vlan-scheme.md)
- [ADR-0010](ADR-0010-dedicated-paw-hypervisor.md)
